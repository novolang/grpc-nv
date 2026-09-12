# Changelog

All notable changes to grpc-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `grpctrans` — `GrpcTransport[e]`, the six methods grpc-codec-nv's
  README says a host must provide, with `GrpcWire` for what arrives and
  `GrpcTransportFault` for what goes wrong under the protocol rather
  than in it.
- `grpcchan` — a channel as policy that outlives its connection, the
  retry policy with gRPC's own field names, and the four deadline
  functions including the subtraction a proxy owes.
- `grpcinvoke` — one `GrpcInvocation` for all four call shapes, the
  loop that walks grpc-codec-nv's `GrpcAction` list, flow control as a
  partial write that spans turns, and `next_attempt`, where the three
  conditions for a safe retry are made to agree.
- `grpcserve` — routing as a lookup, `unimplemented_response` as a
  trailers-only answer, an exchange as a value the server's own loop
  drives, and the serving flag a health check reads.
- `grpcstub` — the contract protobuf-nv's generator emits against, so a
  generated file never touches `GrpcCall`, `GrpcAction` or a frame.
- `grpcfault` — `GrpcCallError`, which is never a status.

### Known

- **There is no HTTP/2 in this project**, so this package connects to
  nothing.  `GrpcTransport[e]` is the interface the missing `http2-nv`
  would implement, and publishing the design against a trait is what
  lets it be reviewed before somebody spends a month on HPACK.
- **The plan's row says "over grpc-codec-nv and std.http" and that half
  is wrong.**  HTTP/1.1 cannot carry gRPC: trailers after a body, two
  independent directions, and per-stream flow control are all required
  and 1.1 has none of the three.  Neither `std.http` nor http-codec-nv
  is a dependency.
- **A status is not a transport fault.**  `GrpcNotFound` arrives as
  `GrpcTrailers` inside an `Ok`; `GrpcCallError` is only the cases
  where the conversation failed.
- **A retry needs three agreements and sometimes four**: the policy
  retries the status, the budget has room, the call is not committed —
  and for a non-idempotent method, that `never_processed` says the
  request never reached the server, which is true for a REFUSED_STREAM
  and a stream above a GOAWAY's `last_stream` and nothing else.
- **A deadline is a duration on the wire and an instant in the
  process.**  `forwarded_deadline` is the subtraction a proxy owes, and
  without it a chain of five proxies multiplies the budget by five.
- **Only two functions name a concrete effect** — `now_nanos` is
  `[time]` and `dial_tls` is `[net]` — because a package whose
  transport does not exist cannot name that transport's effects.
- **The health and reflection services are named, not implemented.**  A
  server with `route` and `is_serving` has what it needs; what this
  package would add is a decision that belongs to a deployment.
- **No device claim.**  gRPC needs HPACK's dynamic table and per-stream
  flow control, which a device that could afford would not need gRPC.
- **Two `core` dependencies**, grpc-codec-nv and protobuf-nv, both
  interfaces themselves today.
