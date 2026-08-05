# json-rpc

JSON-RPC 2.0 for [Milo](https://github.com/milo-language/milo): Content-Length base-protocol
framing over any fd, request/response/notification envelopes, id correlation, and the spec's
standard error codes. No dependencies beyond the standard library.

## Install

Add it to your `milo.json`:

```json
{
  "deps": {
    "json-rpc": "github.com/milo-language/milo-json-rpc@v0.1.0"
  }
}
```

## A complete echo server

Reads framed requests off stdin (fd 0), dispatches one method (`echo`), and writes framed
responses to stdout (fd 1) — the same shape as an LSP or DAP server's request loop, minus the
protocol-specific method table.

```milo
from "json-rpc" import {
    RpcId, RpcMessage, RpcRequest, parseMessage, buildResponse, buildErrorResponse,
    rpcError, METHOD_NOT_FOUND, INVALID_PARAMS
}
from "json-rpc/frame" import {
    readFrame, writeFrame, FrameError, frameErrorMessage
}
from "std/json" import {
    Json
}

fn handleEcho(req: &RpcRequest): string {
    if !req.hasParams {
        return ""
    }
    let msg = req.params.str("message") ?? ""
    return Json.obj().str("echo", msg).build()
}

fn dispatch(req: &RpcRequest): string {
    if req.method == "echo" {
        if !req.hasParams {
            let err = rpcError(INVALID_PARAMS, "echo needs {\"message\": <string>}")
            return buildErrorResponse(req.id.clone(), err)
        }
        return buildResponse(req.id.clone(), handleEcho(req))
    }
    let err = rpcError(METHOD_NOT_FOUND, $"unknown method: {req.method}")
    return buildErrorResponse(req.id.clone(), err)
}

fn main(): void {
    while true {
        match readFrame(0) {
            Result.Ok(body) => {
                match parseMessage(body) {
                    RpcMessage.Request(r) => {
                        let resp = dispatch(r)
                        match writeFrame(1, resp) {
                            Result.Ok(_n) => {
                            }
                            Result.Err(e) => {
                                eprint("write failed: ", frameErrorMessage(e))
                                return
                            }
                        }
                    }
                    RpcMessage.Notification(_n) => {
                        // No id to reply to — nothing sent back either way.
                    }
                    RpcMessage.Response(_r) => {
                        // This process never sends requests of its own in this example.
                    }
                    RpcMessage.ErrorResponse(_e) => {
                    }
                    RpcMessage.Malformed => {
                        // Body wasn't a well-formed envelope. We didn't re-check whether
                        // it was even valid JSON, so default to PARSE_ERROR; a server
                        // that cares about the INVALID_REQUEST distinction re-runs
                        // Json.parse(body) itself (see parseMessage's doc comment).
                    }
                }
            }
            Result.Err(e) => {
                match e {
                    FrameError.Eof => {
                        // Client hung up (closed stdin/its end of the pipe). Not an
                        // error — this is the normal way an editor/DAP client ends
                        // the session.
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

Verified end to end: piping a hand-framed request at this binary produces the framed response
you'd expect —

```
$ printf 'Content-Length: 75\r\n\r\n{"jsonrpc":"2.0","id":1,"method":"echo","params":{"message":"hello world"}}' | ./echo_server
Content-Length: 56

{"jsonrpc":"2.0","id":1,"result":{"echo":"hello world"}}
```

and an unrecognized method gets `METHOD_NOT_FOUND` rather than a crash or a silent drop:

```
$ printf 'Content-Length: 58\r\n\r\n{"jsonrpc":"2.0","id":2,"method":"frobnicate","params":{}}' | ./echo_server
Content-Length: 87

{"jsonrpc":"2.0","id":2,"error":{"code":-32601,"message":"unknown method: frobnicate"}}
```

## Reading params out of a request

`RpcRequest.params` is a `std/json` `Json` value — use its accessors directly. Always check
`hasParams` first: a request with no `params` field at all still gives you a `params` of JSON
`null`, which is a legitimate (if useless) `Json` value, not a signal that the field was
present-and-null.

```milo
from "json-rpc" import {
    RpcRequest
}

fn handleMove(req: &RpcRequest): void {
    if !req.hasParams {
        print("no params")
        return
    }
    let name = req.params.str("name") ?? "anon"
    let count = req.params.i64("count") ?? 0

    // .path() walks a dotted path and hands back the Json subtree at the end
    // of it — use it (or the strPath/i64Path/... shorthands) for a nested
    // field instead of chaining .get() calls.
    match req.params.path("address.zip") {
        Option.Some(z) => {
            print(name, count, z.asStr() ?? "")
        }
        Option.None => {
            print(name, count, "no zip")
        }
    }

    // .get() reads one field off the current Json value — one level deep.
    match req.params.get("tags") {
        Option.Some(_tags) => {
            print("has tags")
        }
        Option.None => {
        }
    }
}
```

## Building results and errors

Everything this package builds (`params`, `result`, `error.data`) is spliced in **already
serialized** — the same convention `std/json`'s `JsonObj.raw` uses. That is the one sharp edge
in this package's API: passing a bare Milo string where JSON is expected does not get quoted
for you. `Json.obj().str("field", value).build()` produces valid JSON; handing the same `value`
straight to `buildResponse` does not — it lands on the wire unquoted, which is not valid JSON at
that position. Always go through `Json.obj()`/`Json.arr()`, or use `jsonQuote(s)` (from
`std/json`) for a bare JSON string.

```milo
from "json-rpc" import {
    RpcId, buildResponse, buildErrorResponse, rpcError, rpcErrorWithData, INVALID_PARAMS
}
from "std/json" import {
    Json
}

fn main(): void {
    // A success result:
    let resultJson = Json.obj().int("sum", 5).build()
    let ok = buildResponse(RpcId.IdNum(1), resultJson)
    print(ok)
    // {"jsonrpc":"2.0","id":1,"result":{"sum":5}}

    // An error with just a message:
    let err = rpcError(INVALID_PARAMS, "expected an integer")
    let bad = buildErrorResponse(RpcId.IdNum(2), err)
    print(bad)
    // {"jsonrpc":"2.0","id":2,"error":{"code":-32602,"message":"expected an integer"}}

    // An error with structured detail in `data`:
    let data = Json.obj().str("field", "count").build()
    let err2 = rpcErrorWithData(INVALID_PARAMS, "expected an integer", data)
    let bad2 = buildErrorResponse(RpcId.IdNum(3), err2)
    print(bad2)
    // {"jsonrpc":"2.0","id":3,"error":{"code":-32602,"message":"expected an integer","data":{"field":"count"}}}
}
```

## Error handling

`readFrame`/`writeFrame` return `Result<_, FrameError>`. Every failure is a distinct variant —
never an empty string standing in for "something went wrong" — so a caller that wants to log or
branch gets to say why, not just that. What each one means, and what to do about it:

| Variant | Wire condition | What to do |
|---|---|---|
| `Eof` | Stream closed before a header or body completed. On a **fresh** read (nothing sent yet) this is the normal way a peer ends the session — e.g. an editor closing stdin. | Stop reading and exit the loop. Not an error worth logging. |
| `MissingContentLength` | A complete, well-terminated header block that never mentions `Content-Length` at all. | Terminal: the peer is not speaking this protocol, or something upstream corrupted the stream. Log and close — don't try to resync by scanning ahead. |
| `MalformedLength` | `Content-Length` is present but its value isn't a valid non-negative integer. | Terminal, same reasoning as above. |
| `TruncatedBody` | The stream ended after promising `N` bytes but before delivering all of them. | Terminal: the peer likely crashed or the connection dropped mid-message. The bytes that did arrive are discarded, not handed back as a short body. |
| `ShortWrite` | `write()` (inside `writeFrame`) accepted fewer bytes than the framed message — send side only, never returned by `readFrame`. | Treat as terminal for that connection: part of a frame is already on the wire, so there is no safe way to retry or resume mid-frame. Close and reconnect if the transport supports it. |

Only `Eof` on an otherwise-idle connection is "the client is done"; every other variant means the
framing itself broke and the stream can no longer be trusted from that point on.

## Client side: correlating responses to requests

`PendingRequests<T>` is the client half: register what you're waiting for when you send a
request, then look it up by id when the matching response arrives. It's generic over `T` so you
decide what "what you're waiting for" means — a channel, a callback, a bare label.

```milo
from "json-rpc" import {
    RpcId, RpcMessage, buildRequest, parseMessage
}
from "json-rpc/pending" import {
    PendingRequests
}
from "json-rpc/frame" import {
    writeFrame, readFrame
}

fn main(): void {
    var inFlight: PendingRequests<string> = PendingRequests<string>.new()

    // Send a request: mint an id, register what we're waiting for BEFORE
    // writing to the wire, then write.
    let id = inFlight.nextId()
    inFlight.register(id.clone(), "add")
    let payload = buildRequest(id, "add", "[1,2]")
    match writeFrame(1, payload) {
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
                            print("unknown id — duplicate reply, stale session, ignore")
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

`take()` removes the entry — a second `take()` for the same id (a duplicate reply, or a stale id
from a previous session) correctly comes back `None` rather than handing out something else's
value.

## DAP note

[DAP](https://microsoft.github.io/debug-adapter-protocol/) shares the Content-Length
base-protocol framing byte-for-byte with JSON-RPC/LSP, but its message envelope predates and
diverges from JSON-RPC 2.0 — DAP messages carry `seq`/`type`/`command`/`arguments` and
`request_seq`/`success`, not `jsonrpc`/`method`/`id`/`params`/`result`/`error`. Forcing DAP
through this package's envelope builders would either misrepresent the DAP wire format or leak
DAP field names into a JSON-RPC-shaped API. A DAP consumer imports `json-rpc/frame` only:

```milo
from "json-rpc/frame" import { readFrame, writeFrame }
```

and builds its own DAP envelope on top. If you need id correlation for DAP's reverse requests
(server → client, e.g. `runInTerminal`), `PendingRequests` still applies — DAP correlates by
echoing a numeric `seq` back in `request_seq`, the same shape JSON-RPC's numeric ids use, so
wrapping your `i64` seq as `RpcId.IdNum(seq)` gets you the same table.

## Running the tests

```bash
milo test tests/
```
