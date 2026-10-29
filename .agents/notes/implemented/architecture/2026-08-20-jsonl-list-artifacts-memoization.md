# Agent Note: JSONL listing probes stats and memoizes header lines

Status: implemented

English | [中文](2026-08-20-jsonl-list-artifacts-memoization.zh.md)

## Problem

A saturated `dsh web` host degraded listing RPCs (`subagent.list`, `session.list`) from ~0.5 s to ~100 s while several agents streamed; the CPU-side half of that saturation — per-connection fanout multiplication on the event stream — is addressed separately. Sharing the fanout removed the multiplication, but the JSONL backend's `listArtifacts` remained expensive by construction: per session directory it awaited two open-close existence probes plus a full header read (open, 8 KB read, decompress, close) — five to seven sequential fs round-trips per session across every project directory, ~600 awaited ops for a 115-session store. Under a near-busy event loop each round-trip queues behind loop work, turning a ~250 ms offline scan into tens of seconds.

## Decision

**Stat identity for header reuse.** `statArtifact` reads the selected generation's bigint device and inode before `headerLine` consults the memo. ENOENT still runs the parent-allows-absence guard, so an inaccessible Session directory remains a storage fault. Generation selection and root encoding checks remain the backend's discovery responsibilities.

**Header memo keyed by artifact path, valid while the stat identity (dev, ino) matches.** The first frame of an append-only log is immutable: appends and tail-truncating crash repair never rewrite frame 0, and materialization publishes a fresh inode. A warm listing therefore performs no log read at all; a replaced file at the same path (new inode) re-reads once. The memo is unbounded by design — one header line per stored session, bounded by the on-disk inventory the backend never prunes.

**Bounded parallel discovery and header reads.** The backend's shared `listLimit` applies the configured `listConcurrency` (default 3) across directory, header, and revision reads. Results and failures follow traversal order. `listArtifacts` preserves translated header metadata, physical paths, and source versions; corrupt or unsupported headers are omitted from listing while direct reads retain their diagnostics.

## Consequences

Warm listings avoid header reads while retaining filesystem discovery and revision checks, including the corpus metadata scan required by historical formats. The original 167-session experiment on a slow disk measured 576–725 ms per listing without the memo, 254 ms for the first memoized listing, and 54–86 ms for warm listings; those figures describe that implementation and input. The regression tests require zero `FileHandle` reads for warm listings, a fresh header after a renamed-in replacement, and cancellation of cold header reads.

## Alternatives considered

- **Durable header index maintained by writers** — O(1) listings, but a new on-disk format plus cross-process coordination; the pre-release stance permits format changes, but the memo reaches warm O(stat) with neither.
- **(size, mtime) validation** — appends bump both, so actively written sessions would never hit; (dev, ino) is the invariant that actually holds for frame 0.
- **Parallelism alone** — improves cold listings but leaves warm listings paying full header reads every time.
