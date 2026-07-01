# Fast Track Optimization Harness

Reference material and tooling behind the performance and thread-safety work in PR #712 (Fast Track on ONI Aquatic). This folder holds the design specs, the per-candidate investigation trail, the audits, the profiling captures, and the test and benchmark tooling. None of this folder ships in the mod. It is kept here so the reasoning and evidence behind each change can be reviewed alongside the PR.

## The loop

Each optimization went through five stages.

1. Survey. Scan the decompiled game assembly (`decompiled/`) and profiling captures for hot paths worth optimizing.
2. Design. Write a candidate file from `candidates/_TEMPLATE.md` stating the target, the mechanism of the win, and the predicted impact.
3. Build. Implement the Harmony patch, add a unit test where the code is testable outside the game, and run the static gates: compile, IL verify, thread-safety, save-compat.
4. Review. A skeptical pass on the candidate, on a stronger model, returning ADVANCE, REVISE, or REJECT.
5. Measure. A/B the change in game, capture Player.log, record the numbers in the candidate file, and resolve to ACCEPTED or PARKED.

## Layout

- `specs/` holds the harness design spec (`2026-06-30-fasttrack-optimization-harness-design.md`) and the earlier Aquatic-fix design (`2026-06-18-fasttrack-fix-design.md`).
- `plans/` holds the implementation plans: the harness bootstrap (`2026-06-30-fasttrack-optimization-harness-bootstrap.md`) and the earlier ambiguous-patch fix (`2026-06-18-fasttrack-ambiguous-patch-fix.md`).
- `candidates/` holds one Markdown file per optimization, `0001` through `0008`, each carrying its full trail from diagnosis to patch to review to measured result. `_TEMPLATE.md` is the blank form.
- `tests/` holds standalone unit-test projects for the pure, game-independent code: `RyuTests` (the Ryu float formatter) and `PathCacheTests` (the `PathCacheGeometry.CellInWindow` predicate).
- `benchmark/` holds `diff_bench.py`, which parses two `[FT-BENCH]` Player.log captures and reports the frame-time and GC deltas, along with its tests (`test_diff_bench.py`) and fixtures.
- `audit-threadsafety-efficiency.md` is the whole-codebase thread-safety and efficiency audit.
- `profiling-baseline.md` and `profiling-after-fix.md` are the hot-path profiles before and after the path-cache work.
- `longsession-log-audit.md` is the two-hour session analysis that resolved the late-game GC hitch as Boehm fragmentation rather than a leak.
- `decompiled/` is a local ilspycmd cache of the game assembly. Gitignored; regenerate locally.
- `baseline-build.log` and `*.bench.log` are local build and benchmark output. Gitignored.

## Diagnostic instrumentation in the mod

Some measurement tooling lives in the Fast Track source itself, all off or DEBUG-gated by default. The `BenchmarkLog` option and `Metrics/BenchmarkLogPatch.cs` provide the `[FT-BENCH]` frame-time and GC sampler. Two added `DebugMetrics` counters split path-cache misses into invalidated, expired, and navigator-moved. A few `harmony.Profile` targets in the existing DEBUG profiler block break specific per-selection costs out of the `Game.Update` bucket. The PR description covers these, and they lift out cleanly if not wanted.
