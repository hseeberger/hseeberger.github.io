---
title: 'Distributed Tracing in Rust, Episode 2: tracing basics'
date: 2023-08-18
permalink: 2023-08-18-dist-tracing-2/
tags:
- rust
- tracing
- opentelemetry
---

In the [previous episode](https://heikoseeberger.de/2023-07-29-dist-tracing-1/) we were looking at the basics of logging with the Tokio tracing framework; now we are covering tracing.

As a quick recap, a trace represents a flow of execution through a system in some period of time. It is made up of at least a root span covering the complete flow and potentially one or more sequential sub spans which themselves can contain sub spans. More details and a formal specification can be found at [OpenTelemetry](https://opentelemetry.io/docs/concepts/observability-primer/).

The following picture shows a simple example of a trace visualized with Grafana Cloud involving two services:

![](/img/hello-tracing-rs-2.png)

To understand this trace, we have to understand how to propagate trace information – the so called trace context – across services. But before that, let's see how we can create spans in code, for which the [`tracing`](https://crates.io/crates/tracing) crate provides two ways: programatically and via the `instrument` attribute. The latter is easy to use and convenient to add spans to functions:

```rust
#[instrument(name = "hello-handler", skip(app_state))]
async fn hello(State(app_state): State<AppState>) -> impl IntoResponse {
    app_state.backend.hello().await.map_err(internal_error)
}
```

Here a span with `INFO` level (default unless overridden) named `hello-handler` (default is function name) is entered when the `hello` function is called and exited when it returns. By default all function arguments are recorded as span fields, but with `skip` we ignore `app_state`.

Sometimes it is necessary to create spans programatically. Our services use the [`tower-http`](https://crates.io/crates/tower-http) crate to add a `TraceLayer` to our [`axum`](https://crates.io/crates/axum) based web server. That `TraceLayer` creates a span for every received request which we customize the following way:

```rust
TraceLayer::new_for_http().make_span_with(make_span)

...

fn make_span(request: &Request<Body>) -> Span {
    let headers = request.headers();
    info_span!("incoming request", ?headers)
}
```

We use the `info_span!` macro to create a span with `INFO` level, we name it `"incoming request"` and we add the headers – its `Debug` implementation via `?` – as field.

Now that we know how to create traces inside a single service, let's see how we can deal with inter-service communication. In our example we have two services: when the `hello-tracing-gateway` service receives a HTTP request, it calls the `hello-tracing-backend` service via gRPC and then responds via HTTP.

Each service uses a `Subscriber` emitting spans. Obviously we somehow have to propagate and connect the spans – this is known as [trace context propagation](https://opentelemetry.io/docs/concepts/signals/traces/#context-propagation) – and the way we want to do it is via the OpenTelemetry standard.

Luckily there are a couple of helpful Rust crates for this purpose:
- [`opentelemetry`](https://crates.io/crates/opentelemetry): the standard in Rust
- [`tracing-opentelemetry`](https://crates.io/crates/tracing-opentelemetry): connect spans from multiple subscribers to an OpenTelemetry trace
- [`opentelemetry-http`](https://crates.io/crates/opentelemetry-http): trace context propagation via HTTP

For trace context propagation we need two sides: the sender and the receiver of the trace context.

Let's first look at the receiver, i.e. the `hello-tracing-backend` service. As the communication happens via gPRC which uses HTTP/2 as transport protocol, we can make use of `opentelemetry-http` and its `HeaderExtractor`:

```rust
/// Trace context propagation: associate the current span with the OTel trace of the given request,
/// if any and valid.
pub fn accept_trace(request: Request<Body>) -> Request<Body> {
    // Current context, if no or invalid data is received.
    let parent_context = global::get_text_map_propagator(|propagator| {
        propagator.extract(&HeaderExtractor(request.headers()))
    });
    Span::current().set_parent(parent_context);

    request
}
```

This function taking a `Request` and returning it unchanged, uses the request headers to create a `HeaderExtractor` which again is used in a call to `get_text_map_propagator` from the `opentelemetry` API. `propagator.extract` then returns an `opentelemetry::Context` which we use to set as the parent of the current `Span` from `tracing`. So if the headers contain valid OpenTelemetry trace context values, these are used to associate the current span with the overall OpenTelemetry trace.

We use this function in our tonic server in `hello-tracing-backend`:

```rust
pub async fn serve(config: Config) -> Result<()> {
    ...

    let app = Server::builder()
        .layer(
            ServiceBuilder::new()
                .layer(TraceLayer::new_for_grpc().make_span_with(make_span))
                .map_request(accept_trace),
        )
        .add_service(v0::hello());

    ...
}
```

Here we use `map_request` from `tower::ServiceBuilder` to add a layer before the actual gRPC service which not only adds the above mentioned tracing to requests but also uses the above `accept_trace` function to propagate the trace context.