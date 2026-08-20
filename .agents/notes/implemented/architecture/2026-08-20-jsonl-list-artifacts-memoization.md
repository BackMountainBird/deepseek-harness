# Agent Note: JSONL listing probes stats and memoizes header lines

Status: implemented

English | [中文](2026-08-20-jsonl-list-artifacts-memoization.zh.md)

## Problem

A saturated `dsh web` host degraded listing RPCs (`subagent.list`, `session.list`) from ~0.5 s to ~100 s while several agents streamed; the CPU-side half of that saturation — per-connection fanout multiplication on the event stream — is addressed separately. Sharing the fanout removed the multiplication, but the JSONL backend's `listArtifacts` remained expensive by construction: per session directory it awaited two open-close existence probes plus a full header read (open, 8 KB read, decompress, close) — five to seven sequential fs round-trips per session across every project directory, ~600 awaited ops for a 115-session store. Under a near-busy event loop each round-trip queues behind loop work, turning a ~250 ms offline scan into tens of seconds.

## Decision

**Stat probes instead of open probes.** `probeSessionDir` stats the opposite-encoding artifact and the primary artifact with bigint identity. Absence keeps `exists()` semantics: ENOENT still runs the parent-allows-absence guard, so a blocked session directory remains a storage fault rather than silent absence. The opposite probe still throws the encoding mismatch; the primary probe's identity feeds the memo.

**Header memo keyed by artifact path, valid while the stat identity (dev, ino) matches.** The first frame of an append-only log is immutable: appends and tail-truncating crash repair never rewrite frame 0, and materialization publishes a fresh inode. A warm listing therefore performs no log read at all; a replaced file at the same path (new inode) re-reads once. The memo is unbounded by design — one header line per stored session, bounded by the on-disk inventory the backend never prunes.

**Bounded parallel probing.** Session directories are probed by `LIST_PROBE_CONCURRENCY = 16` workers, an internal scheduling constant in the family of subagent listing's `COLD_READ_CONCURRENCY = 4`. Results land by directory index, so artifact order and duplicate-id rejection stay deterministic; which of several concurrent per-directory failures surfaces first is not ordered. `listArtifacts`' return shape (`{ header, path }`) is unchanged — `listSnapshots` keeps its own post-discovery stat, which doubles as the external-deletion guard between discovery and snapshot.

## Consequences

A warm listing of N sessions costs ~5 `readdir` + 2N `stat` plus cache misses. Measured back-to-back on a 167-session store on a slow disk: every listing cost 576–725 ms before; after, the first listing costs 254 ms and warm listings 54–86 ms — the steady-state scan is gone. Inode-number reuse cannot misattribute a header across paths (the key includes the path), and same-path recreation arrives on a new inode that misses. Two listing tests changed with the behavior: the header-read cancellation tests no longer warm the listing first (a warm listing reads nothing, so there is nothing to cancel), and the corrupt-frame listing assertion matches either corrupt frame because parallel probing settles first-error nondeterministically. New specs pin the memo contract directly: a warm listing performs zero `FileHandle` reads, and a renamed-in replacement at the same path lists the new header.

## Alternatives considered

- **Durable header index maintained by writers** — O(1) listings, but a new on-disk format plus cross-process coordination; the pre-release stance permits format changes, but the memo reaches warm O(stat) with neither.
- **(size, mtime) validation** — appends bump both, so actively written sessions would never hit; (dev, ino) is the invariant that actually holds for frame 0.
- **Parallelism alone** — improves cold listings but leaves warm listings paying full header reads every time.
