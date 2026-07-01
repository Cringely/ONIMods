# Fast Track Optimization Harness — Design

Date: 2026-06-30
Author: Cringely (jcgamo88@gmail.com)
Status: Approved design, pending implementation plan

## Summary

This is not a rewrite of Fast Track. It is a repeatable, mostly-autonomous pipeline that runs agents against a fork of PeterHan's Fast Track and the current ONI build, discovers candidate optimizations, proves each one correct and faster, and queues it for a human accept/reject on real numbers.

The work targets two things: net-new optimizations the author has not done, and targeted rewrites of patches that no longer apply to the current Aquatic / Unity 6 build. We do not rewrite the whole codebase. A prior ponytail review found Fast Track is already tightly engineered, so the value is in additive finds and surgical repairs, not wholesale replacement.

The design exists partly to answer the maintainer's stated objection. On PR #712 Peter Han wrote that "AI agents are not yet at the level of deeply understanding how Fast Track works." Every stage here produces evidence (IL diffs, unit tests, a debate transcript, A/B frame numbers) so a candidate is defensible rather than plausible-looking. Net-new optimizations are additive and will not collide with Peter's in-progress compat rewrites.

## Goals

- Find optimizations Fast Track does not already implement, ranked by measured hot-path cost.
- Repair or rewrite patches whose targets moved/renamed on the current build.
- Prove correctness and performance for each candidate before it lands.
- Keep a per-candidate evidence trail suitable for an upstream PR if desired.

## Non-goals

- Rewriting Fast Track from scratch.
- Replacing optimizations that already work and are tested.
- Fully unattended game automation (see Constraints).
- A formal security/audit program (this is a local game mod, no network or untrusted-input surface).

## Constraints (the reality this design is shaped by)

- ONI is Mono backend. The Unity Profiler will not attach to the retail player. Measurement is in-mod logging plus A/B benchmark runs, not external profiling.
- A bad IL/transpiler patch can hard-crash the game with no crash handler. In-game testing is gated and batched, never run blind in a tight loop.
- Game-loaded code has no test runner. Only the pure, game-decoupled island is unit-testable.
- Fast Track is MIT licensed (© 2024 Peter Han). The fork preserves the license and attribution.
- Mod option toggles gate Harmony patches at load via `Prepare()`. Changing a flag requires a game restart; flags do not un-apply live.

## Deliverable target

The primary deliverable is a working high-performance fork that the user runs. Candidates are written to PR quality so the work can be offered upstream later, but the pipeline does not block on the maintainer's acceptance.

## Repository layout

```
FastTrack/                  working fork (already in repo)
harness/
  candidates/               one .md per proposed optimization (queue + ledger + evidence + raw A/B logs)
  decompiled/               cached ilspycmd output of game DLLs (gitignored, large)
  benchmark/                fixed late-game save + logging patch + diff script
  tests/                    netstandard test project for the pure island
```

The candidate `.md` files are the orchestration substrate. There is no separate job database or queue server, and no separate reports directory: measured numbers and the raw A/B `Player.log` captures land in the candidate file itself. Each file is the full record of one optimization from idea to accept/reject.

## Model assignment

- REVIEW panel and chairman: Opus. Judgment lives here.
- Every other stage (survey, design, patch authoring, unit-test writing, IL tooling, measurement and diff): Sonnet 5.

## The loop

Stages 1 through 4 run unattended. Stage 5 is the human checkpoint.

```
1 SURVEY    rank opportunities from decompiled hot paths + Fast Track DebugMetrics output
2 DESIGN    candidate .md: what / where / why, predicted impact, risk class, collision check
3 BUILD     patch + unit test (if logic is pure) + IL-safety check, compiles the fork clean
4 REVIEW    Opus review (skeptic, or full panel for high-risk); verdict routes QUEUE vs PARK
5 MEASURE   human triggers batched in-game A/B run; accept/reject on real numbers
```

The old design had a separate GATE stage between review and measurement. It was cut: the mechanical gates (compile, unit test, IL verify, thread-safety, save-compat) already run in Stage 3, so combining them with the review verdict is a one-line routing decision, not a stage. Stage 4 owns that routing.

### Stage 1 — Survey

A Sonnet agent reads the cached decompiled game assemblies and Fast Track's own DebugMetrics output (per-system tick cost buckets), and produces a ranked list of opportunities: per-tick CPU cost, allocation churn, redundant work, methods whose Fast Track patch no longer resolves on the current build. Ranking is by measured or strongly-inferred hot-path cost so later stages do not waste effort on cold-path micro-opts.

Decompile via `ilspycmd` (installed at `~/.dotnet/tools/ilspycmd`). Game assemblies live at the machine's `OxygenNotIncluded_Data/Managed/` folder (see CLAUDE.md for the current absolute path). Decompiled output is cached under `harness/decompiled/` and gitignored.

### Stage 2 — Design

For each opportunity a Sonnet agent writes a candidate `.md` containing:
- What and where: target type/method, source file, the optimization.
- Why: the mechanism of the win (fewer allocations, cheaper algorithm, skipped redundant work).
- Predicted impact: qualitative plus a rough magnitude, tied to the hot-path evidence from Stage 1.
- Risk class: Prefix < Postfix < Transpiler/IL rewrite < threading/static-state. Drives review tier and gate strictness.
- Collision check: does this overlap Peter's in-progress Aquatic rewrites or duplicate an existing Fast Track feature. If yes, note it; net-new is preferred.

### Stage 3 — Build

A Sonnet agent writes the patch in the fork, plus a unit test when the logic is pure (see Verification), plus an IL-safety check for transpilers. It compiles the fork clean (MSBuild, the project's existing build command). The mechanical gates run here: compile, unit test, IL verify, thread-safety check, save-compat check. A patch that fails any of them never reaches Stage 4. Correctness *refutation* is not done here; that is the Skeptic's job in Stage 4, and doing it in both places was a duplicated pass.

### Stage 4 — Review (Opus) + routing

Review is tiered by risk class. The mechanical gates already passed in Stage 3, so a low-risk candidate has already survived a unit test and IL verification; it does not need a panel.

- Low-risk candidate (Prefix/Postfix, pure logic, passing unit test): a single skeptical Opus reviewer. It tries to refute (invalid IL, a race, an edge case the decompile hid; defaults to reject when uncertain) and returns the verdict. No panel, no chairman.
- High-risk candidate (transpiler, IL rewrite, threading, no possible unit test): a full panel that argues by opposing lenses so the agents actually disagree:
  - Skeptic (correctness): tries to refute. Defaults to reject when uncertain.
  - Performance advocate (value): is the predicted gain real and worth a scarce measurement slot? Does DebugMetrics show the path is hot, or is it cold-path polish?
  - Fast Track fit: does it match Peter's architecture and conventions? Collide with or duplicate his rewrites? Genuinely net-new?
  - Maintenance / risk: save-compat, mod-compat (`FastTrackCompat`), fragility on the next game update.

  The proposing agent gets to rebut, then a chairman agent (Opus) synthesizes. Reuse the `council` skill (five thinking styles, anonymous peer review, chairman synthesis) configured to Opus rather than building debate infra.

The verdict is ADVANCE, REVISE, or REJECT, with a value score, a confirmed risk class, and the reasoning. The full transcript and verdict are appended to the candidate `.md`. The verdict also routes the candidate, absorbing what used to be a separate gate stage:
- ADVANCE → QUEUE for measurement.
- REJECT → PARK with the recorded reason.

REVISE loops back to Stage 3 once; the author addresses the strongest objection and the candidate is re-reviewed. A second REVISE auto-PARKs the candidate to avoid infinite polish.

### Stage 5 — Measure (human checkpoint)

Queued candidates accumulate. The user triggers a batched in-game run per game launch: several independent, non-colliding candidates at once to amortize launch cost. The benchmark save is loaded with and without the candidate set; frame time, `GC.GetTotalMemory`, and collection counts are captured over N frames to `Player.log`. A Sonnet agent diffs the A/B logs and writes the result, plus the raw captures, into the candidate `.md`.

- Accepted (measured win, no regression) → merged into the fork.
- Rejected (no win, or a regression) → parked with the numbers attached to the candidate `.md`.

## Verification

The user's "unit / security / regression" goal, scoped to what is real for a game mod.

### Unit tests

Only for the pure island that runs outside the game: the Ryu float formatter, `MinMax`, data-structure logic, pure helpers, and IL-correctness assertions. A plain netstandard test project under `harness/tests/`. Game-coupled patches cannot be unit-tested; the spec states this rather than pretending otherwise.

### "Security" reframed to what actually bites

A local game mod has no network or untrusted-input attack surface. The real failure classes are:
- IL / transpiler correctness: verify emitted IL, no `InvalidProgramException` or native verifier fault.
- Thread-safety: a shared mutable static (the `CACHED_BUILDER` class) already caused a crash. Every candidate touching statics or worker threads gets a re-entrancy check.
- Supply chain: no new dependencies; game-DLL references stay pinned to the machine's Managed folder.

### Regression

- Save-compat: no `[SerializationConfig]` layout change without a deliberate flag. Changing a building/component ID or serialized layout can invalidate existing colonies.
- Behavior parity: at the Stage 5 checkpoint, confirm the colony still simulates as expected with the candidate applied.

Save-compat runs as a mechanical gate in Stage 3; behavior parity is confirmed at the Stage 5 checkpoint.

## Measurement

Both methods, as requested, with preference for extending Fast Track's existing foundation where it adds value.

- Extend Fast Track's DebugMetrics where useful. It already buckets per-system tick cost; that output feeds Stage 1's ranking so the survey is data-driven, not guesswork.
- A fixed late-game benchmark save plus a small logging patch that captures frame time, `GC.GetTotalMemory`, and collection counts over N frames to `Player.log`, plus a diff script. Same save, same seed, A/B with the candidate set on and off. This number decides accept/reject in Stage 5.

## Per-loop deliverables

- Updated fork with accepted optimizations merged.
- A candidate `.md` per optimization: idea → patch → debate transcript → verdict → measured result + raw A/B captures. Doubles as the PR-quality evidence package for upstream submission.

## Risks and mitigations

- Bad IL hard-crashes the game. Mitigation: IL verification and a unit/static safety gate before any in-game run; batched gated runs, never a blind loop.
- Agents propose plausible-but-wrong patches (Peter's stated concern). Mitigation: the adversarial Stage 4 debate plus mandatory A/B measurement; nothing lands on assertion alone.
- Fork diverges from Peter's parallel rewrite. Mitigation: Stage 2 collision check biases toward net-new, additive optimizations that merge cleanly.
- Game update breaks patch targets again. Mitigation: candidates record their target signatures so a future build diff can re-validate them quickly.

## Validated starting position (loop-1 seed)

An Opus audit of the fork (2026-06-30) validated the prior ponytail review and produced a concrete starting position, so loop 1 does not begin from a cold survey. These targets are consumed by the first loop and will go stale; they live here as a seed, not a maintained list.

Scan facts: the fork has 42 transpilers across 28 files and 171 name-only / `GetMethodSafe` / `TargetMethod()` lookups across 38 files. Those two sets are the fragility surface most likely to need rewrites on a game update.

Cleanup freebies (no review needed, trivial, do first):
- Delete `ExtensionMethods.cs:50-56` — redundant and dead `Append(this StringBuilder, StringBuilder)` (BCL chunk-copy overload is faster; no caller).
- Delete `BugFixPatches.cs:239-296` — `#if false` debug scaffolding, never compiled.

Top 5 ranked targets to point agents at first:
1. `GamePatches/GameUtilPatches.cs:184` — `GameUtil_CollectCellsBreadthFirst_Patch`. Hard compile break against Aquatic (method deleted, now `FloodFill` / `AcousticDisturbance+CellCollector`). Blocks the build today. Delete or retarget. Risk: Prefix-safe.
2. `UIPatches/FormatStringPatches.cs:42` — make `CACHED_BUILDER` `[ThreadStatic]`. Correctness fix that preserves the zero-alloc Ryu path. Note: the threadsafety doc concluded this alone does not resolve the Aquatic menu-load crash (a base-game parallel-template-loader race FT only exposes by timing). Risk: static/threading.
3. `PathPatches` `CachePaths` transpiler on `PathProber.Run` — highest-risk active transpiler; signature changed most in Aquatic (overloaded, 10-arg). Verify IL body. Risk: Transpiler/IL.
4. `GamePatches/BugFixPatches.cs:128` — geyser `CACHE` dictionary, unlocked shared static populated lazily during parallel load. Add a lock or eager-populate. Risk: static/threading.
5. `VisualPatches/` mesh renderers (gated `MeshRendererOptions`, currently OFF) — a large already-written optimized surface ships disabled. Verify its 4 transpilers (`FallingWater.Render`, `PrioritizableRenderer.RenderEveryTick`, `TerrainBG.LateUpdate`, `ConduitFlowMesh.End`) are IL-compatible, then enable and measure. Highest-leverage net perf without writing from scratch. Risk: Transpiler/IL.

Leave alone (already maximally optimized): `Ryu/`, `AsyncJobManager.cs`, `PathProbeJobManager.cs`, `MinMax.cs`, `ConcurrentHandleVector.cs`, `ThreadsafePartitionerLayer.cs`.

## Open implementation questions (for the plan, not blocking)

- Exact N (frame count) and which save serves as the canonical benchmark.
- Whether DebugMetrics needs new buckets, or its current output suffices for Stage 1 ranking.
- How candidates are batched in Stage 5 to guarantee non-collision (independent flags / disjoint target methods).
