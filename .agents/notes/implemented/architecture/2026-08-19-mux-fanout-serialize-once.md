# Agent Note: Mux fanout is shared and serializes once

Status: implemented

English | [中文](2026-08-19-mux-fanout-serialize-once.zh.md)

## Problem

A `dsh web` session running many concurrent agents (a workflow surveying the repo with six to eight parallel subagents) pegged the host process's main thread at 100% of one core while RPC reads (`subagent.list`, `session.list`) collapsed from ~0.5 s to ~105 s. Reproduced in isolation on a copied session store: with one streaming session and no connections the CPU sat at ~20% and reads took 0.6 s; holding twelve WebSocket connections pushed the same load to ~76% CPU and one `subagent.list` to 82.7 s. The gap is a saturation cliff — once the loop is near-busy, every awaited file operation in a listing queues behind it.

Fanout accounting showed the mux broadcast multiplied per-consumer work three ways, all measured on the live server and the isolated repro:

- The `session/event`, `session/created`, `session/disposed`, and jobs listeners were registered once **per open stream**, so every streamed event ran `viewFor` (including presenter calls and call-pairing backscans) and minted a fresh rpcId once per connection.
- Each carrier then re-serialized the frame: the WebSocket downlink called `JSON.stringify` per socket per frame, and the SSE carrier did the same per GET stream.
- A broadcast to twelve consumers therefore paid twelve times for everything except the socket write. In the repro, twenty minutes of six sessions pushed 431 k frames / 150 MB to twelve connections.

## Decision

**One mint, one serialization, N writes.** The `EventsApi` contract now yields `StreamFrame<F> = { request, wire }`: the frame's server-request view plus its JSON bytes computed once at push time. `broadcast()` builds one `StreamFrame` and pushes the identical item object to every queue; both carriers (WebSocket downlink and the fetch handler's SSE path) send `frame.wire` verbatim. The wire bytes are unchanged — the browser parses exactly what it parsed before; only the host-internal seam type moved.

**Shared listeners, registered on the first stream open.** The four hot-path listeners moved out of `events.mux()` into a proxy-level `registerMuxFanout()` called on the first open. The open-call pairing table (`openCalls`) is shared — the pairing is a property of the session's event stream, not of any one connection; table misses still backscan. Registration is lazy for two reasons, both load-bearing:

- **Zero-consumer cost is exactly zero.** A proxy with no open stream registers no `session/event` listener at all. An earlier draft registered them eagerly at proxy setup; under coverage instrumentation the per-event guard alone pushed a search test past its 5 s timeout (baseline passes; the eager version failed), and there is no reason a streamless proxy should observe the event bus at all.
- **`ctx.inject` activates asynchronously.** Registering the jobs listener through `ctx.inject(['jobs'], …)` missed registry changes that followed proxy creation in the same tick (the jobs spec's start/kill/settle sequence fired before the inject child activated, so only the settlement push arrived). The lazy registration reads `ctx.get('jobs')` synchronously at first open — the same timing the per-stream code had — and a jobs registry composed later is picked up at the next open.

After the last consumer leaves, the listeners stay registered but no-op on `muxQueues.size === 0` before any frame mint, presenter work, or serialization.

**Per-connection state stays per-connection.** Open-time baselines (subscribed frames, pending approval/question replay with stable rpcIds, queue snapshot, jobs snapshot) are still minted and serialized per stream, at open frequency. Empty-set jobs pushes still send `[]` on unowned registry changes — that transition is the one absence cannot express.

## Consequences

Per streamed event per consumer, the remaining cost is one `socket.send` of pre-computed bytes. The `session/jobs` empty-snapshot semantics, the approval/question replay contract, and the wire format are unchanged and covered by the existing suites; a new spec asserts the sharing contract directly (two consumers receive the identical `StreamFrame` object — reference equality — whose `wire` parses back to the frame's server-request envelope) and that events with no open stream reach no consumer. The client-side fixture mirrors the contract by parsing each frame's `wire` through `serverRequestSchema` on its in-process tap, so every fixture journey now also exercises the serialized form. The consumer-side `IApiClient` face is unchanged: it still yields `RpcRequest` envelopes parsed from the wire, so the browser bundle does not see the `StreamFrame` type.

## Alternatives considered

- **Per-connection WeakMap serialization cache in the carriers** — would have fixed the `JSON.stringify` duplication without touching the contract, but left the per-connection listener duplication (viewFor + rpcId mint, the larger half of the fanout cost) in place, and hid the sharing as an implicit carrier optimization instead of a contract.
- **Eager proxy-level listeners** — simpler registration, but adds a per-event cost to every streamless proxy; rejected on the measured coverage-run regression above.
- **Frame batching / per-connection subscription filtering** — larger wins at higher complexity (a wire-format change for batches; a subscribe protocol for filtering). Deferred until the shared fanout's effect is measured on a real deployment.

## Measurement appendix

Isolated repro (copied session store, same binary): idle server 2% CPU, `subagent.list` 0.54 s; one streaming session, no connections 30% CPU, 0.6 s; six sessions + twelve connections 76% CPU, `subagent.list` 82.7 s (live server with workflow waves: 100% CPU, 105 s). The fanout sharing removes the per-connection serialization and per-connection listener work; the remaining per-connection cost is the socket write itself.
