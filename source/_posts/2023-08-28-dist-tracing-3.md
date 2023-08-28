---
title: 'Distributed Tracing in Rust, Episode 3: tracing basics'
date: 2023-08-28
permalink: 2023-08-28-dist-tracing-3/
tags:
- rust
- tracing
- opentelemetry
---

In the [previous episode](https://heikoseeberger.de/2023-08-18-dist-tracing-2/) we were looking at the basics of tracing with the Tokio tracing framework and showed how to create traces inside a single service. Now let's see how we can deal with inter-service communication.

In our example we have two services: when the `hello-tracing-gateway` service receives a HTTP request, it calls the `hello-tracing-backend` service via gRPC and then responds via HTTP.

Each service uses a `Subscriber` emitting spans – we will look at the respective configuration later. Obviously we somehow have to propagate and connect the spans – this is known as [trace context propagation](https://opentelemetry.io/docs/concepts/signals/traces/#context-propagation) – and the way we want to do it is via the OpenTelemetry standard.

Luckily there are a couple of helpful Rust crates for this purpose:
- [`opentelemetry`](https://crates.io/crates/opentelemetry): the standard in Rust
- [`tracing-opentelemetry`](https://crates.io/crates/tracing-opentelemetry): connect spans from multiple subscribers to an OpenTelemetry trace
- [`opentelemetry-http`](https://crates.io/crates/opentelemetry-http): trace context propagation via HTTP

For trace context propagation we need two sides: the sender and the receiver of the trace context.

## Receiving the Trace Context

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

We use this function in our [tonic](https://crates.io/crates/tonic) based gRPC server in `hello-tracing-backend`:

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

Here we use `tower::ServiceBuilder` to add a layer before the actual gRPC service `v0::hello`. That layer not only adds – like explained in the [previous episode](https://heikoseeberger.de/2023-08-18-dist-tracing-2/) – tracing to requests via the `TraceLayer`, but also uses the above `accept_trace` function in the `map_request` call to propagate the trace context.

## Sending the Trace Context

All right, now let's look at the sender, i.e. the `hello-tracing-gateway`. This time we have to stick to the tonic API – namely an `Interceptor` – to send the trace context. Therefore we cannot use `opentelemetry-http` and its `HeaderInjector`, but we have to roll our own:

```rust
struct MetadataInjector<'a>(&'a mut MetadataMap);

impl Injector for MetadataInjector<'_> {
    fn set(&mut self, key: &str, value: String) {
        match MetadataKey::from_bytes(key.as_bytes()) {
            Ok(key) => match MetadataValue::try_from(&value) {
                Ok(value) => {
                    self.0.insert(key, value);
                }

                Err(error) => warn!(value, error = format!("{error:#}"), "parse metadata value"),
            },

            Err(error) => warn!(key, error = format!("{error:#}"), "parse metadata key"),
        }
    }
}
```

With that `Injector` implementation we can define a function taking a `Request` and returning it unchanged and wrapped in an `Ok`. Again we use `get_text_map_propagator` from the `opentelemetry` API, this time to call `propagator.inject_context` with an OpenTelemetry trace context built from the current `Span` from `tracing`:

```rust
/// Trace context propagation: send the trace context by injecting it into the metadata of the given
/// request.
pub fn send_trace<T>(mut request: Request<T>) -> Result<Request<T>, Status> {
    global::get_text_map_propagator(|propagator| {
        let context = Span::current().context();
        propagator.inject_context(&context, &mut MetadataInjector(request.metadata_mut()))
    });

    Ok(request)
}
```

This `send_trace` is then used as tonic interceptor, such that for outgoing service calls the trace context is propagated:

```rust
let endpoint = Endpoint::from_str(&self.config.endpoint)
    .with_context(|| format!("create endpoint {}", self.config.endpoint))?;
let channel = endpoint
    .connect()
    .await
    .with_context(|| format!("connect to endpoint {}", self.config.endpoint))?;
let mut client = HelloClient::with_interceptor(channel, send_trace);
```

## Configuring the `Subscriber`

In the [first episode](https://heikoseeberger.de/2023-07-29-dist-tracing-1/) we introdued the tracing_subscriber crate and used it to set up a formatter layer logging JSON formatted representations of tracing events. Let's now extend the setup and add a layer exporting tracing data via the [OpenTelemetry Protocol (OTLP)](https://opentelemetry.io/docs/specs/otlp/) to a configurable backend which then can be used to visualize traces:

```rust
/// Create an OTLP layer exporting tracing data.
fn otlp_layer<S>(config: TracingConfig) -> Result<impl Layer<S>>
where
    S: Subscriber + for<'span> LookupSpan<'span>,
{
    let exporter = opentelemetry_otlp::new_exporter()
        .tonic()
        .with_endpoint(config.otlp_exporter_endpoint);

    let trace_config = trace::config().with_resource(Resource::new(vec![KeyValue::new(
        "service.name",
        config.service_name,
    )]));

    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(exporter)
        .with_trace_config(trace_config)
        .install_batch(runtime::Tokio)
        .context("install tracer")?;

    Ok(tracing_opentelemetry::layer().with_tracer(tracer))
}
```

Here we are using the [`opentelemetry_otlp` crate](https://crates.io/crates/opentelemetry-otlp) to create an exporter based on tonic/gRPC with configurable endpoint and service name. With this exporter we create a `Tracer` from the `opentelemetry` API which is then used to create the layer.

All that is left to do is add this OTLP layer to the tracing_subscriber:

```rust
tracing_subscriber::registry()
    .with(EnvFilter::from_default_env())
    .with(fmt::layer().json())
    .with(otlp_layer(config)?)
    .try_init()
    .context("initialize tracing subscriber")
```

With this setup and a properly configured tracing backend – we use Grafana Tempo – we can visualize traces like already shown in the [previous episode](https://heikoseeberger.de/2023-08-18-dist-tracing-2/):

![](/img/hello-tracing-rs-2.png)

Now we are almost there. What's left to do is correlate logs and traces, but we'll leave that to the next episode. As usual, the already fully fleshed out example code can be found at [hello-tracing-rs on GitHub](https://github.com/hseeberger/hello-tracing-rs/).
