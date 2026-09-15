# grpc-nv

gRPC is a remote procedure call protocol in which one call is one HTTP/2
stream. It is specified in
[gRPC over HTTP/2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md).
This package is the client and the server: the party that opens a stream,
sends what the protocol says to send, and reads what arrives. The protocol
arithmetic underneath it is
[grpc-codec-nv](https://novo-lang.org/packages/grpc-codec-nv)'s, the
stream it runs on is
[http2-nv](https://novo-lang.org/packages/http2-nv)'s, and the messages
that travel on it are
[protobuf-nv](https://novo-lang.org/packages/protobuf-nv)'s.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the host half of gRPC is

grpc-codec-nv holds the whole protocol and performs none of it. Every
call into it answers the events a caller has learned and a list of
**actions**: send these headers, send these bytes, send these trailers,
end the stream, reset the stream. This package is the party that carries
out that list, and a host that carries it out in order has implemented
gRPC.

What it carries the list out over is a **transport**: one HTTP/2
connection, expressed here as the trait `grpctrans.GrpcTransport`. The
trait has seven methods. Six are grpc-codec-nv's own list of what a host
must supply, and the seventh asks whether the connection is still usable.
HPACK does not appear in any of them, because the codec hands over
header name-value pairs and takes the same back. No package implements
the trait today. http2-nv is the HTTP/2 connection an implementation
will be written over, and it is an interface release itself.

A **channel** is one endpoint and the policy every call on it uses: the
size limits, the metadata every request carries, the default deadline,
the compression offered, and the retry rules. A channel holds no socket,
so it outlives the connection under it. The transport is an argument to
every function that performs, never a field.

There are four call shapes, and they are two flags on a method
descriptor. The protocol document defines them under *Requests*.

| Shape | Client sends | Server sends |
| --- | --- | --- |
| unary | one message | one message |
| server streaming | one message | any number |
| client streaming | any number | one message |
| bidirectional | any number | any number |

A **deadline** has two spellings. On the wire it is `grpc-timeout`, a
duration counted from receipt, so the two ends need no agreement about
the clock. Inside a process it is an absolute nanosecond reading, because
a machine's own clock is consistent with itself. `grpcchan.now_nanos` is
the one function here that reads a clock. Every other function takes the
reading as an argument.

A **retry** is a second execution of the same request. The policy this
package applies is gRPC's own, from its service config, with the field
names gRPC uses: `max_attempts`, `initial_backoff_ms`, `max_backoff_ms`,
a multiplier as a fraction, and the list of statuses worth retrying. A
call becomes **committed** when a response message has been delivered to
the caller, and a committed call is never retried.

Five constants name the HTTP/2 numbers a gRPC call uses. The codes are
RFC 9113 section 7's, and `h2` is the protocol identifier a client must
offer in ALPN to get an HTTP/2 connection.

| Constant | Value |
| --- | --- |
| `H2_NO_ERROR` | `0x0` |
| `H2_INTERNAL_ERROR` | `0x2` |
| `H2_REFUSED_STREAM` | `0x7` |
| `H2_CANCEL` | `0x8` |
| `ALPN_H2` | `"h2"` |

Two functions declare a concrete effect and the rest do not.

| Function | Declared effects |
| --- | --- |
| `grpcchan.now_nanos` | `[time]` |
| `grpcchan.dial_tls` | `[net]` |
| `grpcinvoke.pump`, `.send`, `.finish_sending`, `.cancel`, `.unary`; `grpcserve.pump`, `.send_head`, `.send`, `.finish` | whatever the transport declares |
| everything else | none |

## Install

```
novo pkg add grpc-nv
```

## Example

```novo
use grpcchan
use grpctrans
use grpcstatus

fn main() [io, time]
    // A channel is one endpoint and its policy. Nothing is opened
    // here: the transport is an argument to every call that performs.
    let base = grpcchan.channel(grpcchan.endpoint("https", "api.example.com", 443))

    // Five seconds is the budget a call gets when it names none, and
    // this channel retries the statuses gRPC's own policy retries.
    let c = grpcchan.with_retries(grpcchan.with_deadline(base, 5000000000),
                                  grpcchan.default_retries())

    // How long to wait before the first retry. The jitter factor is
    // an argument, given in thousandths, so the number is testable.
    println("${grpcchan.backoff_ms(c.options.retry, 1, 1000)} ms")

    // One clock reading starts the call, and every later decision is
    // arithmetic over it.
    let started = grpcchan.now_nanos()
    let at = grpcchan.deadline_at(started, c.options.default_deadline_nanos)
    println("${grpcchan.deadline_passed(at, started)}")   // false

    // What a proxy forwards: what is left of the deadline, not what
    // arrived. Five seconds arrived and one has gone.
    println("${grpcchan.forwarded_deadline(5000000000, 1000000000)}")

    // A transport fault is not a status. This is the status to report
    // when one must be reported, and whether the request can be sent
    // again: the stream was above the GOAWAY's last one, so it was
    // never processed.
    let f: GrpcTransportFault = GrpcWireGoAway(0, 5)
    println(grpcstatus.status_name(grpctrans.status_of_fault(f)))
    println("${grpctrans.never_processed(f, 9)}")   // true
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: grpc-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `grpctrans` | The HTTP/2 transport as a trait, what arrives on a stream, why a transport call failed, and the HTTP/2 error codes a gRPC call uses. |
| `grpcchan` | A channel: an endpoint, the policy every call on it uses, the retry policy and its backoff, the clock reading, and the deadline arithmetic including the subtraction a proxy owes. |
| `grpcinvoke` | The client. One invocation for all four call shapes, the turn that performs the codec's actions and reads what arrived, a flow-controlled send that spans turns, and the retry decision. |
| `grpcserve` | The server. A routing table over descriptors, the trailers-only answer for a path it does not serve, one exchange as a value the server's own loop drives, and the flag a health check reads. |
| `grpcstub` | The contract a generated client is written against, so a generated file calls one function here and never touches a call, an action or a frame. |
| `grpcfault` | `GrpcCallError`, which is every way a call could not be conducted, and never a status a server returned. |

## How to choose an entry point

**`grpcinvoke.unary` is the whole of a unary call.** It sends one
message, half-closes, pumps until the trailers arrive, and answers the
response and the status. Most calls are this one.

**`grpcinvoke.pump` is the turn underneath it.** Use it for a streaming
call, and for any program that does something of its own between turns.
`send`, `finish_sending` and `cancel` are the verbs beside it, and each
answers the events of that turn and how long the caller may wait before
the next.

**`grpcstub.invoke` is what generated code calls.** A generated client
builds a `GrpcMethodStub` once from a descriptor and then calls this,
which is why a change to the calling convention is not a change to every
generated file.

**`grpcserve.exchange` is the server's entry point.** It turns a stream's
request headers into a value, and `pump`, `send_head`, `send` and
`finish` drive it. The handler is not a function this package calls: the
exchange is a value the server's own loop owns.

## The rules a user needs

1. **A transport fault is not a status.** `NOT_FOUND` is a call that
   worked, and it arrives as a `GrpcTrailers` inside an `Ok`.
   `GrpcCallError` is only the cases where the conversation itself
   failed. A caller that folded the two together retries a business
   refusal.
2. **Three things must agree before a retry, and `next_attempt` is where
   they do.** The channel's policy must retry this status, the attempt
   budget must have room, and the call must not be committed.
   Commitment is gRPC's own term for a call that has delivered a
   response message, from its retry design, proposal A6. Retrying a
   committed call delivers a stream twice.
3. **For a method that is not idempotent there is a fourth question.**
   `grpctrans.never_processed` answers it, and it says yes for exactly
   two cases: a `REFUSED_STREAM`, and a stream whose identifier is above
   the `last_stream` in a GOAWAY (RFC 9113 section 6.8). A reset
   mid-call, a socket that went and a deadline that passed may all have
   been processed.
4. **Retries are off by default.** `grpcchan.default_options` uses
   `no_retries`, which is one attempt. `default_retries` is gRPC's
   suggested policy: five attempts, 100 ms doubling to one second,
   retrying `UNAVAILABLE` and nothing else.
5. **The backoff jitter is an argument, not a draw.** `backoff_ms` takes
   a factor between 800 and 1200, in thousandths. gRPC's algorithm
   multiplies the backoff by a random factor between 0.8 and 1.2, and
   the random number is the caller's to supply.
6. **A deadline is an instant in the process and a duration on the
   wire.** Read the clock once with `now_nanos`, turn it into an instant
   with `deadline_at`, and pass that reading to every function that
   needs it.
7. **A proxy forwards what is left.** `grpc-timeout` is a duration from
   receipt, so a chain of five proxies that forward it unchanged
   multiplies the budget by five. `grpcchan.forwarded_deadline` is the
   subtraction, and `grpcserve.deadline_of` is what a handler that is
   itself a client reads first.
8. **A client must offer `h2` in ALPN.** A client that does not gets an
   HTTP/1.1 connection from every conforming server, and fails on the
   first trailers. `grpctrans.ALPN_H2` is the identifier.
9. **`dial_tls` answers a handle, not a transport.** It opens a TLS
   session with `h2` offered. Speaking HTTP/2 over that session is the
   transport's work, and no package here does it yet.
10. **A path this server does not serve is answered with one header
    set.** Section *Trailers-Only*. `grpcserve.unimplemented_response`
    builds it: `grpc-status: 12`, and `end_stream`. A server that sent a
    normal response head first leaves a conforming client waiting for a
    body that is never sent.
11. **Every response ends with trailers, including the successful
    ones.** Section *Responses*. `grpcserve.finish` is that ending. A
    server that sent a body and stopped has sent a call that never
    finishes.
12. **A client-streaming call must half-close.** `finish_sending` is
    what says there are no more request messages. A call that never
    half-closes is a call the server waits on forever.
13. **A send can span turns.** `grpc_send_data` answers how many bytes
    the peer's flow-control window took, and the rest of that message is
    sent on a later turn. A `GrpcWindowClosed` event with a non-zero
    `wait_ms` is the peer pushing back.
14. **A message is `Bytes`, already serialised.** This package does not
    know what a message means. The generated encoder and decoder are
    named on the stub, and `grpcstub.types_agree` checks that they name
    the types the descriptor says they should.
15. **The server's deadline ceiling replaces a longer one.**
    `GrpcServerOptions.max_deadline_nanos` is the longest deadline this
    server honours. Zero honours whatever arrives.

## What is not included

- **HTTP/2.** The transport is the trait `grpctrans.GrpcTransport`, and
  no package implements it yet.
  [http2-nv](https://novo-lang.org/packages/http2-nv) publishes the
  connection an implementation will be written over. Until one exists,
  this package builds and type-checks and connects to nothing.
- **A loop, a thread or a callback.** A handler is not a function this
  package calls. `GrpcExchange` and `GrpcInvocation` are values the
  calling program's own loop drives.
- **Compression.** The frame's flag says that a message is compressed
  and `grpc-encoding` says with what. Inflating the bytes is the calling
  program's work.
- **The health and reflection services.** Both are named —
  `grpcserve.health_service_name` and
  `grpcstub.reflection_service_name` — and neither is implemented. A
  server with `route` and `is_serving` has what it needs. What a health
  service adds is a decision about what "healthy" means, and what a
  reflection service adds is a decision about which descriptors to
  expose.
- **Load balancing and name resolution.** A channel is one endpoint. A
  pool over several is the calling program's.
- **A service config parser.** `GrpcRetryPolicy` is a value with gRPC's
  own field names. Turning JSON into one is the calling program's work.
- **A microcontroller claim.** gRPC needs HPACK's dynamic table and
  per-stream flow control, which a device with no heap allocator cannot
  afford.

## Related packages

- [grpc-codec-nv](https://novo-lang.org/packages/grpc-codec-nv) is the
  core half of the same protocol, and the other end of this one. It
  holds the framing, the status codes, the metadata rules, the deadline
  grammar and the call state machine, and it performs nothing. This
  package depends on it and does no protocol arithmetic of its own.
- [http2-nv](https://novo-lang.org/packages/http2-nv) is the transport
  that carries a gRPC call: the frame types, HPACK, the stream state
  machine and flow control, with no socket of its own. It is what
  `grpctrans.GrpcTransport` will be implemented over.
- [protobuf-nv](https://novo-lang.org/packages/protobuf-nv) supplies the
  descriptors a service is described by and the generator whose output
  `grpcstub` defines the contract for.
- [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) is
  HTTP/1.1, and so is `std.http` in the standard library. Neither can
  carry gRPC: a request may not carry trailers, the two directions are
  not independent, and there is no per-stream flow control.
- [websocket-nv](https://novo-lang.org/packages/websocket-nv) and
  [mqtt-nv](https://novo-lang.org/packages/mqtt-nv) are the other host
  halves shaped this way, each over its own codec package, for a program
  that wants a message stream rather than a remote procedure call.
- `std.tls` in the standard library is what `grpcchan.dial_tls` opens a
  session with, and `std.time` is what `grpcchan.now_nanos` reads.

## Tests

```bash
novo test tests/grpcinvoke_tests.nv     # 20 tests: the channel, the call, the retry
novo test tests/grpcserve_tests.nv      # 15 tests: the routing, the stub, the faults
```

The values the suite asserts against are gRPC's own: the request header
set and the `Timeout` grammar from `PROTOCOL-HTTP2.md`, the status codes
from `grpc/status.proto`, the retry policy's defaults and its backoff
curve from gRPC's retry design, and the HTTP/2 error codes from RFC 9113
section 7. The suite checks that a channel holds no connection, that the
metadata order is the channel's then the call's then the timeout, that a
proxy forwards what is left of a deadline, that a committed call is never
retried, that exactly two faults make a non-idempotent retry safe, and
that an unroutable path is answered with one header set.

Each suite carries a transport over values in memory, which performs
nothing. It is the only implementation of `GrpcTransport` that exists
anywhere today, so a green suite here would not mean a call has reached a
server. The compiler makes the first assertion on its own: that transport
declares no effects, and the call verbs driven over it declare none
either, so the effect parameter is checked before the run starts.

The tests compile today and fail at run, each on the `not implemented:
grpc-nv.<module>.<fn>` panic that is its body. That is the expected state
of an interface release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `grpctrans.H2_NO_ERROR`, `.H2_CANCEL`, `.H2_INTERNAL_ERROR`, `.H2_REFUSED_STREAM`, `.ALPN_H2` | yes (they are constants) |
| `grpctrans.status_of_fault`, `.never_processed`, the `message` impl | no |
| `grpcchan.no_retries`, `.default_retries`, `.retry_policy`, `.backoff_ms`, `.retries_status` | no |
| `grpcchan.default_options`, `.endpoint`, `.channel` | no |
| `grpcchan.with_options`, `.with_metadata`, `.with_deadline`, `.with_retries` | no |
| `grpcchan.now_nanos`, `.deadline_at`, `.deadline_passed`, `.forwarded_deadline` | no |
| `grpcchan.request_metadata`, `.dial_tls`, `.tls_for` | no |
| `grpcinvoke.invocation`, `.pump`, `.send`, `.finish_sending`, `.cancel`, `.unary` | no |
| `grpcinvoke.state_of`, `.trailers_of`, `.wait_ms`, `.expired` | no |
| `grpcinvoke.next_attempt`, `.is_committed` | no |
| `grpcserve.default_options`, `.server`, `.route`, `.unimplemented_response` | no |
| `grpcserve.set_serving`, `.is_serving`, `.health_service_name` | no |
| `grpcserve.exchange`, `.pump`, `.send_head`, `.send`, `.finish` | no |
| `grpcserve.deadline_of`, `.abandoned`, `.request_metadata` | no |
| `grpcstub.method_stub`, `.service_stub`, `.path_of`, `.streaming_of` | no |
| `grpcstub.invoke`, `.types_agree` | no |
| `grpcstub.module_name_for`, `.client_fn_name`, `.server_fn_name`, `.reflection_service_name` | no |
| `grpcfault.status_of`, `.safe_to_replay`, `.reset_code_for`, the `message` impl | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
