# json-rpc API

```milo
// envelopes in
parseMessage(body)              // RpcMessage.Request | Notification | Response | ErrorResponse | Invalid
RpcRequest                      // .method, .id, .params (Json), .hasParams
RpcNotification                 // .method, .params, .hasParams
RpcResponse                     // .id, .result
RpcErrorResponse                // .id, .code, .message, .data

// envelopes out — every *Json argument is spliced in already serialized
buildRequest(id, method, paramsJson)
buildNotification(method, paramsJson)
buildResponse(id, resultJson)
buildErrorResponse(id, err)

// ids
RpcId.IdNum(n)                  RpcId.IdStr(s)
rpcIdEq(a, b)                   id.clone()

// errors
rpcError(code, message)
rpcErrorWithData(code, message, dataJson)
PARSE_ERROR      -32700         INVALID_REQUEST  -32600
METHOD_NOT_FOUND -32601         INVALID_PARAMS   -32602
INTERNAL_ERROR   -32603

// framing — "json-rpc/frame"
readFrame(fd)                   // Result<string, FrameError>
readFrameWith(reader)           // same, over an FdReader you own
writeFrame(fd, payload)         // Result<i64, FrameError>
frameErrorMessage(e)

// correlation — "json-rpc/pending"
PendingRequests<T>.new()
p.nextId()                      p.register(id, val)
p.take(&id)                     // Option<T>, removes the entry
p.len()
```

### Building results and errors

```milo
from "json-rpc" import {
    RpcId, buildResponse, buildErrorResponse, rpcError, rpcErrorWithData, INVALID_PARAMS
}
from "std/json" import { Json }

fn main(): void {
    print(buildResponse(RpcId.IdNum(1), Json.obj().int("sum", 5).build()))

    print(buildErrorResponse(RpcId.IdNum(2), rpcError(INVALID_PARAMS, "expected an integer")))

    let data = Json.obj().str("field", "count").build()
    print(buildErrorResponse(RpcId.IdNum(3),
        rpcErrorWithData(INVALID_PARAMS, "expected an integer", data)))
}
```

```
{"jsonrpc":"2.0","id":1,"result":{"sum":5}}
{"jsonrpc":"2.0","id":2,"error":{"code":-32602,"message":"expected an integer"}}
{"jsonrpc":"2.0","id":3,"error":{"code":-32602,"message":"expected an integer","data":{"field":"count"}}}
```

> **Sharp edge:** `params`, `result`, and `error.data` are spliced in **already
> serialized**, the same convention as `std/json`'s `JsonObj.raw`. A bare Milo
> string handed to `buildResponse` lands on the wire unquoted, which is not
> valid JSON there. Go through `Json.obj()`/`Json.arr()`, or `jsonQuote(s)` for
> a lone string.

## Reading params

`RpcRequest.params` is a `std/json` `Json` value — use its accessors directly.
Check `hasParams` first: a request with no `params` field still gives you a
`params` of JSON `null`, which is a legitimate `Json` value rather than a signal
that the field was missing.

```milo
fn handleMove(req: &RpcRequest): void {
    if !req.hasParams {
        return
    }
    let name = req.params.str("name") ?? "anon"       // one level deep
    let count = req.params.i64("count") ?? 0
    let zip = req.params.strPath("address.zip") ?? "" // dotted path into a subtree

    print(name, count, zip)
}
```

## Frame errors

`readFrame`/`writeFrame` return `Result<_, FrameError>`, one variant per failure
so a caller can say *why*, not just *that*.

| Variant | Wire condition | What to do |
|---|---|---|
| `Eof` | Stream closed before a header or body completed. | On a fresh read this is the normal end of a session — stop the loop, don't log it. |
| `MissingContentLength` | Complete header block with no `Content-Length`. | Terminal. Log and close; don't scan ahead to resync. |
| `MalformedLength` | `Content-Length` present but not a non-negative integer. | Terminal, same as above. |
| `TruncatedBody` | Stream ended after promising `N` bytes. Partial bytes are discarded, not handed back as a short body. | Terminal — the peer crashed or the connection dropped mid-message. |
| `ShortWrite` | `write()` took fewer bytes than the frame. Send side only. | Terminal for that connection: half a frame is already on the wire, so there's no safe resume. Reconnect if the transport allows. |

Only `Eof` means "the peer is done". Every other variant means framing broke and
the stream can't be trusted from that point.

## DAP

[DAP](https://microsoft.github.io/debug-adapter-protocol/) shares the
Content-Length framing byte-for-byte, but its envelope predates JSON-RPC 2.0 —
DAP carries `seq`/`type`/`command`/`arguments` and `request_seq`/`success`, not
`jsonrpc`/`method`/`id`/`params`. So a DAP consumer imports the framing only:

```milo
from "json-rpc/frame" import { readFrame, writeFrame }
```

and builds its own envelope on top. `PendingRequests` still works for DAP's
reverse requests — DAP correlates by echoing a numeric `seq` in `request_seq`,
so wrapping your seq as `RpcId.IdNum(seq)` gets you the same table.

## Tests

```bash
milo test tests/
```
