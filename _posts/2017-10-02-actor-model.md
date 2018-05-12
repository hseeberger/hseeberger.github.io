---
layout: post
title:  The Actor Model in Akka and Akka Typed
---

The [actor model](https://en.wikipedia.org/wiki/Actor_model) was invented in the early seventies by
[Carl Hewitt](https://en.wikipedia.org/wiki/Carl_Hewitt) et al. and very successfully implemented in
the [Erlang](https://www.erlang.org) programming language. Of course it is also the foundation for
[Akka](https://akka.io) and in this post we are taking a look at how exactly Akka implements the
actor model, both in good old Akka and in bleeding edge Akka Typed. As usual we are using Scala as
our programming language of choice.

## The Actor Model

Carl Hewitt [explains](https://www.youtube.com/watch?v=7erJ1DV_Tlo) the actor as the "fundamental
unit of computation embodying processing (do things), storage (state) and communications" where
"everything is an actor" and "one actor is no actor, they come in systems". Collaboration between
actors happens via asynchronous communication, allowing for decoupling of sender and receiver: each
actor has an address and can send messages to actors for which it knows their address without being
blocked in its processing.

When an actor processes a message it has received – which is performed by its so called behavior –
it can do the following, concurrently and in any order:

1. create new actors
1. send messages to known actors
1. designate the behavior for the next message

Given this brief description of the actor model, let's now take a look at how Akka and Akka Typed
implement it thereby ignoring creating new actors, because this happens more or less the same way in
both Akka and Akka Typed.

## The Actor Model in Akka

In Akka an actor is a class extending `Actor` and its so called initial behavior is defined by the
`receive` method:

``` scala
trait Actor {
  def receive: PartialFunction[Any, Unit] // actually type alias `Receive`
}
```

The initial behavior is used to handle messages until we change it, if ever. As we can see, the
behavior is a partial function from `Any` to `Unit` which means that – although Scala and Java are
type safe languages – Akka actors are untyped: we can send any message to an actor even though it
most probably only handles specific ones – the compiler is not able to help us.

This is also reflected in the `ActorRef` trait which represents the address of an actor in Akka:

``` scala
trait ActorRef { // actually `ScalaActorRef`
  def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit
}
```

The `!` operator allows us to send a message of type `Any` to the actor identified by the respective
address. Obviously there is no way to restrict the message to some specific type. Further the
address of the sender is conveyed as an implicit parameter if possible, which is the case when
sending happens from within an actor.

Let's look at another important aspect of the behavior in Akka. It returns `Unit` which means that
designating a different behavior for the next message must happen as a side effect. And indeed each
actor has access to its `ActorContext` which offers the `become` method for that exact purpose:

``` scala
trait ActorContext {
  def become(behavior: PartialFunction[Any, Unit]): Unit // actually type alias `Receive`
}
```

### 1. Same Behavior with Mutable State

Equipped with this knowledge, let's implement a simple example: an actor which produces increasing
sequence numbers. Let's start with an actor which stores the current sequence number value in a
mutable field – Akka makes sure this cannot be shared and all access happens in a thread safe
manner:

``` scala
import akka.actor.Actor

object MutableSeqNoGenerator {

  final case object GetNext
  final case class SeqNo(seqNo: Long)
}

final class MutableSeqNoGenerator(seqNo: Long = 0) extends Actor {
  import MutableSeqNoGenerator._

  private var _seqNo = seqNo

  override def receive = {
    case GetNext =>
      sender() ! SeqNo(_seqNo)
      _seqNo += 1
  }
}
```

As the behavior closes over the mutable field we need not ever change it (the behavior).

### 2. Changing Behavior

While the above implementation represents an idiomatic pattern for Akka, we can also handle the
mutability of the sequence number value by changing the behavior instead of using a mutable field:

``` scala
import akka.actor.Actor

object SeqNoGenerator {

  final case object GetNext
  final case class SeqNo(seqNo: Long)
}

final class SeqNoGenerator(seqNo: Long = 0) extends Actor {
  import SeqNoGenerator._

  override def receive = seqNoGenerator(seqNo)

  private def seqNoGenerator(seqNo: Long): Receive = {
    case GetNext =>
      sender() ! SeqNo(seqNo)
      context.become(seqNoGenerator(seqNo + 1))
  }
}
```

In this case we change the behavior every time the actor processes a `GetNext` message and thus can
get rid of the mutable field. This pattern is particularly useful to implement actors representing
state machines.

## The Actor Model in Akka Typed

As clearly indicated by naming, Akka Typed brings type safety back to Akka. But Akka Typed also
resembles the actual definition of the actor model much closer than Akka by dropping the `Actor`
trait and conceptually treating behavior as a function from a typed message to the next behavior:

``` scala
abstract class ExtensibleBehavior[T] extends Behavior[T] {
  def receiveMessage(ctx: ActorContext[T], msg: T): Behavior[T]
  def receiveSignal(ctx: ActorContext[T], msg: Signal): Behavior[T]
}
```

As we can see, the behavior has a type parameter for the type of messages the actor handles.
Therefore `receiveMessage` is a total function instead of a partial one. Since there is no enclosing
`Actor` class, the `ActorContext` is also passed as a parameter along with the respective message.
And finally `receiveSignal` takes care of a few special messages like `PreRestart` or `Terminated`
which of course are not covered by the message type `T`.

Having a parameterized behavior we obviously also need a parameterized address:

``` scala
implicit final class ActorRefOps[-T](val ref: ActorRef[T]) extends AnyVal {
  def !(msg: T): Unit = ref.tell(msg)
}
```

Now we can only send messages of type `T` to the respective actor: good-bye `Any`!

One important difference is worth mentioning: in Akka Typed there is no implicit sender any more. While this
inevitable change – what exact type should `sender()` return? – might look like a step backwards, we
can even regard it as an advantage: we have to make our message protocols more explicit and
therefore easier to understand by including addresses for replies where needed.

Like in Akka we have two ways to actually create a behavior: a mutable – closing over mutable state –
and an immutable one. This time the second one is exclusively considered the idiomatic one, hence we
only show the following example:

``` scala
import akka.typed.{ ActorRef, Behavior }
import akka.typed.scaladsl.Actor

object SeqNoGenerator {

  final case class GetNext(replyTo: ActorRef[SeqNo])
  final case class SeqNo(seqNo: Long)

  def apply(seqNo: Long = 0): Behavior[GetNext] =
    Actor.immutable {
      case (_, GetNext(replyTo)) =>
        replyTo ! SeqNo(seqNo)
        SeqNoGenerator(seqNo + 1)
    }
}
```

Notice that we have to include the `replyTo` address in the `GetNext` message as a typed
`ActorRef[SeqNo]`, representing an address which only accepts messages of type `SeqNo`.

Using `Actor.immutable` we define a so called immutable behavior, i.e. one which does not close over
mutable state. It is typed to only handle messages of type `GetNext`. We are using pattern matching,
but only to conveniently destruct the message – the message handler function given as argument to
`Actor.immutable` is still a total one. The result of the message handler is the next behavior,
similar like using `ActorContext.become` above, but not as a side-effect which makes testing much
easier.

## Conclusion

We have looked at the ways Akka and Akka Typed implement the actor model. Both fully support the
required behavior rules, i.e. creating new actors, sending messages to known actors and designating
the next behavior. But Akka Typed not only makes the last aspect more tangible by conceptually
treating behavior as a function from message to the next behavior, but also brings back type safety.

The full source code for the examples can be found at
[github.com/hseeberger/blog-actor-model](https://github.com/hseeberger/blog-actor-model) in the
`de.heikoseeberger.blog.actormodel` package.
