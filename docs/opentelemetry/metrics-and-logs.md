# Metrics and Logs

Traces get most of the attention because they're the newest and most
visually distinctive signal, but metrics and logs go through the same
Resource/InstrumentationScope wrapping and the same
[OTLP wire protocol](otlp-protocol.md) — `ExportMetricsServiceRequest` and
`ExportLogsServiceRequest` are the same tree shape as
`ExportTraceServiceRequest`, with a different leaf message. What differs
is the leaf, and the concepts that produce it.

## Instruments and data points

A metric doesn't start life as a number sitting in a table — it starts as
an **instrument**, created once from a `Meter` and then written to
repeatedly over the process's lifetime:

- **Counter** — monotonically increasing, e.g. `requests_total.add(1)`.
- **UpDownCounter** — can go up or down, e.g. an in-flight request gauge
  maintained by incrementing on start and decrementing on completion.
- **Histogram** — records a distribution of values, e.g. request
  latency; the SDK buckets them internally.
- **Gauge** — records the current value of something that isn't
  cumulative, e.g. CPU temperature.
- **Observable** variants of each (`ObservableCounter`,
  `ObservableGauge`, ...) — instead of being written to inline, they
  register a callback that the SDK invokes at export time, useful for
  values that are cheap to *read* but awkward to *push* on every change,
  like `process.memory.usage` pulled from the OS.

At export time, each instrument's accumulated state becomes one or more
**data points** — `NumberDataPoint` for counters and gauges,
`HistogramDataPoint` for histograms — each carrying its own attributes
(so `http.requests_total{method="GET",status="200"}` and
`http.requests_total{method="POST",status="500"}` are two data points
from one Counter, not two separate instruments) and a start/end
timestamp pair.

## Aggregation temporality: the field that causes cross-backend disagreements

Every data point is tagged with an **aggregation temporality**, and it's
the single most common source of "these two dashboards show different
numbers for the same metric":

- **`CUMULATIVE`** — the value is the total since the instrument was
  created (or since the process started). This is Prometheus's native
  model: a `/metrics` scrape always reports the running total, and the
  backend computes rates by diffing consecutive scrapes itself.
- **`DELTA`** — the value is the total *since the last export*, reset to
  zero after each flush. This is more natural for push-based backends
  that want a raw increment per interval without having to remember the
  previous scrape's value themselves.

A `Counter` exported with `CUMULATIVE` temporality and read as if it were
`DELTA` (or vice versa) produces numbers that are off by orders of
magnitude in exactly the way a mis-set feature flag would — not a crash,
just quietly wrong dashboards. The OTLP `HistogramDataPoint` message
carries this explicitly:

```protobuf
message HistogramDataPoint {
  repeated KeyValue attributes = 9;
  fixed64 start_time_unix_nano = 2;
  fixed64 time_unix_nano = 3;
  fixed64 count = 4;
  double sum = 5;
  repeated fixed64 bucket_counts = 6;
  repeated double explicit_bounds = 7;
  repeated Exemplar exemplars = 8;
}
```

`explicit_bounds` are the histogram's bucket boundaries
(`[10, 50, 100, 500, ...]` milliseconds, say) and `bucket_counts` is a
parallel array of how many observations landed in each bucket — the same
shape Prometheus histograms use, which is why OTLP-to-Prometheus export
is largely a field rename rather than a real conversion. `Exemplar`
entries are the mechanism behind "click a spike in a latency histogram,
jump to an actual trace that produced it": each exemplar can carry a
`trace_id`/`span_id`, so a small sample of the raw data points behind an
aggregate keep a pointer back into tracing data.

## Logs: a bridge, not a replacement

Logs are the newest signal in OpenTelemetry and deliberately the least
invasive one. A `LogRecord` is designed as a bridge *over* whatever
logging you already have — `log4j`, `winston`, Python's `logging`
module — not a new API applications are expected to call directly:

```protobuf
message LogRecord {
  fixed64 time_unix_nano = 1;
  fixed64 observed_time_unix_nano = 11;
  SeverityNumber severity_number = 2;
  string severity_text = 3;
  AnyValue body = 5;
  repeated KeyValue attributes = 6;
  bytes trace_id = 9;
  bytes span_id = 10;
}
```

Most of that is what you'd expect from any structured logging system.
The field that isn't is `trace_id`/`span_id` sitting directly on a log
record: when a log statement is emitted from inside an active span, the
OpenTelemetry logging bridge stamps the record with the enclosing
trace's IDs at the moment of emission. That's the *entire* mechanism
behind "click a log line in your observability UI, jump straight to its
trace" — there's no correlation engine matching timestamps or fuzzy
string similarity after the fact; it's a foreign key set once, at write
time, by code that already knows which span is currently active in this
thread's context.

`observed_time_unix_nano` versus `time_unix_nano` exists because logs
are frequently ingested well after they were produced — a batch log file
shipped hours later, or a record replayed from a queue — so the protocol
distinguishes "when the event actually happened" from "when the
collection pipeline first saw it," which matters when reconstructing an
incident timeline from logs that arrived out of order.
