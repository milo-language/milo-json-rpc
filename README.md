# json-rpc

JSON-RPC 2.0 for [Milo](https://github.com/milo-language/milo): Content-Length base-protocol
framing over any fd, request/response/notification envelopes, id correlation, and the spec's
standard error codes. No dependencies beyond the standard library.

## Install

```bash
milo add github.com/milo-language/milo-json-rpc
```

```milo
from "json-rpc" import { parseMessage, buildResponse }
from "json-rpc/frame" import { readFrame, writeFrame }
from "json-rpc/pending" import { PendingRequests }
```

## A complete echo server

Reads framed requests off stdin, dispatches one method, writes framed responses to stdout —
the same shape as an LSP or DAP server's request loop, minus the protocol-specific method
table. Copy into `main.milo` and run `milo build main.milo -o echo_server`:

```milo
from "json-rpc" import {
    RpcMessage, RpcRequest, parseMessage, buildResponse, buildErrorResponse,
    rpcError, METHOD_NOT_FOUND, INVALID_PARAMS
}
from "json-rpc/frame" import { readFrame, writeFrame, FrameError, frameErrorMessage }
from "std/json" import { Json }

fn dispatch(req: &RpcRequest): string {
    if req.method == "echo" {
        if !req.hasParams {
            return buildErrorResponse(req.id.clone(),
                rpcError(INVALID_PARAMS, "echo needs {\"message\": <string>}"))
        }
        let msg = req.params.str("message") ?? ""
        return buildResponse(req.id.clone(), Json.obj().str("echo", msg).build())
    }
    return buildErrorResponse(req.id.clone(),
        rpcError(METHOD_NOT_FOUND, $"unknown method: {req.method}"))
}

fn main(): void {
    while true {
        match readFrame(0) {
            Result.Ok(body) => {
                match parseMessage(body) {
                    RpcMessage.Request(r) => {
                        match writeFrame(1, dispatch(r)) {
                            Result.Ok(_n) => {
                            }
                            Result.Err(e) => {
                                eprint("write failed: ", frameErrorMessage(e))
                                return
                            }
                        }
                    }
                    // A notification has no id to reply to, and this server sends no
                    // requests of its own, so nothing else needs an answer.
                    _ => {
                    }
                }
            }
            Result.Err(e) => {
                match e {
                    // The peer closing stdin is how an editor ends a session, not an error.
                    FrameError.Eof => {
                        return
                    }
                    _ => {
                        eprint("frame error: ", frameErrorMessage(e))
                        return
                    }
                }
            }
        }
    }
}
```

Pipe a hand-framed request at it:

```
$ printf 'Content-Length: 75\r\n\r\n{"jsonrpc":"2.0","id":1,"method":"echo","params":{"message":"hello world"}}' | ./echo_server
Content-Length: 56

{"jsonrpc":"2.0","id":1,"result":{"echo":"hello world"}}
```

An unknown method gets `METHOD_NOT_FOUND` rather than a crash or a silent drop:

```
$ printf 'Content-Length: 58\r\n\r\n{"jsonrpc":"2.0","id":2,"method":"frobnicate","params":{}}' | ./echo_server
Content-Length: 87

{"jsonrpc":"2.0","id":2,"error":{"code":-32601,"message":"unknown method: frobnicate"}}
```

## Reading params

`RpcRequest.params` is a `std/json` `Json` value — use its accessors directly. Check
`hasParams` first: a request with no `params` field still gives you a `params` of JSON `null`,
which is a legitimate `Json` value rather than a signal that the field was missing.

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

## Building results and errors

```milo
from "json-rpc" import {
    RpcId, buildResponse, buildErrorResponse, rpcError, rpcErrorWithData, INVALID_PARAMS
}
from "std/json" import { Json }

fn main(): void {
    print(buildResponse(RpcId.IdNum(1), Json.obj().int("sum", 5).build()))
    // {"jsonrpc":"2.0","id":1,"result":{"sum":5}}

    print(buildErrorResponse(RpcId.IdNum(2), rpcError(INVALID_PARAMS, "expected an integer")))
    // {"jsonrpc":"2.0","id":2,"error":{"code":-32602,"message":"expected an integer"}}

    let data = Json.obj().str("field", "count").build()
    print(buildErrorResponse(RpcId.IdNum(3),
        rpcErrorWithData(INVALID_PARAMS, "expected an integer", data)))
    // {"jsonrpc":"2.0","id":3,"error":{"code":-32602,"message":"expected an integer","data":{"field":"count"}}}
}
```

> **Sharp edge:** `params`, `result`, and `error.data` are spliced in **already serialized**,
> the same convention as `std/json`'s `JsonObj.raw`. A bare Milo string handed to
> `buildResponse` lands on the wire unquoted, which is not valid JSON there. Go through
> `Json.obj()`/`Json.arr()`, or `jsonQuote(s)` for a lone string.

## Client side: correlating responses to requests

`PendingRequests<T>` is the client half: register what you're waiting for when you send a
request, look it up by id when the response arrives. `T` is yours to pick — a channel, a
callback, a bare label.

```milo
from "json-rpc" import { RpcMessage, buildRequest, parseMessage }
from "json-rpc/pending" import { PendingRequests }
from "json-rpc/frame" import { readFrame, writeFrame }

fn main(): void {
    var inFlight: PendingRequests<string> = PendingRequests<string>.new()

    // Register BEFORE writing to the wire — the reply can land first.
    let id = inFlight.nextId()
    inFlight.register(id.clone(), "add")
    match writeFrame(1, buildRequest(id, "add", "[1,2]")) {
        Result.Ok(_n) => {
        }
        Result.Err(_e) => {
            return
        }
    }

    // Later, on a response read off the wire:
    match readFrame(0) {
        Result.Ok(body) => {
            match parseMessage(body) {
                RpcMessage.Response(r) => {
                    match inFlight.take(r.id) {
                        Option.Some(method) => {
                            print("resolved: ", method)
                        }
                        Option.None => {
                            print("unknown id — duplicate reply or stale session")
                        }
                    }
                }
                _ => {
                }
            }
        }
        Result.Err(_e) => {
        }
    }
}
```

`take()` removes the entry, so a duplicate reply or a stale id from a previous session comes
back `None` instead of handing out something else's value.

## Frame errors

`readFrame`/`writeFrame` return `Result<_, FrameError>`, one variant per failure so a caller
can say *why*, not just *that*.

| Variant | Wire condition | What to do |
|---|---|---|
| `Eof` | Stream closed before a header or body completed. | On a fresh read this is the normal end of a session — stop the loop, don't log it. |
| `MissingContentLength` | Complete header block with no `Content-Length`. | Terminal. Log and close; don't scan ahead to resync. |
| `MalformedLength` | `Content-Length` present but not a non-negative integer. | Terminal, same as above. |
| `TruncatedBody` | Stream ended after promising `N` bytes. Partial bytes are discarded, not handed back as a short body. | Terminal — the peer crashed or the connection dropped mid-message. |
| `ShortWrite` | `write()` took fewer bytes than the frame. Send side only. | Terminal for that connection: half a frame is already on the wire, so there's no safe resume. Reconnect if the transport allows. |

Only `Eof` means "the peer is done". Every other variant means framing broke and the stream
can't be trusted from that point.

## DAP

[DAP](https://microsoft.github.io/debug-adapter-protocol/) shares the Content-Length framing
byte-for-byte, but its envelope predates JSON-RPC 2.0 — DAP carries
`seq`/`type`/`command`/`arguments` and `request_seq`/`success`, not
`jsonrpc`/`method`/`id`/`params`. So a DAP consumer imports the framing only:

```milo
from "json-rpc/frame" import { readFrame, writeFrame }
```

and builds its own envelope on top. `PendingRequests` still works for DAP's reverse requests —
DAP correlates by echoing a numeric `seq` in `request_seq`, so wrapping your seq as
`RpcId.IdNum(seq)` gets you the same table.

## Tests

```bash
milo test tests/
```

## License

MIT
