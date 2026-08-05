# json-rpc

JSON-RPC 2.0 for [Milo](https://github.com/milo-language/milo): Content-Length base-protocol
framing over any fd, request/response/notification envelopes, id correlation, and the spec's
standard error codes. No dependencies beyond the standard library.

Add it to your `milo.json`:

```json
{
  "deps": {
    "json-rpc": "github.com/milo-language/milo-json-rpc@v0.1.0"
  }
}
```

## Framing

```milo
from "json-rpc/frame" import { readFrame, writeFrame }

match writeFrame(fd, payload) {
    Result.Ok(_n) => {}
    Result.Err(e) => { eprint(frameErrorMessage(e)) }
}

match readFrame(fd) {
    Result.Ok(body) => { /* body is the raw JSON, exactly Content-Length bytes */ }
    Result.Err(e) => { /* Eof / MissingContentLength / MalformedLength / TruncatedBody */ }
}
```

`readFrame`/`writeFrame` work over stdio, a socket, or a pipe — nothing here is stdio-specific.
Header matching is case-insensitive, per the base protocol LSP and DAP both specify. This is
transport only: DAP consumers (its envelope predates and diverges from JSON-RPC 2.0) use this
module and nothing else from the package.

## Envelopes

```milo
from "json-rpc" import { RpcId, RpcMessage, buildRequest, parseMessage }

let json = buildRequest(RpcId.IdNum(1), "add", "[1,2]")

match parseMessage(incomingBody) {
    RpcMessage.Request(r) => { /* r.id, r.method, r.params */ }
    RpcMessage.Notification(n) => {}
    RpcMessage.Response(r) => {}
    RpcMessage.ErrorResponse(e) => {}
    RpcMessage.Malformed => { /* not valid JSON, or no method/result/error */ }
}
```

An id is a number OR a string (spec §4) — `RpcId` keeps both, never collapsing one into the
other. Standard error codes (`PARSE_ERROR`, `INVALID_REQUEST`, `METHOD_NOT_FOUND`,
`INVALID_PARAMS`, `INTERNAL_ERROR`) and `rpcError`/`rpcErrorWithData` build well-formed error
objects for `buildErrorResponse`.

## Id correlation

```milo
from "json-rpc/pending" import { PendingRequests }

var inFlight: PendingRequests<Handler> = PendingRequests<Handler>.new()
let id = inFlight.nextId()
sendRequest(id.clone(), ...)
inFlight.register(id, handler)
// later, on a response:
match inFlight.take(response.id) {
    Option.Some(handler) => { ... }
    Option.None => { /* unknown id — duplicate reply, stale session, ignore */ }
}
```

## Running the tests

```bash
milo test tests/
```
