---
title: 'Distributed Tracing in Rust, Episode 1: logging basics'
date: 2023-07-29
permalink: 2023-07-29-dist-tracing-1/
tags:
- rust
- tracing
- opentelemetry
---

In order to gain insights into the inner workings of a system – put another way: to have [obervability](https://opentelemetry.io/docs/concepts/observability-primer/#what-is-observability) – we can build upon the three pillars of observability: logs, traces and metrics.

In this series we will explore how we can implement the first two – i.e. logs and traces – in [Rust](https://www.rust-lang.org/) with the [Tokio tracing framework](https://github.com/tokio-rs/tracing) and [OpenTelemetry](https://opentelemetry.io/).

Let's start with some ~~boring~~ basic terminology:

- A log event – aka log message – signifies something that has happened at some specific moment in time.
- A span records a – likely partial – flow of execution through a system and hence represents a period of time. It also functions as context for log events as well as parent for sub spans.
- A trace represents a complete flow of execution through a system from end to end, e.g. from receiving a request to sending a response. Hence it is also a span without parent spans, i.e. a root span.

Log events give us valuable insights about **what** has happened, e.g. not being able to call a downstream service in a multi-service architecture. But without traces it often is hard to understand the context, i.e. the **why**, e.g. how the request looked like or what the various arguments in the call chain leading to the failing call were. Therefore it is very important to use both logs and traces and to make sure they work seamlessly together.

In this episode we are looking at the basics of logs with the Tokio tracing framework. The next episodes will cover traces as well as correlating logs and traces.

To create a log event in code, the [`tracing` crate](https://crates.io/crates/tracing) provides macros for the respective levels, e.g. `debug!` or `error!`:

```rust
debug!(?config, "starting");

...

if let Err(error) = &result {
    error!(
        error = format!("{error:#}"),
        backtrace = %error.backtrace(),
        "hello-tracing-gateway exited with ERROR"
    );
};
```

Beside the level and message we can add arbitrary fields and such get to rich and strucutred log events. To add a field we can use `name = value` or just `name`, if `name` is in scope. The supported types are most basic ones, e.g. `i32` and `String`; by using the `?` prefix, a type's `Debug` implementation is used and `%` makes use of a type's `Display` implementation.

In order to emit log events created in code like above, we use the [`tracing-subsciber` crate](https://crates.io/crates/tracing-subscriber). It's `fmt` module offers various formatters out of which the JSON one is great for structured logging. To use it, we have to initialize the `tracing-subsciber` accordingly:

```rust
tracing_subscriber::registry()
    .with(EnvFilter::from_default_env())
    .with(fmt::layer().json())
    .try_init()
    .context("initialize tracing subscriber")
```

`tracing-subsciber` implements a pretty sophisticated layer architecture, which we will use later in greater depth, but for now the above only applies a filter layer which is initialized from environment variables – `RUST_LOG` can be used to set per module/traget levels – as well a formatter layer that logs JSON formatted representations of tracing events.

Here is an example of a log event created by the `tracing-subsciber` initialized like above, assuming `RUST_LOG` is either set globally to `debug` or at least for the given target `hello_tracing_gateway`:

```json
{
  "timestamp": "2023-07-29T13:47:08.651846Z",
  "level": "DEBUG",
  "fields": {
    "message": "starting",
    "config": "Config { api: Config { addr: 0.0.0.0, port: 8080 }, backend: Config { endpoint: \"http://localhost:8090\" }, tracing: TracingConfig { service_name: \"hello-tracing-gateway\", otlp_exporter_endpoint: \"http://localhost:4317\" } }"
  },
  "target": "hello_tracing_gateway"
}
```

All right, we have arrived at the end of this episode. In the next one we will look at traces. If you are interested in – the already fully fleshed out – example code, take a look at [hello-tracing-rs on GitHub](https://github.com/hseeberger/hello-tracing-rs/).