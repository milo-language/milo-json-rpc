# json-rpc

This is a package for the [Milo language](https://milo-language.github.io/milo/).

## Overview

JSON-RPC 2.0 over the LSP/DAP base protocol: `Content-Length` framing on any fd,
request/response/notification envelopes, id correlation, and the spec's standard
error codes.

One sharp edge worth knowing before you start. `params`, `result` and
`error.data` go on the wire **already serialized**, the same convention as
`std/json`'s `JsonObj.raw`. Build them with `Json.obj()`/`Json.arr()`, or
`jsonQuote(s)` for a lone string; a bare Milo string handed to `buildResponse`
lands unquoted, which is not valid JSON there.

No dependencies beyond the standard library. Full API, the frame-error table and
the DAP notes: [docs/api.md](docs/api.md).

## Installation

```bash
milo add github.com/milo-language/milo-json-rpc
```

```milo
from "json-rpc" import { parseMessage, buildResponse }
from "json-rpc/frame" import { readFrame, writeFrame }
from "json-rpc/pending" import { PendingRequests }
```

## Examples

### A server's request loop

Reads framed requests off stdin, dispatches one method, writes framed responses
to stdout. This is the shape of an LSP or DAP server's main loop, minus the
protocol-specific method table:

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

Build it with `milo build main.milo -o echo_server` and pipe a hand-framed
request at it:

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

### Correlating responses to requests

`PendingRequests<T>` is the client half: register what you are waiting for when
you send a request, look it up by id when the response arrives. `T` is yours to
pick, whether that is a channel, a callback, or a bare label.

```milo
from "json-rpc" import { RpcMessage, buildRequest, parseMessage }
from "json-rpc/pending" import { PendingRequests }
from "json-rpc/frame" import { readFrame, writeFrame }

fn main(): void {
    var inFlight: PendingRequests<string> = PendingRequests<string>.new()

    // Register BEFORE writing to the wire: the reply can land first.
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
                            print("unknown id: duplicate reply or stale session")
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

`take()` removes the entry, so a duplicate reply or a stale id from a previous
session comes back `None` instead of handing out something else's value.
