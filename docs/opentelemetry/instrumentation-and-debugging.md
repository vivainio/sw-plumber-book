# Auto-Instrumentation and Debugging the Pipeline

Manual instrumentation — calling `tracer.start_span()` directly — coexists
with **auto-instrumentation**, which produces spans for common libraries
(HTTP clients/servers, database drivers, message queues) without touching
application source at all. The mechanism differs sharply by language, but
every approach converges on the same output: standard OTLP, indistinguishable
on the wire from a span someone wrote by hand.

## How spans appear without a `tracer.start_span()` call

- **Java**: a `-javaagent:opentelemetry-javaagent.jar` flag attaches at
  JVM startup and uses `java.lang.instrument` plus ByteBuddy to rewrite
  bytecode as classes load — literally injecting span-start/span-end
  calls into `HttpClient.send()`, JDBC drivers, and Kafka clients before
  they ever execute a line of their original bytecode. Nothing in your
  source or your dependencies' source changes; the class files loaded
  into the JVM are not the ones on disk.
- **Python**: `opentelemetry-instrument` monkey-patches target modules at
  import time — replacing `requests.Session.send`, `django`'s request
  handling, or `psycopg2`'s cursor execution with a wrapper that starts a
  span, calls the original function, and ends the span around it. Import
  order matters here in a way it doesn't for the Java agent, since the
  patch has to land before application code imports and binds a
  reference to the original function.
- **.NET**: hooks into `DiagnosticSource`, an eventing facility already
  built into ASP.NET Core and `HttpClient`, so instrumentation subscribes
  to events the runtime already emits rather than rewriting anything —
  closer to a built-in observer pattern than bytecode manipulation.
- **eBPF-based agents** (Grafana Beyla, Odigos) skip language runtimes
  entirely: they attach to kernel probes on socket syscalls and TLS
  library functions, reconstructing HTTP requests and trace context from
  observed bytes with **zero code changes and no language-specific
  agent at all**. The tradeoff is coarser, protocol-level visibility —
  spans built from what crossed a socket — instead of the precise,
  semantically-labeled spans a language-aware agent can produce (e.g.
  knowing the Spring controller method name, not just the URL path).

All four are producing exactly the `Span` messages described in
[Traces and Context Propagation](traces-and-context.md), serialized as
exactly the [OTLP](otlp-protocol.md) described earlier — the instrumentation
approach only changes *how* the span gets created, never its shape on the
wire.

## A troubleshooting checklist

Knowing the pipeline turns "telemetry isn't showing up" from a shrug into
a short, ordered list of things to check, roughly in the order they're
worth ruling out:

1. **Is an SDK registered at all?** API-only calls (no SDK configured)
   are silent no-ops by design — see the
   [section overview](index.md#why-it-exists-two-competing-projects-merged).
   A missing SDK produces zero errors and zero spans.
2. **Did context cross a boundary without being carried along?** A
   `Thread`, goroutine, async callback, or message-queue hop that doesn't
   explicitly propagate `Context` orphans the child work into its own new
   root trace instead of continuing the parent's — see
   [Where propagation silently breaks](traces-and-context.md#where-propagation-silently-breaks).
   The symptom is a trace that looks truncated, not absent.
3. **Is the exporter queue overflowing?** A `BatchSpanProcessor` under
   more load than its background thread can drain drops spans silently
   and increments an internal counter nobody's dashboard is watching by
   default — see
   [SDK Pipeline and the Collector](sdk-and-collector.md#processors-what-happens-the-instant-a-span-ends).
4. **Is the endpoint/port/transport mismatched?** Port `4317` is gRPC;
   port `4318` is HTTP. Pointing a gRPC exporter at the HTTP port (or
   vice versa) fails as a connection or protocol error that has nothing
   to do with instrumentation being wrong — see
   [Three transports, one payload](otlp-protocol.md#three-transports-one-payload).
5. **Do two backends disagree about the same metric's value?** Check
   whether one side is reading `CUMULATIVE` data points as if they were
   `DELTA`, or vice versa — see
   [Aggregation temporality](metrics-and-logs.md#aggregation-temporality-the-field-that-causes-cross-backend-disagreements).
6. **Is a trace missing spans from one specific service, while others
   show up fine?** That's rarely a backend problem — it's almost always
   that one service's SDK isn't registered, or its exporter can't reach
   the Collector/backend at all, so *its* spans never left the process
   even though every other service's did.

None of it is guesswork once you know what's actually being built,
decided, and sent at each stage of the pipeline.
