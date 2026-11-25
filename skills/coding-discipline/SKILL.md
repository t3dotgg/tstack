---
name: coding-discipline
description: Apply when implementing features, fixing bugs, refactoring, or hardening code in any language. Enforces cross-cutting sibling sweeps, cascade-free root-cause fixes, boundary/type/idempotency discipline, full lifecycle completeness, real hot-path complexity, post-final-edit verification, and trustworthy engineering handoffs.
---

# Coding Discipline

Produce code another engineer or orchestrator can accept without reworking it. Fix the whole pattern cleanly, avoid band-aid cascades, verify the final tree, and report exact evidence.

## 1. Sweep before editing (anti-tunnel-vision)

Read the target files and their callers first. Whenever fixing a bug or changing an invariant—coordinate/UI scaling, persistence/fsync cadence, caching, error handling, schema/API shape, or resource cleanup—search the codebase (`rg`) for **every sibling module and caller** sharing that symbol, helper, or pattern before writing code.

- Fix all affected sibling sites consistently using one shared helper or authoritative owner. Before running tests, grep every sibling field, enum arm, and raw channel/helper call site in the touched module so clamps, guards, filters, and wrapper migrations apply uniformly with zero leftover raw calls.
- When adding a new endpoint, stream, or handler beside existing ones, mirror sibling invariants (selection/feature flags, auth, epoch/generation guards) rather than querying raw internal state directly.
- Preserve existing public signatures, return shapes, CLI flags, exit codes, and caller expectations unless the request explicitly changes them.
- Keep unrelated cleanup out of scope; fix siblings that share the broken invariant, not stylistic preferences.

## 2. No band-aid cascades (trace downstream cost)

Solve the root bottleneck instead of layering workarounds that push the failure elsewhere:

- **Trace what helpers actually do before scaling them up:** never raise a batch size, polling rate, or worker count without checking downstream syscalls (`sync_all`/`F_FULLFSYNC`, thread spawning per tick, directory scans, full-document clones).
- **Do not proliferate runtimes, caches, or kill-switch timeouts** to hide a stall caused by sync I/O or a broad lock—remove the blocking I/O or shrink the lock hold instead.
- **No always-on work without an active consumer:** gate heavy disk recording, polling, canvas/rAF loops, and UI formatting on actual open/active features. Skip persistence writes when only an ephemeral timestamp changed.
- **Never hold locks across slow I/O or `.await`**, and **never capture `Instant::now()` before an `.await` or lock wait** to reject fresher work. When messages or telemetry carry monotonic domain IDs (`seq`, `scan_id`, `revision`), order state updates by those domain IDs—never let host arrival time allow an older packet to overwrite a newer state.

## 3. Boundary, type, and idempotency discipline

- **Validate at boundaries:** parse and narrow untrusted input (CLI args, JSON, config, network, files, SQL parameters, floats like `NaN`/`Inf`) once at the entry boundary. Never interpolate untrusted values into SQL, shell commands, or paths; allowlist dynamic identifiers (`ORDER BY`, table/column names).
- **Verify protocol specs against multi-instance and multi-hardware reality:** when parsing hardware/wire packets (MAVLink, CAN, sensors), check source/instance identity (`id`, `system_id`, `component_id`), packed vs unpacked field layouts, exact `<` vs `<=` buffer/delay limits, and protocol extension fields rather than assuming single-instance payloads, and avoid hardcoding single-hardware/pack thresholds when firmware or protocol parameters exist. Before formatting or thresholding telemetry units (e.g., `%` vs `dBm`, `Pa` vs `hPa`), labeling refresh/rescan actions, reporting active/publishing status on a slot/route, or invoking destructive/reboot endpoints, inspect the backend producer or handler for the true wire unit, real configuration/side effect (configured slot vs unconfigured default, cached read vs active scan), and required safety interlocks (never bypass armed/power guards).
- **Authenticate what you decode & trace downstream sentinels/flags:** verify third-party decoder/transport behavior on malformed, prefixed, or batched inputs (e.g., rescanning parsers or buffered UART chunks) and authenticate the exact decoded frame rather than checking raw offset 0 separately. Never route malformed/corrupt input into protocol lost-signal sentinels, and trace every downstream consumer whenever changing a sentinel value (e.g., `0`), validity bit, or `Display`/error contract.
- **Model state honestly:** use sum types / discriminated unions / enums so illegal combinations cannot exist; refuse `any` casts and silent `except Exception: pass` error swallowing. Never conflate "absent/not configured" with "corrupt/invalid" in `Option::None` on security or safety paths—fail closed with an explicit error.
- **Verify library accounting and cap semantics:** before pre-reserving budgets or wrapping decoders/limiters (e.g., `image::Limits::reserve` + `from_decoder`), verify whether the callee already accounts for the same bytes/quota so budgets are never double-charged or silently halved. When hardening untrusted-page shims, capture every relied-on global intrinsic (including timers like `setTimeout`).
- **Make mutations safe on retry and crash without breaking formats:** wrap multi-step storage writes in atomic staging/temp+rename commits, bound append logs with compaction, keep `fsync` O(1) per batch rather than per item inside a loop, handle mid-batch failures without half-applied state, and preserve existing on-disk file names and formats unless a migration is requested.

## 4. Trace the complete lifecycle

For any stateful, buffered, cached, subscribed, or async component, check all five transitions before finishing:

1. **Init / empty state:** buffers, view arrays, and defaults start in a valid state (e.g. non-zero masks, no mutable default arguments).
2. **Steady-state updates:** incremental paths preserve invariants under appends, edits, truncations, wraparound, burst/equal timestamps, and clock skew. Reject out-of-order sequence/reset-counter rollbacks (e.g., `7 → 8 → late 7 → 8`), never let zero/unset timestamps trip duplicate suppression, and reject stale or invalid multi-field packets atomically (never leak velocity/twist downstream after rejecting pose).
3. **Clear / switch / reconnect:** clearing, switching active context, or losing health and reconnecting to the **same endpoint** resets all dependent caches, overlays, trails, live-reading flags, and one-shot fallback probes (never conflate a boot-default fallback value with confirmed live telemetry). Carry `generation`/`epoch` directly inside published snapshots rather than reading an external atomic counter around an unversioned snapshot so stale completions or torn reads cannot resurrect cleared state. For safety-critical, armed, reset, or mode-transition gates, authorize only from fresh post-transition state (never cached heartbeats or pre-reset status), invalidate prior reset/delivery tokens on mode overrides, and fence split stream/TCP frames across epoch changes.
4. **Error / retry / stall:** failed flushes or writes keep pending data recoverable without duplication or silent loss; distinguish transient retryable errors from permanent ones (e.g., `TrySendError::Full` vs `Closed`—never enqueue onto a closed transport, leave fallback/retry queues unbounded, or permanently consume a one-shot fallback on transient failure). Bound error-body and non-2xx response reads with explicit timeouts so a stalled upstream error cannot burn freshness windows or block recovery. For push/WebSocket/SSE streams with fallback polling, enforce a client-side first-frame and inter-frame staleness watchdog so half-open silent links trigger fallback and reconnect instead of waiting indefinitely on `onclose`/`onerror`.
5. **Shutdown / dispose:** flush pending dirty writes, abort pending retry timers cleanly, unsubscribe all event listeners, fence post-`await` registrations so callbacks resolving after unmount/dispose cannot leak listeners or timers, and wait for in-flight work to settle.

## 5. Real complexity on the default path

When fixing performance, I/O storms, or quadratic work:

- Optimize the **default caller path**—never leave existing callers on an O(N²) scan behind an optional fast-path argument.
- **Eliminate irrelevant work before memoizing:** in high-count rendering, indexing, or event loops, cut out-of-scope work (viewport culling, dirty-only updates, closed-UI unsubscription) rather than only caching per-item math while still iterating or allocating for all `N` items; inspect neighboring modules for existing bounds/scale helpers.
- Never game a counter (e.g. keeping `rebuildCount == 1` while scanning `0..N-1` on every append). Prove actual per-operation complexity and measure real work (writes, fsyncs, queries, redraws).
- Match durability to data criticality: disposable caches and live previews do not need per-item `fsync`; batch fsyncs per commit rather than per item inside a loop; separate high-frequency continuous updates from discrete user edits and flush cleanly on shutdown/export.

## 6. Verify after the final edit

- Run the repo's formatter check, strict lint/typecheck gate, and relevant tests on every touched file/crate **after your very last file edit** (e.g., `cargo fmt --check` or `cargo fmt` on owned files + `cargo clippy --all-targets -- -D warnings`, `tsc --noEmit`, or the project's lint suite)—not just `cargo check` or focused unit tests. In parallel shared-checkout Rust runs (or crates with pre-existing unrelated warnings), run formatter checks and targeted tests on owned files, fix all warnings in touched files, and leave full-crate `clippy` to the post-integration pass to avoid Cargo build-lock contention. Any subsequent edit—even a one-line tweak—invalidates prior results and requires a fresh run.
- Confirm the intended tests actually executed and would fail without your fix; verify downstream consumer contracts first so tests never codify a flawed local assumption about sentinels or guards. Make new tests discriminating: assert that required stimuli/preconditions actually held during measurement, perturb the scenario so fallback/coasting cannot pass by accident, and include asymmetric, partial-`NaN`, zero/boundary, and out-of-order inputs alongside nominal fixtures. For resource/size/rate budgets, test near the real cap (e.g., 60–90% allowed plus >100% rejected) so a silently halved budget cannot hide behind tiny fixtures. Never weaken existing tests, widen tolerances/settle windows, or reset baselines/references to make a failing test pass—inspect real failure logs and fix the root cause (or tighten only to measured physical/spec bounds).
- Leave the working directory and temp dirs clean of stray temporary files or logs.

## 7. Trustworthy handoff

Lead with the outcome and exact evidence:

- **Changed files** and the root cause fixed in each.
- **Sibling sites checked** (which were updated and which were verified already safe).
- **Post-final-edit verification:** exact command(s), inline test/check counts, and observed result (inline concise failure/log excerpts directly—never rely on provider-private or local artifact handles that another agent, worktree, or machine cannot dereference).
- **Trade-offs & limits:** call out any intentional security/behavior tightening that breaks existing SDK callers, tests, or fixtures (and what follow-up allowlist/migration is needed), plus any subtle edge-case assumption or unverified runtime/hardware boundary.
