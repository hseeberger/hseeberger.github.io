---
layout: post
title:  "How to (not) use akka-testkit"
---

Testing actors is different from "traditional" testing of objects or functions. First, the only way
to interact with actors is via asynchronous messaging. This means that we can't simply call a method
or function and compare the actual result with the expected. Second – thanks to the baked in
concurrency – actors behave in a nondeterministic fashion.

[Akka](https://akka.io) comes with a neat module for facilitating testing of actors named
"akka-testkit". It has been created quite a while ago and we – including the authors – have learned
some lessons since then. I'm going to introduce the tools provided by "akka-testkit" alongside with
some recommendations on which to use and how.

# TestActorRef

Akka – the core "akka-actor" module, to be more precise – prevents us from getting access to an
instance of an actor. Yet "akka-testkit" introduces the `TestActorRef` which gives access to the
`underlyingActor` instance.

While this might sound like a great idea, it turns out the opposite, because the `TestActorRef`
brings its special dispatcher – the `CurrentThreadDispatcher` – which turns all interactions with
the actor into synchronous ones. Clearly testing within this friendly sandboxed environment is far
from real conditions, namely asynchrony and nondeterminism.

Usually not getting access to an instance of an actor is not an issue, because it should always be
possible to separate our business logic from actor logic, e.g. by defining the business logic as
methods in the companion object of the actor. That way we can still test the business logic the
"traditional" way. And for testing the actor logic we can use `TestProbe`s – another tool provided
by "akka-testkit" and introduced below.

> Avoid using `TestActorRef`!

# TestKit

As mentioned above, we can only interact with actors via asynchronous messaging. Therefore the
message protocol of an actor – the messages it receives and sends – is its de facto API. Hence in
order to test the actor logic we have to send specific messages and expect messages sent in
response.

"akka-testkit" introduces the `TestKit` to extend our test suites from. This gives access to a
`testActor` which can be used as (implicit) sender or mocked `ActorRef` in messages sent. In
addition `TestKit` introduces a lot of so called built-in assertions which allow us to expect – or
not expect – messages sent from the actor to be tested to the `testActor`.

Here comes a simple – and admittedly a little contrived – example which not only shows how to use
`TestKit`, but also demonstrates its core issue:

``` scala
final class EchoSpec
    extends TestKit(ActorSystem("EchoSpec"))
    with WordSpecLike
    with BeforeAndAfterAll {

  "Echo" should {
    "respond with 'Hello' upon receiving 'Hello'" in {
      implicit val sender = testActor
      val echo            = system.actorOf(Echo())
      echo ! "Hello"
      expectMsg("Hello")
      echo ! "By mistake"
    }

    "respond with 'World' upon receiving 'World'" in {
      implicit val sender = testActor
      val echo            = system.actorOf(Echo())
      echo ! "World"
      expectMsg("World")
    }
  }

  override protected def afterAll() = { ... }
}
```

Can you spot the issue? Right, the second test fails, because the `testActor` is shared across all
the tests in the suite. Even though tests usually run sequentially, messages sent but not consumed
by one test – "By mistake" in our example – still remain in the inbox of the `testActor` and get
consumed by subsequent tests instead of the intended ones.

Additionally for advanced protocols we often need more `ActorRef`s which we can use like the
`testActor`. These are provided via the last tool introduced here, the `TestProbe`. So the
recommendation is to always only use test-local `TestProbe`s which avoids the aforementioned issue
of sharing a single `testActor` across tests.

> Don't use `TestKit`, instead use test-local `TestProbe`s!

# TestProbe

`TestProbe`s are more or less like `TestKit` – they actually extend it – without extending the suite
from. Instead they are created as needed for each test.

Here comes the above simple example rewritten to use test-local `TestProbe`s:

``` scala
final class EchoSpec extends WordSpec with BeforeAndAfterAll {

  private implicit val system = ActorSystem("EchoSpec")

  "Echo" should {
    "respond with 'Hello' upon receiving 'Hello'" in {
      val sender             = TestProbe()
      implicit val senderRef = sender.ref
      val echo               = system.actorOf(Echo())
      echo ! "Hello"
      sender.expectMsg("Hello")
      echo ! "This message is sent unintentionally"
    }

    "respond with 'World' upon receiving 'World'" in {
      val sender             = TestProbe()
      implicit val senderRef = sender.ref
      val echo               = system.actorOf(Echo())
      echo ! "World"
      sender.expectMsg("World")
    }
  }

  override protected def afterAll() = { ... }
}

```

As each test has its own `TestProbe`, sending more message than consuming these in the first test
doesn't influence the second. Of course we have to write one more line for each test and also call
`expectMsg` or other built-in assertions on the `TestProbe`, but this minimal additional boilerplate
usually doesn't hurt. And it will pay off, because errors like the above are sometimes almost
impossible to spot.

# Conclusion

After discussing the three main tools "akka-testkit" provides, we have to conclude that only
`TestProbe` should be used. No `TestActorRef` and no `TestKit`.

Happy hakking!
