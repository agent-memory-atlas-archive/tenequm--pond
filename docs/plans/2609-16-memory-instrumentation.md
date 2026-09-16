# Memory instrumentation and fix plan for #245

Plan of record for resolving [#245](https://github.com/tenequm/pond/issues/245)
(unbounded memory growth: 21 OOM kills in 9 days on deployment B, a no-op sync
allocating 3.8 GB in 8 s, per-query monotonic growth in `pond mcp`, and the
kill -> restart -> full re-read amplifier). Two workstreams run in parallel:
instrumentation (track A) and fixes (track B). Instrumentation is not a
prerequisite for writing the fixes - every fix target is already attributed
(#61 massif/vmmap, #229 massif, and the io-buffer evidence below) - it is the
prerequisite for *merging* the two fixes whose value must be proven by
measurement.

## 1. Why this shape

Every pathology in #245 was root-caused once, by hand, with one-off tooling:

- #61: rowmap rebuild materializes ~2.1 GB transient (`Vec<RowMetaEntry>` +
  blob); ~636 MiB of the idle floor is freed-but-retained allocator memory.
- #229: sync flush peaks at `~110 MB + 7 KB x messages + ~10x payload bytes`
  because the flush batch is bounded by session count, not bytes.
- #245: growth follows query load and is never returned until exit; the sync
  cursor is not persisted, so every OOM kill schedules a full re-read.

None of that left repeatable measurement behind. The bench suite samples RSS in
two places (`serve_mem_bench`, `commands_bench`) but nothing covers the sync
path, separates heap from RSS, measures retention or growth slope, or fails a
run on a memory number. That is how the regressions reached 0.17.x releases.

A key attribution lead found while writing this plan: `cap_serve_io_buffer`
(`main.rs`) caps Lance's `LANCE_DEFAULT_IO_BUFFER_SIZE` (default 2 GiB) to
256 MiB **only in serve/mcp**; `sync`/`copy` deliberately keep the default.
That exactly predicts deployment B's 3.8 GB no-op sync against a fast local
Lance dir vs deployment C's 243 MB no-op sync against the same store accessed
remotely (the network throttles readahead; local IO fills it).

## 2. Phases and tracks

### Phase 0 - zero-code experiment (operator, minutes)

On deployment B, which reproduces the spike on every run:

```sh
LANCE_DEFAULT_IO_BUFFER_SIZE=268435456 /usr/bin/time -v pond sync -q --no-wait 2>&1 | grep Maximum
```

If peak RSS collapses (~4 GB -> hundreds of MB), fix B1 is confirmed as a
config-level bound. If it does not move, B1's premise dies before any code is
written. Result feeds B1's launch decision.

### Track A - instrumentation core (1-2 days)

Branch `feat/memory-instrumentation`. The minimal slice that lets a fix carry a
before/after number. Detailed spec in section 3.

1. `memprobe` module behind `mem-probe`/`dhat-heap` features
2. `mem_bench` with the 4 scenarios mapping to the active fires
3. `ops/scripts/profile-mem.sh` (heaptrack/dhat) + `[profile.profiling]`
4. JSON per scenario, appended to `docs/benchmarks/mem-gate-baseline.jsonl`

Explicitly NOT in track A: CI wiring, thresholds/tiering, the other 4
scenarios, serve_mem_bench refactor, PeakRecordingPool, telemetry. All deferred
to hardening (section 5) so fixes are not blocked on gate plumbing.

### Track B - fixes (parallel worktrees, one PR each)

| # | Branch | Fix | Verified by | Merge gate |
|---|---|---|---|---|
| B1 | `fix/sync-io-buffer` | bound the sync/copy scan readahead (extend the io-buffer cap to sync, config-overridable) | Phase 0 measurement on deployment B | field number; merge when green |
| B2 | `fix/sync-cursor-persist` | persist the sync cursor for `serve --with-sync` so a kill does not schedule a full re-read (`syncstate.rs` is the seam) | `first sync from this host` marker gone from the journal after restart | field/journal; merge when green |
| B3 | `fix/sync-flush-byte-budget` | #229: byte budget (~32-64 MB) on the buffered flush batch + chunked encode/append in `messages_batches`; partial flush is already idempotent | `ingest-large-session` + `sync-incremental` before/after rows | **blocked on track A rows** |
| B4 | `fix/linux-alloc-retention` | glibc retention on Linux (gnu builds): `malloc_trim(0)` after sync/rowmap peaks and/or `MALLOC_ARENA_MAX` guidance; `cfg(linux)` only - #61 proved allocator swaps regress macOS | `mcp-query-growth` + `rowmap-build-cold` retained-bytes before/after | **blocked on track A rows** |

B1+B2 alone likely turn deployment B from "21 OOM kills in 9 days" into
"stable"; they merge on field evidence without waiting for track A.

"Before" rows are never lost by this parallelism: once `mem_bench` lands, the
pre-fix commit is checked out into a pool worktree and the scenario runs there,
producing the before row retroactively.

### Later - structural fixes (measured first, then designed)

- Streaming rowmap build (#61 names the blocker: dict indexes borrow into
  `entries`; own the dict keys up front). Target peak ~400 MB. 1-2 weeks.
- Shared daemon + thin stdio shim per session. Architectural; sequenced last -
  B1-B4 shrink what each per-session process costs, which may soften it.
- `oom_score_adj` documentation/drop-in (+200 confirmed correct on two
  deployments) and `MemoryMax=` guidance.

### Later - hardening (after fixes are landing)

The remaining 4 scenarios (`sync-first-full`, `rowmap-delta-compact`,
`sql-heavy`, plus a serve-idle-floor port of serve_mem_bench's check),
thresholds file with two tiers (tier 1: regression ceilings from measured HEAD;
tier 2: targets from #61/#245, flipped to enforced as fixes land), CI job on
`pond-ci` with the 4 cheapest scenarios per PR (warn-only first), sync/serve
telemetry lines (VmHWM/VmRSS/RssAnon at sync end and on the 30 s refresh loop),
`PeakRecordingPool` under `mem-probe` in `pond_sql`, a startup warning when
`LANCE_BYPASS_SPILLING` is set, and optionally a gungraun hard-limit lane.

## 3. Track A spec

### memprobe (`packages/pond/src/memprobe.rs`, feature-gated)

```toml
[features]
mem-probe = []            # counting allocator + RSS probes; benches only
dhat-heap = ["dep:dhat"]  # allocation-site attribution; ad hoc only

[dependencies]
dhat = { version = "0.3.3", optional = true }
```

- Counting allocator: ~25-line wrapper over `System`, `AtomicUsize`
  current/peak/total with `fetch_max` on alloc. Global atomics, accepted
  contention cost in bench builds, because thread-local schemes undercount
  multi-threaded peaks and the true global peak is the metric. Existing crates
  are stale (`cap` 2023, `stats_alloc` 2022) or single-threaded
  (`allocation-counter`). Feature off = the `#[global_allocator]` item does not
  exist = zero impact on shipped binaries.
- RSS probes: Linux writes `5` to `/proc/self/clear_refs` to reset the kernel
  high-water mark, then reads `VmHWM`/`VmRSS`/`RssAnon` from
  `/proc/self/status`. macOS keeps the existing `ru_maxrss`/`phys_footprint`
  helpers (moved here from serve_mem_bench in the hardening phase, not now).
- Sampler: 200 ms thread sampling `VmRSS`; emits max/final/series so a
  scenario distinguishes one spike from monotonic growth.
- Output per scenario: `{scenario, peak_heap_bytes, end_heap_bytes,
  total_alloc_bytes, peak_rss_kb, end_rss_kb, rss_anon_end_kb,
  growth_slope_bytes_per_iter?, wall_ms}`.
- dhat mode: same binary, `--features dhat-heap`, writes `dhat-heap.json` for
  the online DHAT viewer (per-site attribution, sorted by at-t-gmax or
  at-t-end). Diagnosis lane only - dhat is slow and cannot reset mid-run.

### mem_bench (`benches/mem_bench.rs`, harness = false)

One scenario per process invocation (`mem_bench --scenario <name>`): peak RSS
is a process-lifetime high-water mark and dhat cannot reset, so the runner
invokes the binary once per scenario.

| Scenario | Exercises | Catches |
|---|---|---|
| `sync-noop-local` | full sync pipeline, zero new data, local store | the 3.8 GB no-op spike; store-size-proportional sync memory |
| `sync-incremental` | +1 session / +N messages then sync | peak scaling with delta vs store |
| `rowmap-build-cold` | `ensure_rowmap` from nothing | the #61 ~2 GB transient |
| `mcp-query-growth` | N iterations of search + get + sql | the per-query ratchet; retained-bytes slope per iteration after warmup |
| `ingest-large-session` | one session, parameterized steps (#229 shape) | flush-batch byte scaling (B3 needs it, so it lands with or before B3's merge) |

Corpus: generated through the existing `session_events`/`ingest_batched` path,
`--profile ci` (~100k messages) and `--profile large` (1M+), content-addressed
by (generator version, profile) and cached locally.

### Runner and baseline

`ops/scripts/mem-gate.sh` v1: build once, invoke per scenario, append one
combined row to `docs/benchmarks/mem-gate-baseline.jsonl`, print delta vs the
previous row. No thresholds yet - the human reads the delta. Thresholds and CI
arrive in hardening.

### profile-mem.sh

`ops/scripts/profile-mem.sh <scenario> [heaptrack|dhat]` against the same
corpus; heaptrack needs `[profile.profiling] inherits = "release", debug = 1`
(workspace root; dist builds keep the normal release profile). Never wraps
`cargo run` (that profiles cargo). Replaces the ad-hoc recipe in the #245
comments.

## 4. Design principles (bind both tracks)

1. Zero impact on shipped binaries: all allocator instrumentation is
   feature-gated off; production changes are limited to a few `/proc` reads
   and tracing lines (hardening phase).
2. One scenario, one process.
3. Heap and RSS always reported together - the gap is the allocator-retention
   signal.
4. A failing number must point at the cause: every scenario is re-runnable
   under dhat/heaptrack with one command.
5. Fixes merge on evidence: field evidence for B1/B2, before/after scenario
   rows for B3/B4 and everything after.

## 5. Acceptance

- Track A: `mem-gate.sh` produces a full baseline row on the `ci` corpus; each
  #245 pathology has a scenario that visibly exhibits it on pre-fix HEAD
  (rowmap transient, sync spike shape, mcp slope > 0).
- B1: deployment B's no-op sync peak drops to the same order as deployment C's
  (hundreds of MB, not GB).
- B2: a killed/restarted `serve --with-sync` resumes from the cursor (no
  `first sync from this host` full re-read).
- B3/B4: before/after rows show the intended reduction with no equivalence or
  throughput regression (`write_bench` guards the write path).
- Deployment B's OOM cadence is the ultimate metric: target zero pond OOM
  kills over a 7-day window after B1+B2 deploy.
