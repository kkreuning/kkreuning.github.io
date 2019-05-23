---
title: Guardrail with http4s tutorial
category: article
tags: "scala guardrail http4s tutorial"
toc: true
toc_label: "Table of Contents"
---

This article is an introduction on how to use Twilio's 
[Guardrail](https://guardrail.dev) to safely generate and maintain a 
[http4s](https://http4s.org) REST API server. I wanted to write this article as
a reference for my future self and others who are interested in this technology.

![real guardrail in its natural habitat](/assets/images/guardrail.jpg)

## tl;dr -  I just want the code!

Find the finished project at 
[guardrail-http4s-tutorial](https://github.com/kkreuning/guardrail-http4s-example)

## The case for Guardrail

To quote from Guardrail's website:

> Guardrail is a code generation tool, capable of reading from OpenAPI/Swagger
specification files and generating Scala source code, primarily targeting the
akka-http and http4s web frameworks, using circe for JSON encoding/decoding.

That's nice and all, but Swagger is already capable of generating Scala code,
why does Guardrail exist at all?

In two words **type safety**, as we will see it is impossible to do the wrong
thing when working with Guardrail's generated code because the compiler's
type checker will prevent us from making mistakes.

More important, when using Guardrail, we are forced to develop API first and our
OpenAPI/Swagger specification functions as the single source of truth for our
API's.

Now, without further ado, let's build a http4s server using Guardrail!

## Prerequisites and setup

For this tutorial we will need 
[sbt](http://www.scala-sbt.org/1.0/docs/Setup.html). To save some time we are
going to use the http4s g8 template, in your terminal do the following:

```shell
> sbt new http4s/http4s.g8
```

I'm going to use the defaults. If you want to have different names, the paths
and filenames in this tutorial might be different for you. When `sbt` is done
open the newly created directory. Start with deleting the standard routes since
we are not going to use them. Delete the following files and directories:
- `src/main/scala/com/example/quickstart/HelloWorld.scala`
- `src/main/scala/com/example/quickstart/Jokes.scala`
- `src/main/scala/com/example/quickstart/QuickstartRoutes.scala`
- `src/test`

In `QuickstartServer.scala`, remove the references to files you just
deleted so that we are left with the following file:

```scala
package com.example.quickstart

import cats.effect.{ConcurrentEffect, Effect, ExitCode, IO, IOApp, Timer, ContextShift}
import cats.implicits._
import fs2.Stream
import org.http4s.client.blaze.BlazeClientBuilder
import org.http4s.HttpRoutes
import org.http4s.implicits._
import org.http4s.server.blaze.BlazeServerBuilder
import org.http4s.server.middleware.Logger
import scala.concurrent.ExecutionContext.global

object QuickstartServer {

  def stream[F[_]: ConcurrentEffect](implicit T: Timer[F], C: ContextShift[F]): Stream[F, Nothing] = {
    for {
      client <- BlazeClientBuilder[F](global).stream

      httpApp = (
        HttpRoutes.empty[F] // We will add our own routes here later
      ).orNotFound

      finalHttpApp = Logger.httpApp(true, true)(httpApp)

      exitCode <- BlazeServerBuilder[F]
        .bindHttp(8080, "0.0.0.0")
        .withHttpApp(finalHttpApp)
        .serve
    } yield exitCode
  }.drain
}
```

Now let's see if we can still run the application:

```shell
> sbt run
```

The application should compile and we should see some output like this:

```shell
[scala-execution-context-global-92] INFO  o.h.s.b.BlazeServerBuilder - http4s v0.20.0 on blaze v0.14.0 started at http://[0:0:0:0:0:0:0:0]:8080/
```

Congratulations! We have a working http4s application. Right now it just sits
there doing nothing, so let us create some endpoints!

## API specifications and the Guardrail sbt plugin

For this tutorial, we are going to recreate the Hello World endpoint that we
deleted earlier. Save the following specification as 
`src/main/resources/api.yaml`:

```yaml
openapi: "3.0.0"
info:
  title: http4s Guardrail example
  version: 0.0.1
tags:
  - name: hello
paths:
  /hello:
    get:
      tags: [hello]
      x-scala-package: hello
      operationId: getHello
      summary: Returns a hello message
      responses:
        200:
          description: Hello message
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/HelloResponse'
components:
  schemas:
    HelloResponse:
      type: object
      properties:
        message:
          type: string
      required:
        - message
```

Two things to notice in the specification are:
1. We must provide an `operationId` for our operation
2. It is good practice to provide an `x-scala-package` value to group related
operations together.

For the code generation, we are going to use the 
[sbt-guardrail plugin](https://github.com/twilio/sbt-guardrail), to
`project.plugins.sbt` add the following line:

```scala
addSbtPlugin("com.twilio" % "sbt-guardrail" % "0.46.0")
```

To `build.sbt`, append the following lines:

```scala
guardrailTasks in Compile := List(
  ScalaServer(
    specPath = (Compile / resourceDirectory).value / "api.yaml",
    pkg = "com.example.quickstart.endpoints",
    framework = "http4s",
    tracing = false
  )
)
```

And add the following dependency to the existing `libraryDependencies`:

```scala
"io.circe" %% "circe-java8" % CirceVersion,
```

To see the code generator in action, run:

```shell
> sbt compile
```

And take a look in the 
`target/scala-2.12/src_managed/main/com/example/quickstart` directory, this is
where our generated code lives, lets see what is there:

The `definitions` directory contains the case classes that are used as request
and response bodies and helper code for serialization and deserialization. For
example, we defined a `HelloResponse` schema in the API specification we got a
corresponding `HelloResponse.scala` file.

The `hello` package got its name from the `x-scala-package` value.
`hello/Routes.scala` contains a `trait` with methods that we must implement. The
methods in this trait correspond the operations / `operationId`s in the API
specification.

`Http4sImplicits.scala` and `Implicits.scala` contain, well, implicits. They are
there to glue everything together.

So far so good, now we must actually implement the endpoint we generated.

## Implementing the generated endpoint

Create a new file at 
`src/main/scala/com/example/quickstart/endpoints/hello/HelloHandlerImpl.scala`
with the following content:

```scala
package com.example.quickstart.endpoints.hello

import cats.Applicative
import cats.implicits._
import com.example.quickstart.endpoints.definitions.HelloResponse

class HelloHandlerImpl[F[_] : Applicative]() extends HelloHandler[F] {
  override def getHello(respond: GetHelloResponse.type)(): F[GetHelloResponse] = {
    for {
      message <- "Hello, world".pure[F]
    } yield respond.Ok(HelloResponse(message))
  }
}
```

What we've done here is implement the generated `HelloHandler`. Looking at the
signature of the `getHello` method we can see Guardrail genius, everything is
typed! If this still doesn't click with you, try to rewrite the change the code
to respond with something else than a `200 OK` and have it compile (hint, you
can't).

Before we forget, lets add our hello routes to the application, open 
`src/main/scala/com/example/quickstart/QuickstartServer.scala` and replace it
with:

```scala
package com.example.quickstart

import cats.effect.{ConcurrentEffect, Effect, ExitCode, IO, IOApp, Timer, ContextShift}
import cats.implicits._
import com.example.quickstart.endpoints.hello.{HelloHandlerImpl, HelloResource}
import fs2.Stream
import org.http4s.client.blaze.BlazeClientBuilder
import org.http4s.HttpRoutes
import org.http4s.implicits._
import org.http4s.server.blaze.BlazeServerBuilder
import org.http4s.server.middleware.Logger
import scala.concurrent.ExecutionContext.global

object QuickstartServer {

  def stream[F[_]: ConcurrentEffect](implicit T: Timer[F], C: ContextShift[F]): Stream[F, Nothing] = {
    for {
      client <- BlazeClientBuilder[F](global).stream

      httpApp = (
        new HelloResource().routes(new HelloHandlerImpl())
      ).orNotFound

      finalHttpApp = Logger.httpApp(true, true)(httpApp)

      exitCode <- BlazeServerBuilder[F]
        .bindHttp(8080, "0.0.0.0")
        .withHttpApp(finalHttpApp)
        .serve
    } yield exitCode
  }.drain
}
```

What changed is that we added the line 
`new HelloResource().routes(new HelloHandlerImpl())` to the `httpApp`.

Now we can run the application again:

```shell
> sbt run
```

And once it's running we can test our endpoint using `curl`:

```shell
> curl http://localhost:8080/hello
{"message":"Hello, world"}%
```

🎉 Success! Everything is well in the world now. That is, until the API
requirements change...

## API specification changes

Guardrail makes changing the API specification a breeze. Earlier I said that we
were recreating the standard hello world routes provided by the g8 http4s
template. But we are missing something, namely, we want the `/hello` endpoint
to respond with any given name. Let's change the API specification at 
`src/main/resources/api.yaml` to

```yaml
openapi: "3.0.0"
info:
  title: http4s Guardrail example
  version: 0.0.1
tags:
  - name: hello
paths:
  /hello:
    get:
      tags: [hello]
      x-scala-package: hello
      operationId: getHello
      summary: Returns a hello message
      parameters:
        - $ref: '#/components/parameters/NameParam'
      responses:
        200:
          description: Hello message
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/HelloResponse'
components:
  parameters:
    NameParam:
      name: name
      in: query
      description: Name to greet
      schema:
        type: string
  schemas:
    HelloResponse:
      type: object
      properties:
        message:
          type: string
      required:
        - message
```

What changed is that we added a parameter to the `/hello` endpoint.

If we trigger the code generator again by calling:

```shell
> sbt compile
```

Guardrail will inform us that we are changing an existing file:

```shell
Warning:
  The file ~/Developer/quickstart/target/scala-2.12/src_managed/main/com/example/quickstart/endpoints/hello/Routes.scala contained different content than was expected.

  Existing file: ): F[GetHelloResponse] }\nclass HelloResource[F[_]]
  New file     : name: Option[String] = None): F[GetHelloResponse]
```

Followed by a bunch of compiler errors. This is actually the compiler telling us
that we need to change our implementation because it is out of sync with the
generated code. Nice. Open 
`src/main/scala/com/example/quickstart/endpoints/HelloHandlerImpl.scala` and
replace it with the following:


```scala
package com.example.quickstart.endpoints.hello

import cats.Applicative
import cats.implicits._
import com.example.quickstart.endpoints.definitions.HelloResponse

class HelloHandlerImpl[F[_] : Applicative]() extends HelloHandler[F] {
  override def getHello(respond: GetHelloResponse.type)(name: Option[String] = None): F[GetHelloResponse] = {
    for {
      message <- s"Hello, ${name.getOrElse("world")}".pure[F]
    } yield respond.Ok(HelloResponse(message))
  }
}
```

Now run the application again:

```shell
> sbt run
```

And once it is running we can try to get a personalized greeting:

```shell
> curl http://localhost:8080/hello\?name\=Kay
{"message":"Hello, Kay"}%
```

And that is how easy it is to update your API specification!

## Conclusion

In a few minutes we were able to create a simple REST API server with safely
typed endpoints generated from an API specification. Better yet, we now have a 
basis to build our application on. As we have seen Guardrail makes our lives
easier by forcing us to stay true to our API specification.

Guardrail is production ready IMO but can be rough around the edges sometimes.
If you like Guardrail, they are looking for 
[contributions](https://github.com/twilio/guardrail/blob/master/CONTRIBUTING.md).

You can find the finished project at 
[guardrail-http4s-tutorial](https://github.com/kkreuning/guardrail-http4s-example).

Thank you for reading.