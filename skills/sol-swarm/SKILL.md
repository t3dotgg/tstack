---
name: sol-swarm
description: >-
  Orchestrate parallel high-reasoning Codex GPT Sol workers via T3 Code's
  `delegate_task` tool. Activate on complex, multi-subsystem, parallelizable,
  open-ended, or high-stakes engineering tasks where fanning out implementation,
  deep codebase audits, adversarial testing, or independent post-integration
  reviews to parallel Sol agents improves quality or speed.
---

# Sol Swarm (Parallel High-Reasoning Codex Sol Delegation)

Use this workflow to fan out multi-subsystem engineering work across parallel
**Codex GPT Sol** workers (at `high` reasoning effort when exposed) via T3
Code's `delegate_task` tool, then close with an independent read-only Sol
review over the combined diff.

Pairs naturally with `coding-discipline`, `systemic-audit`, and `sol-loop`.

## How to Launch Workers

1. **Check capabilities once per thread** before your first `delegate_task`
   call: call `orchestrator_capabilities` (only once per thread unless an error
   or target environment change occurs). Pick the latest Codex Sol model
   exposed by the provider unless the user named a specific version (do not
   hardcode a model ID blindly), and request `reasoningEffort: "high"` when
   exposed.
2. **Fan out in parallel** using `delegate_task` with `mode: "async"` and:
   ```json
   {
     "mode": "async",
     "role": "implementation",
     "isolation": "shared",
     "target": {
       "driverKind": "codex",
       "model": "<latest-sol-model-from-orchestrator_capabilities>",
       "options": { "reasoningEffort": "high" }
     }
   }
   ```
   - Set `isolation: "worktree"` whenever two or more workers may edit
     overlapping files, or when each worker should produce an independent branch
     or patch to compare or merge.
   - Use `isolation: "shared"` only for read-only workers (`research`, `review`)
     or writers with strictly disjoint directories/files.
   - Launch all independent workers in the same turn, then stop calling tools
     (or continue non-overlapping work) and let T3 Code wake you reactively as
     each finishes. Never poll `task_status` in a loop.

## Good Swarm Patterns

- **Parallel implementation slices:** Split a multi-module feature or refactor
  into disjoint subsystems (e.g., storage, async pipeline, UI/canvas, test
  harness) and dispatch one Sol worker per slice. Freeze shared contracts/types
  before fan-out and decouple parallel UI slices via render props/slots so every
  worker can typecheck and test independently with zero file collisions. When
  the repo has strict mock/dependency test loaders, grep `tests/` for owned
  component basenames before fan-out (or budget a quick coordinator test-harness
  sync right after integration).
- **Wide systemic audits:** Fan out read-only workers across different risk
  lenses (concurrency/locking, disk/fsync/I/O storms, UI scaling/rendering,
  lifecycle/state leaks) and synthesize their findings.
- **Post-integration adversarial review (mandatory closing gate on multi-worker,
  security, or hardware swarms):** After integrating parallel worker outputs,
  launch a fresh read-only `delegate_task` (`role: "review"`, high-reasoning
  Sol) over the **combined** diff to audit wire/protocol edges, rescanning/auth
  seams, reconnect/stall lifecycles, cross-worker contracts, and coordinator
  integration glue without sharing the implementers' blind spots. On
  safety-critical state machines, control/EKF math, hardware/wire protocols, or
  async concurrency, do **not** treat one green test suite or a single review
  pass as final—run iterative fresh-context read-only review passes (typically
  2–3 rounds until a pass returns clean) paired with real simulation or wire
  binaries where available rather than mocks alone, and explicitly instruct
  reviewers to audit test diffs as strictly as production code (flagging
  weakened tolerances/settle windows, reset baselines, and non-discriminating
  assertions). When composing with `sol-loop`, use `sol-swarm` for parallel
  implementation, integrate worker outputs, and run `sol-loop` strictly as a
  single-reviewer closing loop over the combined diff. For UI or mockup-driven
  swarms, pair the code review with a rendered visual check at both standard
  (`1440-1920px`) and wide (`2560px`) viewports (or explicitly report that
  rendered visual parity was not verified yet).
- **Multi-machine work:** Pass `target.environmentId` when targeting another
  linked environment.

## Writing Effective Worker Briefs

Each `delegate_task` child starts with **zero parent history**, so every `task`
prompt must be self-contained:

- **Exact scope:** repository path and owned files/modules (plus explicit
  "do not modify" boundaries when sharing a checkout).
- **Goal and invariants:** user symptoms, acceptance criteria, API/on-disk
  compatibility requirements, and non-goals. For UI slices backed by approved
  mockups, include the actual mockup file paths so workers inspect them directly
  (never rely on prose-only summaries) and state key layout geometry/chrome
  invariants (grids vs flat rows, badges, CTAs, max-width caps).
- **Verification & toolchain:** pass the repo's exact package/workspace scripts
  (`npm run type-check`, `bun run lint`, `cargo fmt --check` or `cargo fmt` on
  owned files, `cargo clippy --all-targets -- -D warnings`, etc.) alongside
  tests after the final edit so workers do not invoke raw tools (`vue-tsc`,
  `tsc`) with inconsistent flags, plus explicit toolchain `PATH`s on
  non-interactive shells when Rustup, NVM, or Homebrew are not on the default
  `PATH`. When multiple Rust workers share one checkout (`isolation: "shared"`),
  have workers run formatter checks on owned files + targeted tests (treating
  only touched-file diagnostics as failures if pre-existing warnings exist
  elsewhere) and run full-crate `cargo clippy --all-targets -- -D warnings` once
  after integration so workers do not contend on Cargo's build lock.
- **Handoff:** ask for changed files, root causes fixed, exact inline
  verification counts and concise excerpts (including fmt/lint status; never
  cite provider-private or local artifact handles that the coordinator cannot
  dereference), any intentional security/SDK compatibility or fixture breaks,
  and residual risks or assumptions.

## Coordination Discipline

- Skip the swarm for trivial 1-2 file edits that you can finish directly in one
  turn; use it freely for anything multi-file, subtle, vague, or parallelizable.
- When integrating worker outputs, skim each diff for cross-module contract
  drift, sibling-endpoint guard parity, bypassed safety interlocks,
  cached-status safety authorization or lock-drop-before-effect TOCTOU,
  partial-packet leakage after rejection, out-of-order counter rollbacks,
  stalled error-body reads, weakened or non-discriminating test diffs, backend
  wire-unit/side-effect mismatches, global `z-index`/fixed-dock occlusion,
  host-framework CSS specificity leaks, ultrawide/SVG scaling and composed-view
  duplicate headers or toolbars, decoupled auth-vs-decode parsers,
  `Option::None` fail-open paths, droppable-vs-control channel backpressure,
  unbounded fallback queues / closed-channel handling, double-charged library
  budgets, domain-specific assumptions (e.g., sequence monotonicity across
  reconnects, hardcoded single-hardware thresholds), or newly added private
  fields on public structs, then run final integrated verification on the
  combined tree before reporting to the user.
