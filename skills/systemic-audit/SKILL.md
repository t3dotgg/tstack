---
name: systemic-audit
description: Audit medium or large codebases (and optional runtime logs) for systemic performance bottlenecks, disk/fsync storms, runtime/lock contention, rendering/WebGL lag, cross-app UI scale drift, and lifecycle/data-loss defects without tunnel vision or excessive token burn.
---

# Systemic Audit

Find the highest-impact cross-cutting defects in a codebase quickly and completely. Budget reads aggressively: check available runtime logs first (when present), map and pattern-search the repo, then inspect only high-signal files in bounded slices.

## Scope

- Treat audit requests as strictly **read-only** unless implementation is explicitly requested. Do not modify files or run git write commands.
- Do not load unrelated prose/style skills (`unslop`, `i-have-adhd`) during a technical audit. Completeness and exact citations outrank formatting rituals.

## 4-Pass Audit Method

### Pass 1: Logs + architecture & hot-path map (1–2 commands)
If application logs or recent telemetry are referenced/available, summarize disk read/write volume, stall timestamps, RSS, and log volume first to ground hypotheses in real measurements. List top-level directories, runtime setup, storage/persistence modules, renderers/UI layers, and background loops via `rg --files` and targeted symbol searches. Do not dump large files blindly.

### Pass 2: High-yield pattern sweeps (batched `rg`)
Run targeted `rg` sweeps across the whole repo to catch cross-cutting defect classes and ALL sibling sites in one pass:

1. **Disk / fsync / write storms & idle I/O:** `sync_all`, `sync_data`, `fsync`, `write_atomic`, `set_times`, `setItem`, `localStorage`, periodic save/hash timers, per-item loops calling `save`/`commit`, unconditional disk recording even when no viewer/feature is open, ephemeral fields (`last_sync_ms`) triggering full-document saves.
2. **Runtime proliferation, blocking async workers & lock contention:** count `Builder::new_multi_thread` / thread pools and check OS QoS priorities (avoid video/UI outranking control/navigation); check `std::fs`, `block_in_place`, subprocess spawns, or `Mutex`/`RwLock` guards held across `.await`ed network/disk/RPC I/O or timeouts; check `Instant::now()` captured before `.await` with `checked_duration_since` (clock-order drop race) and tight timeouts on single-worker runtimes.
3. **Frontend re-render, WebGL/canvas & per-event rebuild storms:** root-level fast timers (`setInterval`/`setNow`) re-rendering large app trees, closed modals formatting/computing on every render, `hasFeatureAtPixel` on `pointermove` without `!event.dragging`, `updateWhileInteracting`/`updateWhileAnimating` on large vector layers, `toDataURL`/`readPixels`/per-item `drawElements` loops, `JSON.parse`/`structuredClone` on hot renders, duplicate pollers, uncompressed/compressed loopback overhead (`CompressionLayer`).
4. **Cross-app coordinate & UI scale drift:** `uiScale`, `zoom`, `getBoundingClientRect`, `ResizeObserver`, `contentRect`, `clientX`, `devicePixelRatio`—compare every consumer against the shared scaling helper to spot unscaled sibling views.
5. **Data-loss, crash-atomicity & lifecycle gaps:** missing-file fallbacks that overwrite intact sibling data, multi-step file/dir renames without startup recovery, `catch`/fallback branches replacing unreadable state with empty defaults, lost-update whole-document saves without revision checks, cache `clear()` vs in-flight resurrection, missing timer/listener teardown on switch/unmount, GPU/WebGL context-loss handlers missing restoration.

### Pass 3: Verify mechanisms with bounded reads
Open only the matched line ranges and their immediate callers/callees. Confirm each candidate has a **reachable failure path**; drop speculative concerns or already-mitigated paths. For every confirmed defect, check whether sibling modules repeat it or already solve it better, and note any prior "band-aid" fixes that amplified downstream load.

### Pass 4: Rank and report cleanly
Rank findings **P0 (data loss / corruption / startup failure)** then **P1 (live disk storms, runtime stalls, UI lag, wrong coordinates, broken lifecycle)**, grouped by subsystem. For every finding include:

- **Severity + concrete headline** (state the exact failure, not vague symptoms).
- **Mechanism & trigger:** exact execution path, bounded rates/sizes from source (or measured log numbers), and user-visible impact.
- **Exact citations:** `[symbol or step](/abs/path/file:line)` for every link in the chain.
- **Sibling status:** name sibling modules that repeat the bug vs siblings that already handle it safely.
- **Cheapest structural fix** (prefer removing blocking I/O/locks/timers over adding more runtimes or layers) and the **single most informative runtime check** if execution wasn't run.

State clearly what was inspected (commit/branch, logs if any), what was NOT executed, and do not cap findings artificially if real P0/P1 issues remain.
