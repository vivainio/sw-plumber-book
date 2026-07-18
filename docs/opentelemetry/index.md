---
icon: lucide/satellite-dish
---

# OpenTelemetry

"Add OpenTelemetry" usually means: install an SDK, call it from application
code (or let an agent do it without touching code at all), and somewhere
downstream a dashboard shows traces, metrics, and logs. What actually
happens in between is a specific, well-documented data model, a wire
protocol, and a pipeline of buffering and batching components — none of it
mysterious, all of it worth knowing when a trace goes missing or a
collector falls over. This section is about that machinery, chapter by
chapter:

- **[Traces and Context Propagation](traces-and-context.md)** — what a
  span actually contains, how a `trace_id` survives a network hop via the
  W3C `traceparent` header, and why traces silently break at thread,
  queue, and async boundaries.
- **[The OTLP Wire Protocol](otlp-protocol.md)** — the actual protobuf
  messages sent over the wire, gRPC versus HTTP transports, and what a raw
  export request looks like in both binary and JSON form.
- **[Metrics and Logs](metrics-and-logs.md)** — instruments, aggregation
  temporality, and how a log line gets a `trace_id` stamped onto it for
  free.
- **[SDK Pipeline and the Collector](sdk-and-collector.md)** — what
  happens between `span.end()` and a byte hitting the network: processors,
  samplers, exporters, and why a separate Collector process exists at all.
- **[Auto-Instrumentation and Debugging the Pipeline](instrumentation-and-debugging.md)** —
  how spans appear without `tracer.start_span()` ever being called, and a
  concrete checklist for the ways this pipeline fails in practice.

## Why it exists: two competing projects merged

Before 2019 there were two incompatible ways to instrument code for
distributed tracing: **OpenTracing** (an instrumentation API, vendor-neutral
but with no reference implementation) and **OpenCensus** (Google's project,
which bundled an API *and* an opinionated SDK with built-in exporters).
Library authors who wanted to support both had to instrument twice, and
users were stuck picking a side. OpenTelemetry is the 2019 merger of the
two, and it inherited a design decision from that history that still shapes
everything downstream: **the API and the SDK are separate artifacts.**

Application and library code depends only on the API — `tracer.start_span()`,
`meter.create_counter()`. If no SDK is registered, those calls are no-ops
with near-zero overhead (a `Tracer` that returns a `Span` whose methods do
nothing). The actual behavior — sampling, batching, exporting — is supplied
by an SDK that's wired in once, at application startup, by whoever owns the
process. This is why a library can safely add OpenTelemetry calls without
forcing a dependency on any particular backend, and why "OpenTelemetry" is
correctly described as a specification plus API/SDK implementations, not a
product you point at a dashboard.

## The three signals share one shape

Traces, metrics, and logs are handled by parallel sets of components —
`TracerProvider`/`Tracer`/`Span`, `MeterProvider`/`Meter`/instrument,
`LoggerProvider`/`Logger`/`LogRecord` — and every piece of telemetry, of any
signal, is stamped with two pieces of shared context before it goes
anywhere:

- **Resource** — attributes describing the *process emitting the
  telemetry*, attached once per SDK and never per-event: `service.name`,
  `service.version`, `host.name`, `k8s.pod.name`, `cloud.region`. This is
  how a backend groups a flood of spans and metrics back into "which
  service, which instance."
- **InstrumentationScope** — which library or code path produced this
  particular signal: `name` and `version` of the instrumentation itself
  (e.g. `io.opentelemetry.instrumentation.requests`, not your app). This is
  what lets a backend tell "a span from your handler" apart from "a span
  the HTTP client library generated automatically."

Everything else — the fields on a span, the fields on a metric data point,
the fields on a log record — is signal-specific, and that's where the rest
of this section goes next.
