# Traces and Context Propagation

A trace is not a stored object anywhere, and it isn't built by a single
piece of code either. It's an emergent property of many independently
created **spans**, from many processes, that happen to share an ID.
Understanding tracing means understanding two things: the exact shape of a
span, and the mechanism that hands the same ID to every process on a
call's path.

## The span

A span is a specific record shape defined by the OpenTelemetry
specification, not a log line with a duration attached. Every field on it
exists for a reason:

| Field | Purpose |
|---|---|
| `trace_id` | 16 bytes, shared by every span in one request's call graph |
| `span_id` | 8 bytes, unique to this span |
| `parent_span_id` | links this span to its caller, forming the tree |
| `name` | low-cardinality operation name, e.g. `GET /orders/:id`, not `GET /orders/482` |
| `kind` | `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER`, or `INTERNAL` — tells a backend how to pair spans across a network hop |
| `start_time_unix_nano` / `end_time_unix_nano` | wall-clock nanoseconds since epoch |
| `attributes` | key/value pairs, e.g. `http.status_code=200`, `db.statement=...` |
| `events` | timestamped points *within* the span's lifetime — a log line with a place to live, e.g. `exception` |
| `links` | references to *other* spans that aren't the parent — the mechanism batch and async fan-in use |
| `status` | `UNSET`, `OK`, or `ERROR`, plus an optional message |

Nothing computes the tree at creation time. A trace is reconstructed at
query time in the backend, by grouping every span that shares a
`trace_id` and then following each one's `parent_span_id` to lay them out
as a graph. This is why a "trace" can have gaps in it (a service that
wasn't instrumented, or whose spans never arrived) without the whole
thing falling over — it just shows up as a child span with no visible
parent.

## Getting the trace_id across a network hop

The hard problem tracing actually solves is **context propagation**:
service A starts a span, calls service B over HTTP, and B needs to
continue the *same* trace rather than start a new one. OpenTelemetry
doesn't invent its own header for this — it implements
[**W3C Trace Context**](https://www.w3.org/TR/trace-context/), a W3C
Recommendation (finalized 2020) that predates and is independent of
OpenTelemetry itself — Trace Context defines the header format, and any
tracing system is free to implement it, which is exactly why a request
can hop through OpenTelemetry-instrumented and non-OpenTelemetry
services alike without losing its trace ID. So a `traceparent` header
rides along on the outbound request:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  └───────────trace-id───────────┘ └──parent-id──┘ └flags┘
           version                                              (01 = sampled)
```

That's the entire propagation mechanism: 55 bytes of ASCII hex. The
receiving service's instrumentation parses it, and instead of starting a
new root span, starts a child span whose `parent_span_id` is the
`00f067aa0ba902b7` from the header — stitching two independent processes'
spans into one trace without either side needing to know the other's
implementation.

Two companion headers ride along the same hop, and only one of them comes
from the same spec:

- **`tracestate`** is defined by Trace Context itself, alongside
  `traceparent`, and carries vendor-specific extra state through the same
  hop without OpenTelemetry needing to understand its contents —
  `tracestate: congo=t61rcWkgMzE,rojo=00f067aa0ba902b7`, a comma-separated
  list of `key=value` pairs each vendor owns.
- **`baggage`** is a *separate* W3C specification,
  [**W3C Baggage**](https://www.w3.org/TR/baggage/), not part of Trace
  Context — it just happens to travel alongside it in practice. It
  propagates arbitrary user-defined key/value pairs —
  `baggage: user.tier=gold,tenant.id=42` — the same way. Baggage is *not*
  attached to spans automatically; it's just carried along in-process and
  across hops, readable by any downstream instrumentation that chooses to
  read it and add it as a span attribute.

## Where propagation silently breaks

This is also the exact mechanism that breaks silently: fire off a
`Thread`, a fire-and-forget goroutine, or a message onto a queue without
carrying the current context across that boundary, and the child work
starts a brand-new trace with no parent — the classic "my trace stops at
the queue" symptom. The context lives in an in-process `Context` object
(thread-local in Python, `ContextVar`-based in Node via `async_hooks`,
goroutine-argument-based in Go because Go has no implicit thread-local);
anything that hops execution contexts without explicitly carrying it
severs the trace. Message queues are a particularly common break point
because the wire format is the application's own message schema, not
HTTP headers — propagating context across Kafka or SQS means an
instrumentation library (or your own code) has to explicitly inject
`traceparent` into message headers on the producer side and extract it on
the consumer side; nothing does this automatically the way an HTTP client
interceptor does.

## Sampling: deciding what to keep, and when

Recording every span for every request is often too expensive, so
OpenTelemetry decides whether to sample **before the span even
finishes** — at span-start time, using only the information available
then (trace ID, name, parent's sampling decision, attributes set so far).
This is *head-based sampling*, and the flag it produces is exactly that
trailing `01`/`00` in `traceparent` above: once the root service decides
"sampled," every downstream service honors that decision via
`ParentBased(root=TraceIdRatioBased(0.1))` or similar, rather than each
service re-rolling the dice and producing a trace with gaps in it.

The tradeoff head-based sampling can't avoid: you can't decide "keep this
trace because it errored" until the request is *over*, but the sampling
decision was made at the start. The workaround isn't in the SDK — it's a
feature of the Collector, covered in
[SDK Pipeline and the Collector](sdk-and-collector.md), which can buffer
*all* spans briefly and apply **tail-based sampling**: keep every trace
containing an error or a p99-latency span, discard a random slice of the
boring ones, after the fact.
