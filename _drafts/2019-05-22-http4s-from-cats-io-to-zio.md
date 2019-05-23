---
title: http4s, from Cats IO to ZIO
category: article
tags: "scala guardrail http4s cats zio tutorial"
toc: true
toc_label: "Table of Contents"
---


===
# TODO: spellcheck
# TODO: Cats IO / Cats Effect distinction
# TODO: Link to previous post
===

In this episode we'll continue on our [http4s](https://http4s.org/) REST API
server from the [previous article](about:blank), this time we replace the 
default [Cats](https://typelevel.org/cats/) runtime with 
[Scalaz ZIO](https://scalaz.github.io/scalaz-zio/).

## tl;dr - Show me the code!

Finished project here:
[guardrail-http4s-zio-example](https://github.com/kkreuning/guardrail-http4s-zio-example)

## Why?

I wanted to learn more about Scalaz ZIO after watching some talks from
[John A de Goes](http://degoes.net) on Scalaz ZIO, and what better way to learn
than actually build something with it?

## How?

We will continue with our
[guardrail-http4s-example](https://github.com/kkreuning/guardrail-http4s-example),
if you like to follow along you can clone the repo or download the code.

## Picking up where we left

First of all, we need to add the Scalaz ZIO dependencies to our `build.sbt` so
that it looks like this:

```scala
val Http4sVersion = "0.20.0"
val CirceVersion = "0.11.1"
val Specs2Version = "4.1.0"
val LogbackVersion = "1.2.3"
val ZioVersion = "1.0-RC4"

lazy val root = (project in file("."))
  .settings(
    organization := "com.example",
    name := "quickstart",
    version := "0.0.1-SNAPSHOT",
    scalaVersion := "2.12.8",
    scalacOptions ++= Seq("-Ypartial-unification"),
    libraryDependencies ++= Seq(
      "org.http4s"      %% "http4s-blaze-server"     % Http4sVersion,
      "org.http4s"      %% "http4s-blaze-client"     % Http4sVersion,
      "org.http4s"      %% "http4s-circe"            % Http4sVersion,
      "org.http4s"      %% "http4s-dsl"              % Http4sVersion,
      "org.scalaz"      %% "scalaz-zio"              % ZioVersion,
      "org.scalaz"      %% "scalaz-zio-interop-cats" % ZioVersion,
      "io.circe"        %% "circe-generic"           % CirceVersion,
      "io.circe"        %% "circe-java8"             % CirceVersion,
      "ch.qos.logback"  %  "logback-classic"         % LogbackVersion,
    ),
    addCompilerPlugin("org.spire-math" %% "kind-projector"     % "0.9.6"),
    addCompilerPlugin("com.olegpy"     %% "better-monadic-for" % "0.2.4")
  )

scalacOptions ++= Seq(
  "-deprecation",
  "-encoding", "UTF-8",
  "-language:higherKinds",
  "-language:postfixOps",
  "-feature",
  "-Ypartial-unification",
  "-Xfatal-warnings",
)

guardrailTasks in Compile := List(
  ScalaServer(
    specPath = (Compile / resourceDirectory).value / "api.yaml",
    pkg = "com.example.quickstart.endpoints",
    framework = "http4s",
    tracing = false
  )
)
```

We added

```scala
"org.scalaz" %% "scalaz-zio" % ZioVersion,
"org.scalaz" %% "scalaz-zio-interop-cats" % ZioVersion,
```

which includes the ZIO runtime and Cats interop which we need because http4s
(and Guardrail) are built on top of Cats
[type classes](https://typelevel.org/cats/typeclasses.html).

## Updating the endpoint implementation

Next, we are going to update the
`src/main/scala/com/example/quickstart/endpoints/HelloHandlerImpl.scala` to work
with `ZIO`. Unfortunately we can't use our handler implementation in this form.
Guardrail generated our `Routes.scala` with following structure:

```scala
trait HelloHandler[F[_]] { def getHello(respond: GetHelloResponse.type)(name: Option[String] = None): F[GetHelloResponse] }
class HelloResource[F[_]]()(implicit F: Async[F]) extends Http4sDsl[F] {
  ...
}
```

This will not work (directly) with ZIO because the type parameter `F[_]` has one
hole. This fits with the cats `IO[A]` monad but the `ZIO` primitive has three
parameters: `ZIO[R, E, A]`. The big difference is that ZIO is explicit about the
failures (the `E` type) while cats IO doesn't show this to the user. This means
that we can't define a program with Cats IO that never ever fails based on it's
type (remember, there is just `IO[A]`).

ZIO has some extra
[type aliases](https://scalaz.github.io/scalaz-zio/overview/index.html#type-aliases)
for `ZIO[R, E, A]` that allow you to describe programs more precisely, for
example:

- `Task[A]`, which mirrors Cats IO, is an alias for
`ZIO[Any, Throwable, A]`. This structure tells us that, yes, this program may
fail with a `Throwable`
- `UIO[A]` is an alias for `ZIO[Any, Nothing, A]`. Here the failure type is
`Nothing` so we can never fail (U is for uninterruptible)

IMO this is a tremendous step up from Cats IO since we can desribe truly pure
programs.

Back to the problem, we can leverage type aliases to fit a `ZIO[R, E, A]` into a
`F[_]` shaped slot. How? By introducing a level of indirection of course! Rename
`HelloHandlerImpl.scala` to `HelloEndpoints.scala` and lets wrap our
implementation:

```scala
package com.example.quickstart.endpoints

import com.example.quickstart.endpoints.definitions.HelloResponse
import com.example.quickstart.endpoints.hello._
import org.http4s.HttpRoutes
import scalaz.zio._
import scalaz.zio.interop.catz._

final class HelloEndpoint[R]() {

  type HelloEndpointsTask[A] = TaskR[R, A]

  def routes: HttpRoutes[HelloEndpointsTask] = new HelloResource[HelloEndpointsTask]().routes(new HelloHandlerImpl)
    
  private class HelloHandlerImpl extends HelloHandler[HelloEndpointsTask] {

    def getHello(respond: GetHelloResponse.type)(name: Option[String]): HelloEndpointsTask[GetHelloResponse] = {
      val message = s"Hello, ${name getOrElse "world"}!"
      UIO.succeed(respond.Ok(HelloResponse(message)))
    }
  }
}
```

We defined a new type `HelloEndpointTask[A]` which aliases ZIO's `TaskR[R, A]`
which itself is an alias for `ZIO[R, Throwable, A]`. We use this type in place
of the `F[A]`.

As seen above, the
`class HelloResource[F[_]]()(implicit F: Async[F]) extends Http4sDsl[F]` also
required an implicit `Async` type class. This is taken care by importing 
`scalaz.zio.interop.catz._`.

Inside the `getHello` method definition we see `UIO` in action, we now for sure
that `respond.Ok(HelloResponse(message))` can not fail. Now in this simple
example that should be obvious, but in case of more complex logic it is nice to
be able to tell real pure programs from the just the presence of the `UIO` type.

## ZIO Environment type

Up until now we have conveniently ignored the `R` type in `ZIO[R, E, A]`. `R`
stands for the environment type. We will see how to use that next.

Let's pretend that coming up with a greeting should actually be handled by
another component that can have multiple implementations, maybe we want to
consult the database first or read from the file system. This is where to
environment comes in. We can define that, for our `HelloEndpoint` to work we
need access to a greeter service. You can think for it as some form of
dependency injection.

First let's create a greeter service, under
`src/main/scala/com/example/quickstart/greeter/` create the following
files:

`Greeter.scala`,

```scala
package com.example.quickstart.greeter

import scalaz.zio.TaskR

trait Greeter {

  def greeterService: Greeter.Service[Any]
}

object Greeter {

  trait Service[R] {

    def createGreeting(name: Option[String]): TaskR[R, String]
  }
}

trait HelloGreeter extends Greeter {

  override def greeterService: Greeter.Service[Any] = new Greeter.Service[Any] {

    def createGreeting(name: Option[String]): TaskR[Any,String] =
      TaskR.succeed(s"Hello, ${name getOrElse "world"}!")
  } 
}
```

In which we define our service interfaces, and `package.scala`:

```scala
package com.example.quickstart.greeter

import scalaz.zio.{TaskR, ZIO}

package object greeter extends Greeter.Service[Greeter] {

  def createGreeting(name: Option[String]): TaskR[Greeter, String] =
    ZIO.accessM(_.greeterService.createGreeting(name))
}
```

Where we define a way for other components to interact with the Greeter. We can
just import this package object wherever we want to use the greeter logic. ZIO
accesses the environment through `ZIO.accessM`.

Now we can change `HelloEndpoint.scala` to the following:

```scala
package com.example.quickstart.endpoints

import com.example.quickstart.endpoints.definitions.HelloResponse
import com.example.quickstart.endpoints.hello._
import com.example.quickstart.greeter.Greeter
import com.example.quickstart.greeter._
import org.http4s.HttpRoutes
import scalaz.zio._
import scalaz.zio.interop.catz._

final class HelloEndpoint[R <: Greeter]() {

  type HelloEndpointsTask[A] = TaskR[R, A]

  def routes: HttpRoutes[HelloEndpointsTask] = new HelloResource[HelloEndpointsTask]().routes(new HelloHandlerImpl)
    
  private class HelloHandlerImpl extends HelloHandler[HelloEndpointsTask] {

    def getHello(respond: GetHelloResponse.type)(name: Option[String]): HelloEndpointsTask[GetHelloResponse] = {

      for {
        greeting <- createGreeting(name)
      } yield respond.Ok(HelloResponse(greeting))
    }
  }
}
```

We imported the greeter package object so we can use `createGreeting`, and in
the class definition we have `HelloEndpoint[R <: Greeter]` to tell the compiler
that yes, we want an environment `R` that extends (has) a `Greeter`. We will
see how to wire up a specific `Greeter` to the environment.

## ZIO Application

First, remove
`src/main/scala/com/example/quickstart/QuickstartServer.scala`. We are going to
put all the logic in the Main class next, all the code in here is tied to Cats
effects anyway so it's easier to just rewrite it.

Update `src/main/scala/com/example/quickstart/Main.scala` with:

```scala
package com.example.quickstart

import cats.effect.ExitCode
import cats.syntax.semigroupk._
import com.example.quickstart.endpoints.HelloEndpoint
import com.example.quickstart.greeter.{Greeter, HelloGreeter}
import org.http4s.implicits._
import org.http4s.server.blaze.BlazeServerBuilder
import scalaz.zio._
import scalaz.zio.blocking.Blocking
import scalaz.zio.clock.Clock
import scalaz.zio.console.{Console, putStrLn}
import scalaz.zio.interop.catz._
import scalaz.zio.scheduler.Scheduler

object Main extends App {

  type AppEnv = Clock with Console with Blocking with Greeter
  type AppTask[A] = TaskR[AppEnv, A]

  def run(args: List[String]): ZIO[Environment, Nothing, Int] = {

    val endpoints = Seq(
      new HelloEndpoint[AppEnv].routes
    ).reduce(_ <+> _)

    val program = for {
      httpApp <- UIO(endpoints.orNotFound)
      server   = ZIO.runtime[AppEnv].flatMap { implicit rts =>
                   BlazeServerBuilder[AppTask]
                     .bindHttp(8080, "0.0.0.0")
                     .withHttpApp(httpApp)
                     .serve
                     .compile[AppTask, AppTask, ExitCode]
                     .drain
                 }
      program <- server.provideSome[Environment] { base =>
                   new Clock with Console with Blocking with HelloGreeter {
                     override val clock: Clock.Service[Any] = base.clock
                     override val console: Console.Service[Any] = base.console
                     override val blocking: Blocking.Service[Any] = base.blocking
                     override val scheduler: Scheduler.Service[Any] = base.scheduler
                   }
                 }
    } yield program

    program.foldM(
      e => putStrLn(s"Execution failed with error $e") *> ZIO.succeed(1),
      _ => ZIO.succeed(0)
    )
  }
}
```

There is a lot to unpack here, let's go step by step:

```scala
type AppEnv = Clock with Console with Blocking with Greeter
```

defines our own environment with some standard stuff and our own `Greeter`. Nice
thing is that our code is actually type checked, if you were to remove the
`Greeter`, you'll get a nice compiler error, try it.

```scala
ZIO.runtime[AppEnv].flatMap { implicit rts =>
```

allows us to run some (possible non-functional) code inside the ZIO runtime with
our own environment and return that as a ZIO program.

```scala
program <- server.provideSome[Environment] { base =>
```

is used to construct our concrete environment, the `Environment` type provides
default 'building blocks' when defining a customized environment.

Finally

```scala
program.foldM(
  e => putStrLn(s"Execution failed with error $e") *> ZIO.succeed(1),
  _ => ZIO.succeed(0)
)
```

Actually runs our program and returns an exit code, as required by
`scala.zio.App`.

## Conclusion

That's it! We refactored our cats IO based http4s application to use ZIO and we
introduced a service to show off ZIO's environment mechanism.

Shout out to [Maxim Schuwalow](https://github.com/mschuwalow) for his
[zio-todo-backend](https://github.com/mschuwalow/zio-todo-backend), which was an
inspiration for this article.

You can find the finished project at
[guardrail-http4s-zio-example](https://github.com/kkreuning/guardrail-http4s-zio-example)

Thank you for reading.
