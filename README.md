Http4s Good Practices
=====================

This is a collection of what I consider good practices that I've been learning along the way, designing and writing APIs using [Http4s](http://http4s.org/). Be aware that it could be bias towards my preferences.

Stream App
----------

It is recommended to start the Http Server by extending the given `fs2.StreamApp`. It'll handle resources cleanup automatically for you. Example:

```scala
class HttpServer[F[_]: Effect] extends StreamApp[F] {

  override def stream(args: List[String], requestShutdown: F[Unit]): Stream[F, ExitCode] =
    for {
      exitCode <- BlazeBuilder[F]
                    .bindHttp(8080, "0.0.0.0")
                    .mountService(httpServices)
                    .serve
    } yield exitCode

}
```

Also notice that I chose to abstract over the effect. This gives you the flexibility to choose your effect implementation only once in exactly one place. For example:

```scala
import monix.eval.Task

object Server extends HttpServer[Task]
```
Or

```scala
import cats.effect.IO

object Server extends HttpServer[IO]
```

Fs2 Scheduler
-------------

TODO: Explain why it's good to create a Scheduler in the StreamApp.

Usage of Http Client
--------------------

Whenever any of your services or HTTP endpoints need to make use of an HTTP Client, make sure that you create the object only once and pass it along.

Given the following service:

```scala
class MyService[F[_]: Sync](client: Client[F]) {

  val retrieveSomeData: Stream[F, A] = {
    val request = ???
    client.streaming[Byte](request)(_.body)
  }

}
```

You'll create the client where you start the HTTP server:

```scala
for {
  client   <- Stream.eval(Http1Client[F]())
  service  = new MyService[F](client)
  exitCode <- BlazeBuilder[F]
                .bindHttp(8080, "0.0.0.0")
                .serve
} yield exitCode
```

The same applies to the usage of any of the Fs2 data structures such as `Topic`, `Queue` and `Promise`. Created on startup and pass it along wherever needed.

HTTP Services Composition
-------------------------

`HttpService[F[_]]` is an alias for `Kleisli[OptionT[F, ?], Request[F], Response[F]]` so it is just a function that you can compose. Here's where `cats.SemigroupK` comes in handy.

Given the following http services, you can combine them into one HttpService using the `<+>` operator from `SemigroupK`.

```scala
val oneHttpService: HttpService[F] = ???
val twoHttpService: HttpService[F] = ???
val threeHttpService: HttpService[F] = ???

val httpServices: HttpService[F] = (
  oneHttpService <+> twoHttpService <+> threeHttpService
)
```

HTTP Middleware Composition
---------------------------

`HttpMiddleware[F[_]]` is also a plain function. Basically an alias for `HttpService[F] => HttpService[F]`. So you can compose it.

```scala
def middleware: HttpMiddleware[F] = {
  {(service: HttpService[F]) => GZip(service)(F)} compose
    { service => AutoSlash(service)(F) }
}
```
