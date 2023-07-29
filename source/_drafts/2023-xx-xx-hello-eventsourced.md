---
title: 'Hello, EventSourced!'
date: 2023-01-10
tags:
- rust
- async
---

In late 2022 it was announced that [async fn in trait MVP comes to nightly](https://blog.rust-lang.org/inside-rust/2022/11/17/async-fn-in-trait-nightly.html). That resulted in quite some excitement in the Rust community. I also got excited, because I had been waiting for this feature to properly start my [EventSourced](https://github.com/hseeberger/eventsourced/) project.

But I also had some doubts whether the current state of this and related features were already mature enough to use it in anger. Spoiler alert: along the way I ran into some issues, but none turned out as a roadblock. Put another way, for me it was and is a success story and I think I might talk about my experiences with "async fn in trait" in a future post.

In this post though, I would like to give a high-level introduction to EventSourced, a framework for building applications using Event Sourcing (ES) and possibly CQRS. These architecture styles/patterns were the predominant ones for me over the last ten years, when I was working for [Lightbend](https://www.lightbend.com/) and using [Akka](https://akka.io/) for mostly every software product I built. To no surprise, EventSourced is heavily inspired by Akka Persistence – actually it's quite similar to Akka Persistence, but without actors.

# Concepts

EventSourced, amongst other things, makes use of the following concepts, which I think are pretty common in the ES/CQRS domain:
- entity: can be uniquely identified and holds state
- command: signals the intent that something should happen
- event: signals that something has happened
- event log: used to store and query events
- snapshot store: used to store state snapshots

# Usage

EventSourced lets us `spawn` entities, identified by a `Uuid`, which are kind of "running" instances of implementations of the `EventSourced` trait. That one defines types for commands, events, snapshot state and command handler errors as well as methods for command handling, event handling and setting a snapshot state.

Let's take a look at a very simple example, a counter: it can handle `Inc` and `Dec` commands taking into account over and underflow and it emits `Increased` and `Decreased` events.

```rust
#[derive(Debug, Default)]
pub struct Counter {
    value: u64,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Cmd {
    Inc(u64),
    Dec(u64),
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Evt {
    Increased(u64),
    Decreased(u64),
}

#[derive(Debug, Clone, Copy, Error)]
pub enum Error {
    #[error("Overflow: value={value}, increment={inc}")]
    Overflow { value: u64, inc: u64 },

    #[error("Underflow: value={value}, decrement={dec}")]
    Underflow { value: u64, dec: u64 },
}

impl EventSourced for Counter {
    type Cmd = Cmd;
    type Evt = Evt;
    type State = u64;
    type Error = Error;

    /// Command handler, returning the to be persisted event or an error.
    fn handle_cmd(&self, cmd: Self::Cmd) -> Result<impl IntoTaggedEvt<Self::Evt>, Self::Error> {
        let value = self.value;

        match cmd {
            Cmd::Inc(inc) if inc > u64::MAX - value => Err(Error::Overflow { value, inc }),
            Cmd::Inc(inc) => Ok(Evt::Increased(inc)),

            Cmd::Dec(dec) if dec > value => Err(Error::Underflow { value, dec }),
            Cmd::Dec(dec) => Ok(Evt::Decreased(dec)),
        }
    }

    /// Event handler, also returning whether to take a snapshot or not.
    fn handle_evt(&mut self, evt: Self::Evt) -> Option<Self::State> {
        match evt {
            Evt::Increased(inc) => self.value += inc,
            Evt::Decreased(dec) => self.value -= dec,
        }

        // No snapshots.
        None
    }

    fn set_state(&mut self, _state: Self::State) {
        // This method cannot be called as long as `handle_evt` always returns `None`.
        panic!("No snapshots");
    }
}
```

Once we have an `EntityRef`, which is returned by `spawn`, we can pass commands to the represented entity by invoking `handle_cmd`:


