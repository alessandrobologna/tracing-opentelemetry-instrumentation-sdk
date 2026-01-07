# Integrating an Axum App with OpenTelemetry Tracing

This repository provides two building blocks for Axum tracing:

- `init-tracing-opentelemetry`: sets up a `tracing` subscriber and an OpenTelemetry exporter.
- `axum-tracing-opentelemetry`: Axum middleware that extracts incoming trace context and creates spans.

For a working reference, see `examples/axum-otlp/`.

## 1) Add dependencies

In an external app, add the crates and enable the init features that match the desired exporter:

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tracing = "0.1"

axum-tracing-opentelemetry = "0.32"
init-tracing-opentelemetry = { version = "0.34", features = ["tracing_subscriber_ext"] }
```

If the exporter needs TLS or metrics, enable `tls` and `metrics` on `init-tracing-opentelemetry`.

### AWS X-Ray propagation (using this fork branch)

To accept `X-Amzn-Trace-Id` headers, enable the `xray` feature.
This is not released on crates.io yet, so patch `crates.io` to use the fork branch:

```toml
[dependencies]
init-tracing-opentelemetry = { version = "0.34", features = ["tracing_subscriber_ext", "xray"] }

[patch.crates-io]
init-tracing-opentelemetry = { git = "https://github.com/alessandrobologna/tracing-opentelemetry-instrumentation-sdk.git", branch = "feat/re-enable-xray-propagator" }
```

At runtime, select propagators via `OTEL_PROPAGATORS`:

```bash
export OTEL_PROPAGATORS="xray,tracecontext,baggage"
```

To validate propagation, send a request with an X-Ray header:

```bash
curl -H 'X-Amzn-Trace-Id: Root=1-5f84c7a0-2c2c3c3c4d4d5e5e6f6f7070;Parent=53995c3f42cd8ad8;Sampled=1' \
  http://localhost:3000/
```

## 2) Initialize tracing and OpenTelemetry once

Initialize at process startup and keep the returned guard alive until shutdown.

```rust
#[tokio::main]
async fn main() -> Result<(), axum::BoxError> {
    let _otel_guard =
        init_tracing_opentelemetry::TracingConfig::production().init_subscriber()?;

    let app = app();
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app.into_make_service()).await?;
    Ok(())
}
```

## 3) Add Axum middleware

Use `OtelAxumLayer` to create the server span and `OtelInResponseLayer` to return trace context headers.

```rust
use axum::{routing::get, Router};
use axum_tracing_opentelemetry::middleware::{OtelAxumLayer, OtelInResponseLayer};

fn trace_filter(path: &str) -> bool {
    path != "/health"
}

fn app() -> Router {
    Router::new()
        .route("/", get(index))
        .route("/health", get(health))
        .layer(OtelInResponseLayer::default())
        .layer(OtelAxumLayer::default().filter(trace_filter))
}
```

If an endpoint must bypass all tracing-related layers, place it on a separate `Router` and `merge()` it.

To record `client.address`, enable IP extraction and serve with connect info:

```rust
use std::net::SocketAddr;
let app = app().layer(OtelAxumLayer::default().try_extract_client_ip(true));
axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>());
```

## 4) Instrument handlers and add trace IDs

- Add `#[tracing::instrument]` to handlers that should create child spans.
- Override the operation name via `tracing::Span::current().record("otel.name", "...")` when needed.
- Include the trace id in logs or responses with `axum_tracing_opentelemetry::tracing_opentelemetry_instrumentation_sdk::find_current_trace_id()`.

## 5) Configure the exporter and validate

Export traces by setting OpenTelemetry environment variables:

```bash
export OTEL_SERVICE_NAME="my-axum-service"
export OTEL_TRACES_SAMPLER="always_on"
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT="http://localhost:4317"
export OTEL_EXPORTER_OTLP_TRACES_PROTOCOL="grpc"
```

To accept and emit AWS X-Ray trace headers, enable the `xray` feature on `init-tracing-opentelemetry` and set:

```bash
export OTEL_PROPAGATORS="xray"
```

If no traces are exported, ensure the `otel::tracing` target is enabled at `trace` level (or `info` when using the `tracing_level_info` feature).

For local validation in this repo, run `mise run run-jaeger`, then start an Axum example and confirm the `traceparent` response header.
