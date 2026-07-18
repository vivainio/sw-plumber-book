# The OTLP Wire Protocol

Every SDK component described elsewhere in this section — tracer,
processor, sampler — exists to produce one thing: a serialized **OTLP**
(OpenTelemetry Protocol) message. OTLP is not a REST API with a JSON
schema that evolved organically; it's defined as a fixed set of
`.proto` files in the `opentelemetry-proto` repository, and every SDK,
Collector, and backend in the ecosystem compiles code from the *same*
`.proto` sources. That's why a Java SDK, a Go Collector, and a Rust
backend can all understand each other's export requests byte-for-byte
without ever agreeing on anything at the source-code level.

## The message hierarchy

The proto definitions are split across a few packages that mirror each
other for traces, metrics, and logs. `common.proto` defines the types
every signal reuses:

```protobuf
message AnyValue {
  oneof value {
    string string_value = 1;
    bool bool_value = 2;
    int64 int_value = 3;
    double double_value = 4;
    ArrayValue array_value = 5;
    KeyValueList kvlist_value = 6;
    bytes bytes_value = 7;
  }
}

message KeyValue {
  string key = 1;
  AnyValue value = 2;
}

message InstrumentationScope {
  string name = 1;
  string version = 2;
  repeated KeyValue attributes = 3;
  uint32 dropped_attributes_count = 4;
}
```

`AnyValue` being a `oneof` rather than a plain `string` is deliberate: it's
the same reason `http.status_code` shows up as a real integer instead of
the string `"200"` in a backend's query engine — the type survives the
wire, so a backend can do `WHERE http.status_code >= 500` without parsing
strings. Every attribute anywhere in OTel — span attributes, resource
attributes, metric data point attributes, log attributes — is a
`repeated KeyValue`, which is why "attributes" behave identically across
all three signals: it's the literal same message type.

`resource.proto` defines the Resource wrapper introduced in the section
overview:

```protobuf
message Resource {
  repeated KeyValue attributes = 1;
  uint32 dropped_attributes_count = 2;
}
```

And `trace.proto` defines the actual span, with `Event` and `Link` as
nested message types:

```protobuf
message Span {
  bytes trace_id = 1;              // 16 bytes, raw — not hex-encoded here
  bytes span_id = 2;               // 8 bytes, raw
  string trace_state = 3;
  bytes parent_span_id = 4;
  string name = 5;
  SpanKind kind = 6;
  fixed64 start_time_unix_nano = 7;
  fixed64 end_time_unix_nano = 8;
  repeated KeyValue attributes = 9;
  uint32 dropped_attributes_count = 10;
  repeated Event events = 11;
  uint32 dropped_events_count = 12;
  repeated Link links = 13;
  uint32 dropped_links_count = 14;
  Status status = 15;

  message Event {
    fixed64 time_unix_nano = 1;
    string name = 2;
    repeated KeyValue attributes = 3;
    uint32 dropped_attributes_count = 4;
  }

  message Link {
    bytes trace_id = 1;
    bytes span_id = 2;
    string trace_state = 3;
    repeated KeyValue attributes = 4;
    uint32 dropped_attributes_count = 5;
  }
}

message Status {
  string message = 2;
  StatusCode code = 3;   // STATUS_CODE_UNSET | STATUS_CODE_OK | STATUS_CODE_ERROR
}
```

Notice the `dropped_*_count` fields scattered throughout: every
repeated collection an SDK might truncate (an attribute limit, an event
limit) carries its own counter, so a backend can tell "this span really
only had 3 events" apart from "this span had 400 events and the SDK
dropped 397 of them to stay under its cap" — a real distinction when
debugging why a suspiciously round number of attributes shows up.

The top-level message that actually gets sent over the network nests
`Resource` and `InstrumentationScope` *once*, around potentially many
spans, rather than repeating them per span:

```protobuf
message ExportTraceServiceRequest {
  repeated ResourceSpans resource_spans = 1;
}

message ResourceSpans {
  Resource resource = 1;
  repeated ScopeSpans scope_spans = 2;
  string schema_url = 3;
}

message ScopeSpans {
  InstrumentationScope scope = 1;
  repeated Span spans = 2;
  string schema_url = 3;
}
```

So one export request from a single process typically contains **one**
`ResourceSpans` (that process has one Resource), a handful of
`ScopeSpans` (one per instrumentation library that produced spans:
`spring-webmvc`, `jdbc`, your own manual `Tracer`), each holding however
many spans that scope produced since the last flush. This nesting is
purely a wire-efficiency choice — it's what keeps a batch of 500 spans
from a single process from repeating `service.name=checkout-api` 500
times.

Metrics and logs mirror the exact same shape —
`ExportMetricsServiceRequest` → `ResourceMetrics` → `ScopeMetrics` →
metric data points, and `ExportLogsServiceRequest` → `ResourceLogs` →
`ScopeLogs` → `LogRecord` — which is why, once you've read one, the other
two are the same tree with a different leaf.

## The RPC contract

The service definition that turns this message into an actual network
call is small on purpose:

```protobuf
service TraceService {
  rpc Export(ExportTraceServiceRequest) returns (ExportTraceServiceResponse);
}

message ExportTraceServiceResponse {
  ExportTracePartialSuccess partial_success = 1;
}
```

One RPC, one request type, one response type, and the response can carry
a `partial_success` — the receiving end (usually a Collector) telling the
sender "I accepted 480 of 500 spans, here's why the other 20 were
rejected" without failing the whole batch over a handful of malformed
records.

## Three transports, one payload

That same `ExportTraceServiceRequest` reaches the wire in one of three
ways, and recognizing them on sight is the fastest way to debug a
connectivity problem:

**gRPC** — the default for most SDKs, port `4317`. Binary protobuf,
carried as HTTP/2 DATA frames, called against the path
`/opentelemetry.proto.collector.trace.v1.TraceService/Export`. gRPC adds
its own 5-byte frame header in front of each serialized message (a
1-byte compression flag plus a 4-byte big-endian length), so what's
actually on the wire is `[flag][length][serialized ExportTraceServiceRequest]`
per HTTP/2 DATA frame — this is why gRPC traffic can't be replayed with
plain `curl`, and why a proxy that doesn't understand HTTP/2 trailers
(gRPC's status code rides in an HTTP/2 trailer, sent *after* the body)
will report "success" on calls that gRPC itself considers failed.

**HTTP/protobuf** — port `4318`, a plain `POST /v1/traces` with
`Content-Type: application/x-protobuf` and the *same* serialized bytes as
the gRPC payload, minus the gRPC framing — just a normal HTTP/1.1 or
HTTP/2 request body. This has become the more common default in newer
SDKs precisely because it survives load balancers, proxies, and
corporate firewalls that HTTP/2-only gRPC sometimes doesn't.

**HTTP/JSON** — same endpoint, `Content-Type: application/json`, mostly
useful for reading a payload by eye or scripting against it with `curl`.
proto3's canonical JSON mapping applies, and it has consequences worth
knowing before you go looking for a `traceId` field that "should" be
hex: `bytes` fields are **base64-encoded**, and 64-bit integers
(`fixed64`, `int64`) are encoded as **JSON strings**, not numbers — a
deliberate hedge against JavaScript's `Number` losing precision above
2^53. A raw export request over HTTP/JSON looks like this:

```json
{
  "resourceSpans": [{
    "resource": {
      "attributes": [
        { "key": "service.name", "value": { "stringValue": "checkout-api" } }
      ]
    },
    "scopeSpans": [{
      "scope": { "name": "io.opentelemetry.instrumentation.spring-webmvc" },
      "spans": [{
        "traceId": "S/kvNXezTaajzpKdDg5HNg==",
        "spanId": "APBnqguqArc=",
        "name": "GET /orders/{id}",
        "kind": "SPAN_KIND_SERVER",
        "startTimeUnixNano": "1700000000000000000",
        "endTimeUnixNano": "1700000000042000000",
        "attributes": [
          { "key": "http.status_code", "value": { "intValue": "200" } }
        ],
        "status": { "code": "STATUS_CODE_OK" }
      }]
    }]
  }]
}
```

Decode that `traceId` from base64 and you get back
`4bf92f3577b34da6a3ce929d0e0e4736` — the exact same 16 raw bytes as the
`traceparent` header from
[Traces and Context Propagation](traces-and-context.md), just in a
different text encoding. The propagation header and the exported span
aren't independently generated; they're two textual views of the same
binary trace ID, which is precisely what makes it possible to paste a
`traceparent` value from a curl `-v` log straight into a backend's search
box and land on the right trace.

You can watch this happen without writing any code: point an SDK's OTLP
exporter at `http://localhost:4318/v1/traces` and run a plain HTTP proxy
or `mitmproxy` in front of it — with the JSON variant selected, the
export requests are fully human-readable, which makes it a reasonable
first debugging step before reaching for `grpcurl` against the gRPC
endpoint.
