# grpc-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

**And there is a second thing to say plainly: there is no HTTP/2
implementation in this project, so even when the bodies land this
package connects to nothing until `http2-nv` exists.**  The transport
is a trait — `grpctrans.GrpcTransport[e]` — and nothing implements it.
The section below says why that is the right shape rather than a
placeholder.

## What this is

The host half of gRPC.  grpc-codec-nv turns a call into a state machine
that says what the host must send; this package is the host that sends
it — the four call shapes as values a loop pumps, deadlines against a
real clock, metadata merged in the right order, retries that know when
a request was never processed, a channel that outlives its connection,
a server's routing, and the contract protobuf-nv's generator emits
against.

## Adding it, and checking it

```bash
novo pkg add grpc-nv        # into your novo.toml
novo pkg build              # type- and effect-check the package
novo test --isolate tests/grpcinvoke_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: grpc-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use grpcchan
use grpcinvoke
use grpcstub

// A unary call, once a transport exists.  Every deadline decision in
// it is made from one clock reading the caller took.
fn say_hello<T: GrpcTransport[e]>(t: T, set: PbFileSet, body: Bytes) -> Result<GrpcUnaryResult, GrpcCallError> [e, time]
    let c = grpcchan.with_deadline(
        grpcchan.channel(grpcchan.endpoint("https", "api.example.com", 443)),
        5000000000)
    let now = grpcchan.now_nanos()
    let s = grpcstub.method_stub(set, "helloworld.Greeter", "SayHello") ?? nothing_stub()
    let i = grpcstub.invoke(s, c, grpcmeta.empty(), 0, now)!
    grpcinvoke.unary(i, t, body, now)
```

## The missing transport, and why the trait is the deliverable

HTTP/1.1 cannot carry gRPC, and the three reasons are hard
requirements rather than preferences:

- **Trailers.**  Every gRPC response ends with `grpc-status` *after*
  the body, because a server that has streamed nine of ten responses
  and then fails has already sent `:status: 200`.  HTTP/1.1 has
  trailers only in chunked transfer encoding, proxies strip them
  routinely, and a *request* cannot carry them at all.
- **Two independent directions.**  A bidirectional call sends and
  receives at the same time for minutes.  HTTP/1.1 is request then
  response.
- **Per-stream flow control.**  A client-streaming call that outran its
  server would have nothing to push back with.

So `std.http` is not a dependency and neither is http-codec-nv — the
plan's row for this package says "over grpc-codec-nv and std.http", and
that half of the row does not survive contact with the protocol.
grpc-codec-nv's README names the row that is actually missing:
**`http2-nv`**, `core`/`networking` — the frame layer (HEADERS, DATA,
SETTINGS, WINDOW_UPDATE, RST_STREAM, GOAWAY, PING), HPACK with its
dynamic table, the stream state machine, connection and stream flow
control, and the h2c and ALPN preludes.  It is not a gRPC package: an
HTTP/2 server that never speaks gRPC needs exactly the same thing.

Publishing this package with a transport it cannot have is not a
pretence.  It is the only way to have the interface reviewed, the
effect rows checked by the compiler and the call shapes argued about
before somebody spends a month on HPACK — which is the whole premise of
the interfaces-first milestone.  `grpctrans.GrpcTransport[e]` has six
methods and they are grpc-codec-nv's own list of what a host must
provide, with nothing added.

`grpcchan.dial_tls` is the one piece the standard library *can* supply
today: a TLS session with `h2` offered in ALPN.  It answers a handle
rather than a transport, and the README says so rather than the
signature pretending otherwise.

## The layer, and why

`host`, and only two functions name a concrete effect.

| module | row | why |
| --- | --- | --- |
| `grpcchan.now_nanos` | `[time]` | the one function in the package that reads a clock |
| `grpcchan.dial_tls` | `[net]` | `std.tls`'s own row |
| `grpcinvoke.pump`, `.send`, `.finish_sending`, `.cancel`, `.unary`; `grpcserve.pump`, `.send_head`, `.send`, `.finish` | `[e]` | effect-POLYMORPHIC over the transport |
| everything else — the channel, the policy, the routing, the stub contract, the deadline arithmetic | `[]` | values |

That is unusually narrow for a `host` package, and it is a consequence
rather than a goal: a package whose transport does not exist cannot
name the transport's effects, so it names a parameter instead.

## The load-bearing interface

`GrpcTransport[e]`, and the fact that it is the whole of what this
package needs from a machine.

```novo
pub trait GrpcTransport[e]
    fn grpc_open(self) -> Result<GrpcStream, GrpcTransportFault> [e]
    fn grpc_send_headers(self, s: GrpcStream, md: GrpcMetadata, end_stream: Bool) -> Result<Unit, GrpcTransportFault> [e]
    fn grpc_send_data(self, s: GrpcStream, data: Bytes, end_stream: Bool) -> Result<Int, GrpcTransportFault> [e]
    fn grpc_send_trailers(self, s: GrpcStream, md: GrpcMetadata) -> Result<Unit, GrpcTransportFault> [e]
    fn grpc_receive(self, s: GrpcStream) -> Result<GrpcWire, GrpcTransportFault> [e]
    fn grpc_reset(self, s: GrpcStream, http2_code: Int) -> Result<Unit, GrpcTransportFault> [e]
    fn grpc_is_open(self) -> Bool
```

Six methods, and HPACK appears in none of them: the codec hands over
`GrpcMetadata` and takes it back, and how those pairs are compressed is
the transport's business entirely.  `grpc_send_data` answers **how
many** bytes the peer's window took, which is the whole of flow control
from this side and the one thing HTTP/1.1 has no way to express.

The second decision is that **a status is not a transport fault**, and
they are different types here exactly as they are in the codec.
`NOT_FOUND` is a call that *worked* — the stream opened, the frames
parsed, the trailers arrived — and it comes back as `GrpcTrailers`
inside an `Ok`.  `GrpcCallError` is only the cases where the
conversation itself failed.  A function that returned one type for both
is what makes a caller retry a business-logic refusal, which is the
most common mistake in gRPC client code.

## Deadlines, and the subtraction a proxy owes

`grpc-timeout` is a **duration from receipt**, not an instant — which
is what lets a call cross a machine whose clock is wrong.  Forwarding
one unchanged gives the next hop the whole budget again, and a chain of
five proxies multiplies the deadline by five.
`grpcchan.forwarded_deadline` is that subtraction, published so a proxy
has a function to call rather than arithmetic to get right, and
`grpcserve.deadline_of` is what a handler that is itself a client reads
before it makes its own call.

Inside the process a deadline is an absolute nanosecond reading and on
the wire it is a duration.  The two spellings exist because a machine's
own clock is consistent with itself and two machines' clocks are not.

## Retries, and the narrow definition of safe

Three things have to agree before a call is retried, and
`grpcinvoke.next_attempt` is the one place they are made to:

1. the channel's policy retries this status — `UNAVAILABLE` and by
   default nothing else, which is also `grpcstatus.is_retryable`'s
   answer;
2. the attempt budget has room;
3. the call is **not committed** — no response message has been
   delivered, because a retry after one would deliver a stream twice.

And for a non-idempotent method there is a fourth, narrower question:
was the request ever processed?  `grpctrans.never_processed` says yes
for exactly two cases — a `REFUSED_STREAM`, and a stream above the
`last_stream` in a GOAWAY — and no for everything else.  A reset
mid-call, a socket that went, a deadline that passed: all of those may
have been processed, and retrying them is how a payment gets taken
twice.

## The stub contract

protobuf-nv's `pbgen` turns a `.proto` into novo-lang source.  What it
must **not** do is invent a calling convention: if a generated client
chose its own arguments, every change here would be a change to every
generated file, and a generated file is one nobody edits.

So the convention is in `grpcstub`, and a generated stub is the
thinnest possible layer over it: a `GrpcMethodStub` built once at
module level from the descriptor, and one function per method whose
body is one call into this package.  A generated file never touches
`GrpcCall`, `GrpcAction` or a frame.

`types_agree` is there for the failure that otherwise shows up at the
worst moment: a generated file regenerated for one `.proto` and linked
against another, which appears as a decode failure on a field number
nobody recognises.

## What this does not do, on purpose

- **It does not implement HTTP/2.**  That is `http2-nv`, and it does
  not exist.
- **It does not own a loop**, a thread, a channel or a callback.  A
  handler is not a function this package calls; `GrpcExchange` is a
  value the server's own loop drives.
- **It does not compress.**  The codec's flag says *that* a message is
  compressed and `grpc-encoding` says with what; the bytes are the
  host's to inflate, with flate-nv or whatever the header named, and a
  package that chose an algorithm would be choosing for a protocol
  whose whole point is that two ends negotiate one.
- **It does not implement the health or reflection service.**  Both are
  named — `grpcserve.health_service_name`,
  `grpcstub.reflection_service_name` — and a server with `route` and
  `is_serving` has everything it needs.  What this package would add is
  a decision about what "healthy" means and which descriptors to
  expose, and both are a deployment's.
- **It does not do load balancing or name resolution.**  A channel is
  one endpoint; a pool over several is a program's.
- **It does not parse a service config.**  `GrpcRetryPolicy` is a value
  with gRPC's own field names; turning JSON into one is a caller's.
- **No device claim.**  gRPC needs HTTP/2, which needs HPACK's dynamic
  table and per-stream flow control, and a device that could afford
  those would not be using gRPC.  grpc-codec-nv's README says the same
  and names what an embedded producer should reach for instead.

## The reference implementation

`tonic` for the client and server shapes and `grpcio` for the channel
vocabulary, with gRPC's own `PROTOCOL-HTTP2.md` and `grpc/status.proto`
as the specification.

Three things change in the port.  `tonic` is built on `hyper` and
`tower`, so its transport is a concrete HTTP/2 client and its
middleware is a service stack; here the transport is a trait and there
is no middleware at all, because a `host` package that brought an async
runtime with it would be choosing one for every program that depends on
it.  Its `Request<T>` and `Response<T>` are generic over the message
type; here a message is `Bytes` and the generated stub converts, which
is what keeps `GrpcMethodStub` a value that can sit in a list.  And its
`Duration` deadlines become nanosecond integers taken as arguments,
which is what makes the deadline arithmetic — including a proxy's
subtraction — a table a test asserts.

## Status

| item | implemented |
| --- | --- |
| `grpctrans` — `GrpcTransport[e]`, `GrpcStream`, `GrpcWire`, `GrpcTransportFault` | types only |
| `grpctrans.H2_NO_ERROR`, `.H2_CANCEL`, `.H2_INTERNAL_ERROR`, `.H2_REFUSED_STREAM`, `.ALPN_H2` | yes — they are constants |
| `grpctrans.status_of_fault`, `.never_processed`, the `message` impl | no |
| `grpcchan` — `GrpcEndpoint`, `GrpcRetryPolicy`, `GrpcChannelOptions`, `GrpcChannel` | types only |
| `grpcchan.no_retries`, `.default_retries`, `.retry_policy`, `.backoff_ms`, `.retries_status` | no |
| `grpcchan.default_options`, `.endpoint`, `.channel` | no |
| `grpcchan.with_options`, `.with_metadata`, `.with_deadline`, `.with_retries` | no |
| `grpcchan.now_nanos`, `.deadline_at`, `.deadline_passed`, `.forwarded_deadline` | no |
| `grpcchan.request_metadata`, `.dial_tls`, `.tls_for` | no |
| `grpcinvoke` — `GrpcInvocation`, `GrpcInvokeEvent`, `GrpcInvokeStep`, `GrpcUnaryResult`, `GrpcRetryDecision` | types only |
| `grpcinvoke.invocation`, `.pump`, `.send`, `.finish_sending`, `.cancel`, `.unary` | no |
| `grpcinvoke.state_of`, `.trailers_of`, `.wait_ms`, `.expired` | no |
| `grpcinvoke.next_attempt`, `.is_committed` | no |
| `grpcserve` — `GrpcServiceEntry`, `GrpcServerOptions`, `GrpcServer`, `GrpcExchange`, `GrpcServeEvent`, `GrpcServeStep` | types only |
| `grpcserve.default_options`, `.server`, `.route`, `.unimplemented_response` | no |
| `grpcserve.set_serving`, `.is_serving`, `.health_service_name` | no |
| `grpcserve.exchange`, `.pump`, `.send_head`, `.send`, `.finish` | no |
| `grpcserve.deadline_of`, `.abandoned`, `.request_metadata` | no |
| `grpcstub` — `GrpcMethodStub`, `GrpcEncodeFn`, `GrpcDecodeFn`, `GrpcServiceStub` | types only |
| `grpcstub.method_stub`, `.service_stub`, `.path_of`, `.streaming_of` | no |
| `grpcstub.invoke`, `.types_agree` | no |
| `grpcstub.module_name_for`, `.client_fn_name`, `.server_fn_name`, `.reflection_service_name` | no |
| `grpcfault` — `GrpcCallError`, the `message` impl | type only |
| `grpcfault.status_of`, `.safe_to_replay`, `.reset_code_for` | no |
