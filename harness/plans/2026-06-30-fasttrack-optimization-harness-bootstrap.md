# Fast Track Optimization Harness — Bootstrap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the agent-driven optimization harness (test island, benchmark tooling, decompile cache, candidate ledger) on a versioned working branch of the Fast Track fork, then drive the first optimization candidate end-to-end through the loop to prove the pipeline works.

**Architecture:** Work happens in the git-tracked monorepo clone `_onimods_upstream/` (the only buildable tree, carrying the `cringely` and `origin` remotes) on a fresh branch off `cringely/aquatic-compat`. The harness lives in `_onimods_upstream/harness/` so its evidence trail is versioned with the fork. Two real code artifacts get TDD (the netstandard test island for Fast Track's pure Ryu float formatter, and the A/B log-diff script); the rest are setup and a documented loop procedure with command-level verification gates.

**Tech Stack:** C# / .NET (Fast Track targets netstandard2.1, SDK-style csproj with `Platforms=Vanilla;Mergedown`, `UsePublicized=true`), Harmony 2.x, `dotnet` CLI, `ilspycmd` for decompilation, Python 3 for the diff script, xUnit (net8.0) for the test island.

## Global Constraints

Every task's requirements implicitly include these.

- Working fork root: `E:\projects\ONI\_onimods_upstream\` (git, remotes `cringely` = github.com/Cringely/ONIMods, `origin` = github.com/peterhaneve/ONIMods).
- Fast Track project: `_onimods_upstream\FastTrack\FastTrack.csproj`. Target framework netstandard2.1. SDK-style. `Platforms=Vanilla;Mergedown`.
- Build command (deployable merged DLL): `dotnet build _onimods_upstream/FastTrack/FastTrack.csproj -c Debug -p:Platform=Mergedown`.
- Game assembly references: `F:\SteamLibrary\steamapps\common\OxygenNotIncluded\OxygenNotIncluded_Data\Managed\`.
- Deploy is automatic: a Debug build with `DistributeMod=true` copies the merged DLL to `$(ModFolder)` = `…\Klei\OxygenNotIncluded\mods\dev\FastTrack\` (via `Directory.Build.targets` `CopyArtifactsToInstallFolder`). Do NOT also hand-copy to `mods\local` — duplicate copies load duplicate Harmony patches. Before any in-game test, confirm exactly one FastTrack loads: disable any Steam Workshop copy and any `mods\local\FastTrack` (Workshop loads over dev per project memory). Game log: `%USERPROFILE%\AppData\LocalLow\Klei\Oxygen Not Included\Player.log`.
- Commit identity: the user's GitHub identity only (Cringely). Never add a `Co-Authored-By` trailer or any Claude attribution. Plain commit messages.
- License: Fast Track is MIT © Peter Han. Preserve the license and attribution headers; do not relicense.
- No new runtime dependencies in the Fast Track DLL (supply-chain rule). Harness-only dev deps (xUnit) are fine since they never ship in the mod.
- Mod option flags gate Harmony patches at load via `Prepare()`; a flag change needs a game restart, it does not un-apply live.
- Save-sensitive: do not change any `[SerializationConfig]` layout or a building/component ID without an explicit flag and a call-out.
- A bad IL/transpiler patch hard-crashes the game with no crash handler. Never run an unverified IL change in-game; gate it behind the static checks first.

## File Structure

Created or modified by this plan:

- `_onimods_upstream/harness/README.md` — what the harness is, how the loop runs (the spec in brief).
- `_onimods_upstream/harness/.gitignore` — ignores `decompiled/` (large cache).
- `_onimods_upstream/harness/candidates/_TEMPLATE.md` — the candidate ledger template.
- `_onimods_upstream/harness/candidates/0001-*.md` — first real candidate (Task 7).
- `_onimods_upstream/harness/decompiled/` — gitignored ilspycmd output cache.
- `_onimods_upstream/harness/benchmark/diff_bench.py` — A/B Player.log diff script.
- `_onimods_upstream/harness/benchmark/fixtures/` — sample log captures for the diff-script test.
- `_onimods_upstream/harness/benchmark/test_diff_bench.py` — unit test for the diff script.
- `_onimods_upstream/harness/tests/RyuTests/RyuTests.csproj` — net8.0 xUnit project linking Fast Track's pure Ryu sources.
- `_onimods_upstream/harness/tests/RyuTests/RyuFloatTests.cs` — known-value Ryu tests.
- `_onimods_upstream/FastTrack/FastTrackOptions.cs` — add a `BenchmarkLog` option (Task 5).
- `_onimods_upstream/FastTrack/Metrics/BenchmarkLogPatch.cs` — new frame-time/GC sampler patch (Task 5).

---

### Task 1: Working branch + baseline build triage

Establishes the versioned home and finds out what actually breaks against the current game, so later fix-tasks target real errors, not assumptions.

**Files:**
- Modify: git state of `_onimods_upstream/` (new branch).
- Create: `_onimods_upstream/harness/baseline-build.log` (triage capture, gitignored later).

**Interfaces:**
- Produces: branch `fasttrack-aquatic-optim` based on `cringely/aquatic-compat`; a captured build-error list that Task 7 and any follow-up fix work consume.

- [ ] **Step 1: Fetch the user's fork and inspect the base branch**

```bash
cd /e/projects/ONI/_onimods_upstream
git fetch cringely
git log --oneline cringely/aquatic-compat -5
```
Expected: the `aquatic-compat` branch history prints (this is the PR #712 work). If the branch is missing, stop and ask the user which branch holds their Aquatic work.

- [ ] **Step 2: Create the working branch off it**

```bash
git switch -c fasttrack-aquatic-optim cringely/aquatic-compat
git status
```
Expected: "On branch fasttrack-aquatic-optim", clean tree.

- [ ] **Step 3: Build Fast Track against the current game and capture the result**

```bash
dotnet build FastTrack/FastTrack.csproj -c Debug -p:Platform=Mergedown 2>&1 | tee harness/baseline-build.log
```
Expected: either a clean build, or a list of compile errors. Either way `harness/baseline-build.log` now records the true baseline. Do not fix anything yet — this task only establishes ground truth.

- [ ] **Step 4: Record the verdict in the harness README scratch and commit the branch point**

If the build succeeded, note "baseline builds clean" in the commit body. If it failed, list the erroring files. Scope the add — do NOT `git add -A`: the working tree carries an untracked stray `Docs/superpowers/` folder inside the fork that must not be swept into the branch. Then:
```bash
git add harness/baseline-build.log
git commit -m "Start fasttrack-aquatic-optim branch from aquatic-compat baseline"
```
Expected: a commit exists capturing only the triage log (it gets gitignored in Task 2 and untracked there). If `harness/` does not exist yet, skip the add and make an empty-tree marker commit instead: `git commit --allow-empty -m "Start fasttrack-aquatic-optim branch from aquatic-compat baseline"`.

---

### Task 2: Harness scaffolding + candidate template

**Files:**
- Create: `_onimods_upstream/harness/.gitignore`
- Create: `_onimods_upstream/harness/README.md`
- Create: `_onimods_upstream/harness/candidates/_TEMPLATE.md`
- Create: `_onimods_upstream/harness/decompiled/.gitkeep`

**Interfaces:**
- Produces: the `_TEMPLATE.md` schema every candidate copies; the `decompiled/` ignore rule.

- [ ] **Step 1: Write the gitignore**

Create `_onimods_upstream/harness/.gitignore`:
```
decompiled/
baseline-build.log
*.bench.log
```

- [ ] **Step 2: Remove the triage log from tracking (it is ignored now)**

```bash
cd /e/projects/ONI/_onimods_upstream
git rm --cached harness/baseline-build.log 2>/dev/null || true
```
Expected: the log stays on disk but is no longer tracked.

- [ ] **Step 3: Write the candidate template**

Create `_onimods_upstream/harness/candidates/_TEMPLATE.md`:
```markdown
# Candidate NNNN: <short title>

- Status: DESIGN | BUILT | REVIEWED | QUEUED | ACCEPTED | PARKED
- Target: <Type.Method> — <file:line>
- Risk class: Prefix | Postfix | Transpiler/IL | static/threading
- Gating flag: <FastTrackOptions flag, or "none">
- Collision with Peter's rewrites: yes/no — <note>

## What and why
<the optimization and the mechanism of the win>

## Predicted impact
<qualitative + rough magnitude, tied to a hot-path source>

## Patch
<diff or file/line summary of the change>

## Unit test
<name + path, or "not unit-testable: game-coupled">

## Static gates (Stage 3)
- Compiles: <y/n>
- Unit test: <pass/fail/na>
- IL verify: <pass/fail/na>
- Thread-safety check: <note/na>
- Save-compat: <note/na>

## Review (Stage 4)
<single skeptic verdict, or full council transcript> → ADVANCE | REVISE | REJECT

## Measurement (Stage 5)
<A/B numbers + raw Player.log capture excerpts>

## Outcome
ACCEPTED (merged) | PARKED (reason)
```

- [ ] **Step 4: Write a one-screen README and keep the decompiled dir**

Create `_onimods_upstream/harness/README.md` summarizing the 5-stage loop (Survey → Design → Build → Review → Measure), the model split (Opus for review, Sonnet elsewhere), and a pointer to the design spec at `E:\projects\ONI\docs\superpowers\specs\2026-06-30-fasttrack-optimization-harness-design.md`. Create an empty `_onimods_upstream/harness/decompiled/.gitkeep`.

- [ ] **Step 5: Commit**

```bash
git add harness/.gitignore harness/README.md harness/candidates/_TEMPLATE.md harness/decompiled/.gitkeep
git commit -m "Add optimization harness scaffolding and candidate template"
```

---

### Task 3: Decompile cache

Caches the game assembly decompilation so Survey-stage agents read source, not raw IL.

**Files:**
- Create: `_onimods_upstream/harness/decompiled/Assembly-CSharp/` (ilspycmd output, gitignored)

**Interfaces:**
- Produces: a readable C# tree of the game assembly under `harness/decompiled/` that Stage 1 agents grep.

- [ ] **Step 1: Confirm ilspycmd is available**

```bash
~/.dotnet/tools/ilspycmd --version
```
Expected: a version prints. If not, install: `dotnet tool install -g ilspycmd`.

- [ ] **Step 2: Decompile the main game assembly into the cache**

```bash
cd /e/projects/ONI/_onimods_upstream
~/.dotnet/tools/ilspycmd -p -o harness/decompiled/Assembly-CSharp \
  "F:/SteamLibrary/steamapps/common/OxygenNotIncluded/OxygenNotIncluded_Data/Managed/Assembly-CSharp.dll"
```
Expected: a project tree of `.cs` files appears under `harness/decompiled/Assembly-CSharp/`.

- [ ] **Step 3: Verify the cache is usable and ignored**

```bash
ls harness/decompiled/Assembly-CSharp/*.csproj
git status --porcelain harness/decompiled
```
Expected: a `.csproj` exists; `git status` shows nothing under `decompiled/` (the `.gitignore` from Task 2 covers it). No commit — this is a local cache.

---

### Task 4: Ryu test island (TDD)

Proves the pure, game-decoupled island is unit-testable outside the game. Ryu is Fast Track's zero-alloc float→string core; it has no Unity dependency, so it compiles standalone into a test assembly.

**Files:**
- Create: `_onimods_upstream/harness/tests/RyuTests/RyuTests.csproj`
- Create: `_onimods_upstream/harness/tests/RyuTests/RyuFloatTests.cs`

**Interfaces:**
- Consumes: Fast Track's Ryu sources at `_onimods_upstream/FastTrack/Ryu/*.cs`. VERIFIED by review: namespace is `Ryu` (not `PeterHan.FastTrack.Ryu`); sources are pure (`System`/`System.Globalization`/`System.Text` only, zero UnityEngine); they use `unsafe` bit-casts (`RyuUtils.cs:753,757`), so the test project must allow unsafe. The only public formatter is `Ryu.RyuFormat.ToString(StringBuilder result, double value, int precision, RyuFormatOptions options = RoundtripMode, IFormatProvider provider = null)` — it returns void and writes into a caller-supplied `StringBuilder`.
- Produces: a green `dotnet test` over the Ryu formatter.

- [ ] **Step 1: Confirm the entry point signature in source**

```bash
cd /e/projects/ONI/_onimods_upstream
grep -n "public static.*ToString" FastTrack/Ryu/RyuFormat.cs
```
Expected: confirms `Ryu.RyuFormat.ToString(StringBuilder, double, int, RyuFormatOptions = ..., IFormatProvider = null)`. If the signature differs from the review's finding, adjust the wrapper in Step 2 to match.

- [ ] **Step 2: Write the failing test**

Create `_onimods_upstream/harness/tests/RyuTests/RyuFloatTests.cs`. The wrapper adapts the void `StringBuilder`-out method to a `double -> string` shape. `precision=0` requests Ryu's shortest round-trip form:
```csharp
using System;
using System.Text;
using Xunit;
using Ryu;

public class RyuFloatTests {
    private static string Fmt(double d) {
        var sb = new StringBuilder();
        RyuFormat.ToString(sb, d, 0);   // precision 0 = shortest round-trip
        return sb.ToString();
    }

    [Theory]
    [InlineData(0.0, "0")]
    [InlineData(1.0, "1")]
    [InlineData(0.5, "0.5")]
    [InlineData(-2.25, "-2.25")]
    public void FormatsKnownValues(double input, string expected) {
        Assert.Equal(expected, Fmt(input));
    }

    // Round-trip: every formatted double must parse back to itself.
    [Theory]
    [InlineData(3.14159265358979)]
    [InlineData(123456.789)]
    [InlineData(1e-7)]
    [InlineData(1e21)]
    public void RoundTrips(double input) {
        Assert.Equal(input, double.Parse(Fmt(input)));
    }
}
```
Note: the known-value literals are best-guesses; Ryu's canonical output may differ (for example it may emit `0` vs `0.0`, or scientific notation for `1e21`). Step 5 corrects the literals to whatever Ryu actually returns. The round-trip tests are format-agnostic and are the real correctness check.

- [ ] **Step 3: Write the test project that links the pure Ryu sources**

Create `_onimods_upstream/harness/tests/RyuTests/RyuTests.csproj`. `AllowUnsafeBlocks` is REQUIRED (Ryu's bit-casts are `unsafe`):
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>disable</Nullable>
    <IsPackable>false</IsPackable>
    <LangVersion>latest</LangVersion>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>
  <ItemGroup>
    <!-- Link Fast Track's pure float-formatting sources directly (no game refs needed). -->
    <Compile Include="..\..\..\FastTrack\Ryu\*.cs" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageReference Include="xunit" Version="2.9.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
  </ItemGroup>
</Project>
```

- [ ] **Step 4: Run the test, expect a failure**

```bash
cd /e/projects/ONI/_onimods_upstream/harness/tests/RyuTests
dotnet test
```
Expected: FAIL — most likely a known-value assertion mismatch (the round-trip tests should already pass). If instead a missing-type compile error appears from a linked Ryu file referencing a pure helper outside `Ryu/`, add that one pure source file to the `<Compile Include>` list (never a file pulling a UnityEngine/game reference).

- [ ] **Step 5: Correct the known-value literals and make it pass**

Run the test, read the actual Ryu output from the assertion failures, and update the `[InlineData]` expected strings to match Ryu's canonical form (do not force a format — Ryu's shortest round-trip output is correct by definition). The round-trip theories should not need changes.
```bash
dotnet test
```
Expected: PASS, all theories green.

- [ ] **Step 6: Commit**

```bash
cd /e/projects/ONI/_onimods_upstream
git add harness/tests/RyuTests/RyuTests.csproj harness/tests/RyuTests/RyuFloatTests.cs
git commit -m "Add Ryu float-formatter unit test island"
```

---

### Task 5: Benchmark logging patch + flag

A small always-available sampler that writes frame time, GC memory, and collection counts over a fixed window to `Player.log`, behind a default-off flag. This is the A/B data source for Stage 5. Game-coupled, so verification is build + in-game, not a unit test.

**Files:**
- Modify: `_onimods_upstream/FastTrack/FastTrackOptions.cs` (add `BenchmarkLog` flag)
- Create: `_onimods_upstream/FastTrack/Metrics/BenchmarkLogPatch.cs`

**Interfaces:**
- Consumes: `FastTrackOptions.Instance.BenchmarkLog` flag.
- Produces: lines in `Player.log` of the form `[FT-BENCH] frame=<n> ms=<f> gcMB=<f> gen0=<n>` that Task 6's diff script parses.

- [ ] **Step 1: Add the option flag**

In `FastTrackOptions.cs`, follow the existing option-property pattern (a `[Option]`-attributed bool auto-property defaulting to false). Add:
```csharp
[Option("Benchmark Logging", "Logs frame time and GC to Player.log over a fixed window for A/B measurement. Off for normal play.")]
[JsonProperty]
public bool BenchmarkLog { get; set; }
```
Confirm the default is false by leaving it unset in the constructor (match how the other off-by-default flags are initialized). Convention note: existing options use localized `STRINGS.UI.FRONTEND.FASTTRACK.*` keys plus a category arg (see `FastTrackOptions.cs:59`); the inline-literal 2-arg form above compiles (PLib falls back to the literal in a default category) and is acceptable for an internal dev flag. Leaving it unlocalized is intentional.

- [ ] **Step 2: Write the sampler patch**

Create `_onimods_upstream/FastTrack/Metrics/BenchmarkLogPatch.cs`. Patch a per-frame hook (`Game.Update`) with a `Prepare()` gate on the flag, sampling `Time.deltaTime`, `GC.GetTotalMemory(false)`, and `GC.CollectionCount(0)` each frame, and emitting one summary line per frame (or per 60-frame bucket to keep the log small):
```csharp
using HarmonyLib;
using UnityEngine;
using System;
using System.Globalization;

namespace PeterHan.FastTrack.Metrics {
    [HarmonyPatch(typeof(Game), nameof(Game.Update))]
    public static class BenchmarkLogPatch {
        private static int frame;
        public static bool Prepare() => FastTrackOptions.Instance.BenchmarkLog;
        public static void Postfix() {
            long mem = GC.GetTotalMemory(false);
            int gen0 = GC.CollectionCount(0);
            // InvariantCulture: the diff script regex expects a '.' decimal, not a locale comma.
            // unscaledDeltaTime: ONI scales Time.deltaTime by timeScale (0 when paused, rescaled by
            // game speed), which would zero or distort the measurement — use wall-clock instead.
            Debug.Log(string.Format(CultureInfo.InvariantCulture,
                "[FT-BENCH] frame={0} ms={1:F3} gcMB={2:F2} gen0={3}",
                frame++, Time.unscaledDeltaTime * 1000.0, mem / 1048576.0, gen0));
        }
    }
}
```
Notes: `Game.Update` is already a live patch target in the fork (`SharedPatches.cs:49`, `BackgroundConduitUpdater.cs:239`), so it is a valid, safe Postfix target — no decompile lookup needed. The patch logs every frame; the per-frame `Debug.Log` overhead is present equally in both A/B captures so the delta stays meaningful, but absolute `gcMB`/`gen0` values are inflated vs normal play — note this in usage. Both culture and unscaled-time corrections were found by the Task 5 review.

- [ ] **Step 3: Build with the patch**

```bash
cd /e/projects/ONI/_onimods_upstream
dotnet build FastTrack/FastTrack.csproj -c Debug -p:Platform=Mergedown
```
Expected: clean build (0 errors). If `Game.Update` does not resolve, correct the target from the decompile cache.

- [ ] **Step 4: Smoke-test in-game — HUMAN CHECKPOINT (subagents hard-pause here)**

A subagent cannot launch or play the game; it must stop and hand back to the user. The Debug build already auto-deployed the DLL to `mods\dev\FastTrack\` (do not hand-copy). The user: confirms no Workshop/`mods\local` FastTrack is enabled (so only one loads), launches ONI with `BenchmarkLog` set true in the Fast Track config, loads any save, lets it run ~10 seconds, quits. Then verify:
```bash
grep -c "FT-BENCH" "$USERPROFILE/AppData/LocalLow/Klei/Oxygen Not Included/Player.log"
```
Expected: a nonzero count of `[FT-BENCH]` lines, confirming the sampler fires. The subagent runner marks this task complete only after the user reports the count.

- [ ] **Step 5: Commit**

```bash
git add FastTrack/FastTrackOptions.cs FastTrack/Metrics/BenchmarkLogPatch.cs
git commit -m "Add benchmark logging patch behind default-off BenchmarkLog flag"
```

---

### Task 6: A/B log-diff script (TDD)

Parses two `Player.log` captures (with/without a candidate) and reports the deltas that decide accept/reject.

**Files:**
- Create: `_onimods_upstream/harness/benchmark/diff_bench.py`
- Create: `_onimods_upstream/harness/benchmark/fixtures/a.log`, `fixtures/b.log`
- Create: `_onimods_upstream/harness/benchmark/test_diff_bench.py`

**Interfaces:**
- Consumes: `[FT-BENCH] frame=<n> ms=<f> gcMB=<f> gen0=<n>` lines from Task 5.
- Produces: `summarize(path) -> {frames, mean_ms, p95_ms, gc_growth_mb, gen0}` and a CLI `diff_bench.py A.log B.log` printing the deltas.

- [ ] **Step 1: Write fixtures**

Create `_onimods_upstream/harness/benchmark/fixtures/a.log` (the "before"):
```
[FT-BENCH] frame=0 ms=20.000 gcMB=100.00 gen0=10
[FT-BENCH] frame=1 ms=20.000 gcMB=101.00 gen0=10
[FT-BENCH] frame=2 ms=20.000 gcMB=102.00 gen0=11
[FT-BENCH] frame=3 ms=20.000 gcMB=103.00 gen0=11
```
Create `fixtures/b.log` (the "after" — faster, less GC):
```
[FT-BENCH] frame=0 ms=16.000 gcMB=100.00 gen0=10
[FT-BENCH] frame=1 ms=16.000 gcMB=100.50 gen0=10
[FT-BENCH] frame=2 ms=16.000 gcMB=101.00 gen0=10
[FT-BENCH] frame=3 ms=16.000 gcMB=101.50 gen0=10
```

- [ ] **Step 2: Write the failing test**

Create `_onimods_upstream/harness/benchmark/test_diff_bench.py`. Uses stdlib `unittest` — pytest is not installed and the script has no deps, so we add none:
```python
import os, unittest
from diff_bench import summarize

HERE = os.path.dirname(__file__)

class TestDiffBench(unittest.TestCase):
    def test_summarize_mean_and_gc(self):
        s = summarize(os.path.join(HERE, "fixtures", "a.log"))
        self.assertEqual(s["frames"], 4)
        self.assertEqual(s["mean_ms"], 20.0)
        self.assertEqual(s["gc_growth_mb"], 3.0)   # 103 - 100
        self.assertEqual(s["gen0"], 1)             # 11 - 10

    def test_b_is_faster(self):
        a = summarize(os.path.join(HERE, "fixtures", "a.log"))
        b = summarize(os.path.join(HERE, "fixtures", "b.log"))
        self.assertLess(b["mean_ms"], a["mean_ms"])
        self.assertLess(b["gc_growth_mb"], a["gc_growth_mb"])

if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 3: Run the test, expect failure**

```bash
cd /e/projects/ONI/_onimods_upstream/harness/benchmark
python -m unittest test_diff_bench -v
```
Expected: FAIL — `ModuleNotFoundError: No module named 'diff_bench'`.

- [ ] **Step 4: Write the script**

Create `_onimods_upstream/harness/benchmark/diff_bench.py`:
```python
#!/usr/bin/env python3
"""Diff two Fast Track benchmark Player.log captures. ponytail: regex + statistics, no deps."""
import re, sys, statistics

LINE = re.compile(r"\[FT-BENCH\] frame=(\d+) ms=([\d.]+) gcMB=([\d.]+) gen0=(\d+)")

def summarize(path):
    ms, gc, gen0 = [], [], []
    with open(path, encoding="utf-8", errors="ignore") as f:
        for line in f:
            m = LINE.search(line)
            if m:
                ms.append(float(m.group(2)))
                gc.append(float(m.group(3)))
                gen0.append(int(m.group(4)))
    if not ms:
        raise ValueError(f"no [FT-BENCH] lines in {path}")
    ms_sorted = sorted(ms)
    return {
        "frames": len(ms),
        "mean_ms": round(statistics.fmean(ms), 4),
        "p95_ms": round(ms_sorted[min(len(ms_sorted) - 1, int(0.95 * len(ms_sorted)))], 4),
        "gc_growth_mb": round(gc[-1] - gc[0], 4),
        "gen0": gen0[-1] - gen0[0],
    }

def main(a, b):
    sa, sb = summarize(a), summarize(b)
    print(f"{'metric':<14}{'A (before)':>14}{'B (after)':>14}{'delta':>14}")
    for k in ("mean_ms", "p95_ms", "gc_growth_mb", "gen0"):
        d = sb[k] - sa[k]
        print(f"{k:<14}{sa[k]:>14}{sb[k]:>14}{d:>+14}")
    verdict = "FASTER" if sb["mean_ms"] < sa["mean_ms"] else "SLOWER/EQUAL"
    print(f"\nverdict: {verdict} (mean frame time)")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        sys.exit("usage: diff_bench.py BEFORE.log AFTER.log")
    main(sys.argv[1], sys.argv[2])
```

- [ ] **Step 5: Run the test, expect pass**

```bash
python -m unittest test_diff_bench -v
```
Expected: PASS, both tests green (`Ran 2 tests ... OK`).

- [ ] **Step 6: Commit**

```bash
cd /e/projects/ONI/_onimods_upstream
git add harness/benchmark/diff_bench.py harness/benchmark/test_diff_bench.py harness/benchmark/fixtures/a.log harness/benchmark/fixtures/b.log
git commit -m "Add A/B benchmark log-diff script with tests"
```

---

### Task 7: First candidate end-to-end (pipeline proof)

Drives one real optimization through all five stages to prove the loop produces evidence, not vibes. Use the `CACHED_BUILDER` `[ThreadStatic]` fix (`FormatStringPatches.cs:42`): it is concrete in our baseline, it is a real correctness defect, and it preserves the headline zero-alloc path. This is a Stage-1-through-5 walkthrough, not a TDD code task; its deliverable is a complete candidate `.md`.

**Files:**
- Create: `_onimods_upstream/harness/candidates/0001-cachedbuilder-threadstatic.md`
- Modify: `_onimods_upstream/FastTrack/UIPatches/FormatStringPatches.cs:42` (and any sibling shared builders the review flags)

**Interfaces:**
- Consumes: the `_TEMPLATE.md` schema (Task 2), the build command (Global Constraints), the benchmark patch + diff script (Tasks 5, 6).
- Produces: candidate `0001` in QUEUED or ACCEPTED state with a filled-in evidence trail.

- [ ] **Step 1: Design — open the candidate file from the template**

Copy `_TEMPLATE.md` to `harness/candidates/0001-cachedbuilder-threadstatic.md`. Fill Target (`FormatStringPatches.CACHED_BUILDER`, `FastTrack/UIPatches/FormatStringPatches.cs:42`), Risk class (`static/threading`). Write the What/why honestly: this is one shared mutable `StringBuilder`. Making it thread-local removes a real re-entrancy hazard on the off chance these format paths are touched off the main thread, while preserving the zero-alloc win on the hot main-thread path. Do NOT overclaim: the review confirmed there are ~14 sibling shared-static builders across the UIPatches files (other `CACHED_BUILDER`s in `FormatStringPatches.3.cs:249,439,799`, plus `BUFFER` in `DescriptorAllocPatches.cs:45`, `OUTER_BUILDER`/`PART_BUILDER`/`ACTION_BUFFER`/`HOTKEY_BUFFER` in `FormatStringPatches.3.cs`), and the design spec itself states this change does NOT fix the Aquatic menu-load crash (that is a base-game parallel-template-loader race). State in the candidate that this fixes line 42 only and lists the sibling set as tracked follow-up. The real purpose of candidate 0001 is to prove the pipeline end-to-end on a small, safe, concrete change; the perf delta is expected to be near-zero and that is an acceptable ACCEPT for a correctness/hygiene fix.

- [ ] **Step 2: Build — apply the fix**

In `FormatStringPatches.cs:42`, change the shared static to a thread-static with lazy per-thread init. Replace:
```csharp
private static readonly StringBuilder CACHED_BUILDER = new StringBuilder(32);
```
with:
```csharp
[ThreadStatic]
private static StringBuilder cachedBuilder;
private static StringBuilder CACHED_BUILDER => cachedBuilder ??= new StringBuilder(32);
```
Check whether call sites mutate it expecting a shared instance across threads (they should not — each use does `Clear().Append(...).ToString()`); uses at lines 614 and 663 take a local `var text = CACHED_BUILDER;` then build and stringify, which is exactly the per-thread-safe pattern. `??=` compiles fine (the project defaults to C# 8 on netstandard2.1). Scope is line 42 only this candidate; the sibling builders are deliberately left for separate candidates so each gets its own evidence trail rather than bundling an unverified sweep.

- [ ] **Step 3: Build — static gates**

```bash
cd /e/projects/ONI/_onimods_upstream
dotnet build FastTrack/FastTrack.csproj -c Debug -p:Platform=Mergedown
```
Expected: 0 errors. Record in the candidate's Static gates section: Compiles y; Unit test na (game-coupled UI formatting); IL verify na (no transpiler); Thread-safety: now per-thread, no shared mutable static; Save-compat: no serialized layout change.

- [ ] **Step 4: Review — dispatch the Stage 4 reviewer (Opus)**

This is a `static/threading` change, so it is not the trivial tier. Dispatch the high-risk review: use the `council` skill configured to Opus, or at minimum a single Opus skeptic plus a thread-safety lens, on the diff. Feed it the patch and the candidate file. Paste the verdict (ADVANCE/REVISE/REJECT) and reasoning into the candidate's Review section. If REVISE, address the strongest objection once and re-review; second REVISE auto-PARKs.

- [ ] **Step 5: Measure — A/B in-game — HUMAN CHECKPOINT (subagents hard-pause here)**

A subagent cannot run the game; it stops and hands back. The user, with the benchmark patch from Task 5, captures two runs on the same save: baseline (`git stash` the one-line change) and patched. Use a UI-heavy scenario (open info panels / hover tooltips, where the format path is hot). Save each `Player.log` as `before.bench.log` / `after.bench.log`, then either party runs:
```bash
python harness/benchmark/diff_bench.py before.bench.log after.bench.log
```
Expected: no regression (mean frame time equal-or-better). The perf delta on this fix is expected near-zero; the correctness/hygiene win is the value, which is an acceptable ACCEPT. Paste the numbers into the candidate's Measurement section. The runner marks this task complete only after the user provides the two captures.

- [ ] **Step 6: Outcome — accept and commit**

Set the candidate Outcome to ACCEPTED. Commit the fix and its evidence together. Keep the message scoped — it fixes one builder, not the whole class of them:
```bash
git add FastTrack/UIPatches/FormatStringPatches.cs harness/candidates/0001-cachedbuilder-threadstatic.md
git commit -m "Make FormatStringPatches.cs:42 CACHED_BUILDER thread-local (1 of ~14 shared builders; pipeline-proof candidate 0001)"
```

- [ ] **Step 7: Push the branch**

```bash
git push -u cringely fasttrack-aquatic-optim
```
Expected: the branch is on the user's fork, ready for further loop iterations or an upstream PR.

---

## Self-Review

**Spec coverage:**
- Repo layout / fork home → Tasks 1, 2 (adjusted from the spec's `FastTrack/` to the real buildable clone `_onimods_upstream/`, per the source-of-truth decision).
- 5-stage loop → Task 7 walks all five; Tasks 2–6 build the machinery each stage needs.
- Survey stage decompile substrate → Task 3.
- Build/test pure island → Task 4.
- Measurement (DebugMetrics extension + A/B benchmark) → Tasks 5, 6. Note: this plan adds a dedicated lightweight sampler rather than extending DebugMetrics; the spec allowed both and preferred extending where it adds value. Extending DebugMetrics is deferred — the standalone sampler is simpler for clean A/B and does not require the `Metrics` subsystem to be enabled. Flagged as a deliberate simplification.
- Opus review / Sonnet split → Task 7 Step 4 (review on Opus); all other stages are Sonnet/mechanical.
- Candidate `.md` evidence ledger → Task 2 template, Task 7 instance.
- Verification (unit / IL / thread-safety / save-compat / regression) → Task 4 (unit island), Task 7 static gates + measurement.
- Seed targets → Task 7 uses seed #2 (`CACHED_BUILDER`); seed #1 (compile break) was re-scoped to Task 1's live triage since it is absent from the chosen baseline.

**Placeholder scan:** The Ryu `EntryPoint` in Task 4 is an intentional TDD placeholder resolved within the same task (Steps 1, 5) once the real public method name is read from source — it cannot be hard-coded because the exact signature must be confirmed against the baseline. No other placeholders.

**Type consistency:** `summarize()` keys (`frames`, `mean_ms`, `p95_ms`, `gc_growth_mb`, `gen0`) match between the test (Task 6 Step 2) and the script (Step 4). The `[FT-BENCH]` log format is identical between the producer (Task 5 Step 2) and the parser regex (Task 6 Step 4). The `CACHED_BUILDER` line content matches the grounding grep.

**Deliberate simplifications (ponytail):** standalone sampler over DebugMetrics extension (noted above); single Opus skeptic acceptable for the low tier though Task 7's target is high-tier so it gets the panel; harness lives in-repo rather than a separate tooling repo to keep the evidence trail versioned with the fork.
