# SDK Pipeline and the Collector

Calling `span.end()` doesn't send anything over the network. Between a
span finishing and an [OTLP](otlp-protocol.md) request landing on a
backend, the finished span passes through a small, well-defined pipeline
inside the SDK — and, usually, through an entire second copy of a very
similar pipeline running as its own process: the Collector.

## Processors: what happens the instant a span ends

`span.end()` hands the finished span to a `SpanProcessor`, and which
processor is configured changes production behavior enormously:

- **`SimpleSpanProcessor`** exports synchronously, one span at a time, on
  the calling thread. It's what you want in a unit test with a console
  exporter, because output appears deterministically before the test
  assertion runs. It's a real liability in production: every traced call
  now blocks on a network write to the backend before it can return.
- **`BatchSpanProcessor`** is what production actually uses: it drops
  the finished span into an in-memory queue and returns immediately. A
  background thread flushes the queue to the exporter when either a
  timer fires (default 5s), the batch hits a max size (default 512), or
  the queue fills up entirely — in which case new spans are dropped and
  a dropped-spans counter increments. That's a real, silent data-loss
  mode: a service under enough load to fill the queue faster than the
  background thread can drain it will quietly lose spans with no error
  raised anywhere in application code.

## Samplers: deciding what the processor even sees

Before a span is even created with real work behind it, a `Sampler`
decides whether it should be recorded at all — this is the head-based
sampling decision described in
[Traces and Context Propagation](traces-and-context.md), and it's
implemented as a small, composable interface:

- `AlwaysOn` / `AlwaysOff` — trivial, mostly for tests or "definitely
  don't sample this dev environment."
- `TraceIdRatioBased(0.1)` — samples a deterministic fraction of traces,
  using the trace ID itself as the source of randomness so the decision
  is reproducible without any shared state.
- `ParentBased(root=...)` — the default in practice: if this span has a
  parent with a sampling decision already made, honor it; only consult
  the wrapped sampler (e.g. a ratio-based one) when starting a brand-new
  root trace. This is what keeps a trace from having sampled spans in
  service A and unsampled gaps in service B.

## Exporters: the only backend-aware piece

The exporter is the only component in this whole pipeline that knows
about a specific wire format or destination —
`OTLPSpanExporter`, `ConsoleSpanExporter`, `JaegerExporter`,
`ZipkinExporter`. Everything above it — API, SDK, processor, sampler — is
backend-agnostic. Swapping Jaeger for Honeycomb is a one-line change to
which exporter is registered at startup, not a rewrite of
instrumentation, which is the entire point of separating the API from
the SDK described in the [section overview](index.md).

## Why a separate Collector process exists at all

An SDK could export straight to a backend, and for a toy setup it does.
The **OpenTelemetry Collector** exists as a separate deployable (a Go
binary, run as a sidecar, a per-host daemon, or a standalone cluster)
because several things are much better done once, out of the
application's hot path, than duplicated inside every service's SDK. Its
internal architecture is the *same* receiver → processor → exporter
shape as the SDK's own internals, just running out-of-process and
configured in YAML instead of code:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  memory_limiter:
    limit_mib: 512
  tail_sampling:
    policies:
      - name: keep-errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: keep-slow
        type: latency
        latency: { threshold_ms: 500 }
  batch: {}

exporters:
  otlp:
    endpoint: backend.example.com:4317
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch]
      exporters: [otlp]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
```

A `receiver` speaks OTLP (or Jaeger, Zipkin, a Prometheus scrape target,
and others) inbound, terminating exactly the same gRPC/HTTP endpoints an
SDK exporter would otherwise dial directly. A chain of `processors` can
batch, drop, redact PII, or — as configured above — hold spans in memory
long enough to make a **tail-sampling** decision no single SDK instance
could ever make on its own, because an individual process's SDK only
ever sees the spans *it* produced, never the whole trace across every
service that touched the request. An `exporter` re-serializes outbound to
one or more backends, and a pipeline can fan the same signal out to
several exporters at once — the `traces` pipeline above could just as
easily list both a tracing backend and a logging exporter for debugging.

Pointing every service's SDK at the Collector instead of a backend
directly buys three concrete things: one place to change backends
without touching application deployments, one place to apply sampling
and redaction policy consistently, and a buffer that absorbs a backend
outage without every application process's `BatchSpanProcessor` queue
backing up and dropping spans on its own. The cost is an extra hop and an
extra process to operate — which is why small setups reasonably skip it
and point SDKs straight at a backend that speaks OTLP natively, and why
that decision is worth revisiting the moment tail sampling or multi-backend
fan-out becomes a real requirement.
