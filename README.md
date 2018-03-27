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

Usage of Http Client
--------------------

Whenever any of your services or HTTP endpoints need to make use of an HTTP Client, make sure that you create only one instance and pass it along.

Given the following service:

```scala
class MyService[F[_]: Sync](client: Client[F]) {

  val retrieveSomeData: Stream[F, A] = {
    val request = ???
    client.streaming[Byte](request)(_.body)
  }

}
```

You'll create the client where you start the HTTP server using the `Http1Client.stream` (uses `Stream.bracket` under the hood):

```scala
for {
  client   <- Http1Client.stream[F]())
  service  = new MyService[F](client)
  exitCode <- BlazeBuilder[F]
                .bindHttp(8080, "0.0.0.0")
                .serve
} yield exitCode
```

The same applies to the usage of any of the Fs2 data structures such as `Topic`, `Queue` and `Promise`. Create it on startup and pass it along wherever needed.

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

***NOTE***: *Don't combine plain `HttpService` with `AuthedService`. Use `mountService` from Server Builder instead for the latter to avoid conflicts since the AuthedService protects an entire namespace and not just an endpoint.*

***NOTE 2***: Since `Http4s 0.18.1` you can use `AuthMiddleware.withFallThrough(authUser)` allowing you to combine plain services with authenticated services.

HTTP Middleware Composition
---------------------------

`HttpMiddleware[F[_]]` is also a plain function. Basically an alias for `HttpService[F] => HttpService[F]`. So you can compose it.

```scala
def middleware: HttpMiddleware[F] = {
  {(service: HttpService[F]) => GZip(service)(F)} compose
    { service => AutoSlash(service)(F) }
}
```

Fs2 Scheduler
-------------

It's a very common practice to have an `fs2.Scheduler` as an implicit parameter that many of your services might use to manage time sensible processes. So it makes sense to create it at the the server startup time:


```scala
override def stream(args: List[String], requestShutdown: F[Unit]): Stream[F, ExitCode] =
  Scheduler(corePoolSize = 2).flatMap { implicit scheduler =>
    for {
      exitCode <- BlazeBuilder[F]
                    .bindHttp(8080, "0.0.0.0")
                    .serve
    } yield exitCode
  }
```

Encoders / Decoders
-------------------

`Http4s` exposes two interfaces to encode and decode data, namely `EntityDecoder[F, A]` and `EntityEncoder[F, A]`. And most of the time Json is the data type you're going to be working with. Here's where `Circe` shines and integrates very well.

#### Circe

You need 3 extra dependencies: `circe-core`, `circe-generic` and `http4s-circe`. One option is to define the json codecs in the package object where you define all your `HttpService`s. This is my setup with a workaround (see https://github.com/http4s/http4s/issues/1648):

```scala
import cats.effect.Sync
import io.circe.{Decoder, Encoder}
import org.http4s.{EntityDecoder, EntityEncoder}
import org.http4s.circe.{jsonEncoderOf, jsonOf}

package object http {
  implicit def jsonDecoder[F[_]: Sync, A <: Product: Decoder]: EntityDecoder[F, A] = jsonOf[F, A]
  implicit def jsonEncoder[F[_]: Sync, A <: Product: Encoder]: EntityEncoder[F, A] = jsonEncoderOf[F, A]
}
```

And then you can use the case class auto derivation feature by just importing `io.circe.generic.auto._` in your `HttpService`s.

If you also want to support value classes out of the box, these two codecs will be helpful (you need an extra dependency `circe-generic-extras`):

```scala
import io.circe.generic.extras.decoding.UnwrappedDecoder
import io.circe.generic.extras.encoding.UnwrappedEncoder

implicit def valueClassEncoder[A: UnwrappedEncoder]: Encoder[A] = implicitly
implicit def valueClassDecoder[A: UnwrappedDecoder]: Decoder[A] = implicitly
```

#### Streaming Json Parsers

- Jawn Fs2: https://github.com/http4s/jawn-fs2
- Circe Fs2: https://github.com/circe/circe-fs2

Error Handling
--------------

- `MonadError` -> `F[A]`
- `Either` -> `F[Error Either A]`
- Both Either and MonadError (adapt to Throwable)

Authentication
--------------

- `AuthedService[T, F[_]]`
- `AuthedMiddleware[F[_], T]`
- `AuthedRequest[F[_], T]`
