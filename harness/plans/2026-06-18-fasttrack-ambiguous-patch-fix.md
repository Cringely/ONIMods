# FastTrack AmbiguousMatchException Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Identify and fix the `AmbiguousMatchException` that prevents FastTrack from running on ONI Aquatic (build 737790), so the game reaches and stays at the main menu with FT loaded.

**Architecture:** Two parallel analysis agents produce a source map and an assembly overload list; a synthesis step cross-references them to find the ambiguous patch target; a fix agent edits the exact call to use an explicit overload signature; MSBuild rebuilds the DLL; user verifies via game launch. If the cross-reference produces no match, fall back to Approach B (binary disable with diagnostic logging).

**Tech Stack:** C# / .NET Framework 4.8, Harmony 2.4.2.0, PLib (PeterHan), MSBuild (VS 2026 Community), PowerShell reflection for assembly inspection.

## Global Constraints

- Build tool: MSBuild at `C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe` — never `dotnet build`
- FT project: `E:\projects\ONI\_onimods_upstream\FastTrack\FastTrack.csproj`
- FT source root: `E:\projects\ONI\_onimods_upstream\FastTrack\`
- Built DLL lands at: `E:\projects\ONI\_onimods_upstream\FastTrack\bin\Mergedown\Release\netstandard2.1\FastTrack.dll`
- Deploy target: `C:\Users\jcgam\Documents\Klei\OxygenNotIncluded\mods\local\FastTrack\FastTrack.dll`
- Game assemblies: `F:\SteamLibrary\steamapps\common\OxygenNotIncluded\OxygenNotIncluded_Data\Managed\`
- Reference crash log: `E:\projects\ONI\logs\crash-ft-loaded.log` (line 830 — `AmbiguousMatchException`)
- Fix pattern: use `GetMethodSafe` / `AccessTools.Method` with explicit parameter types — never name-only resolution on overloaded methods
- Game must be fully closed before deploying a new DLL (DLL is locked while game runs)
- Grep tool returns no matches on FT source due to UTF-8 BOM — use Bash `grep` for FT source files

---

## Task 1: Extract all name-only Patch() calls from FT source

**Files:**
- Read: all `.cs` files under `E:\projects\ONI\_onimods_upstream\FastTrack\`
- Produce: printed list of `(SourceFile:Line, TargetType, MethodName)` tuples

**Interfaces:**
- Produces: list consumed by Task 3 synthesis

- [ ] **Step 1: Run grep for PLib name-only Patch() calls**

PLib's `ExtensionMethods.Patch()` signature: `Patch(Harmony, Type, string methodName, ...)`.
The calls look like `harmony.Patch(typeof(X), "Name",` or `harmony.Patch(typeof(X), nameof(X.Name),`.

```bash
grep -rn "harmony\.Patch(typeof" "E:/projects/ONI/_onimods_upstream/FastTrack/" --include="*.cs" \
  | grep -v "GetMethodSafe\|GetMethod(\|\.Patch(typeof.*AccessTools\|//\|\.bak"
```

- [ ] **Step 2: Capture and record results**

Expected output: several lines like:
```
GamePatches/ElectricalPatches.cs:43:  harmony.Patch(typeof(CircuitManager), nameof(CircuitManager.Refresh), ...
ConduitPatches/ConduitFlowVisualizerPatches.cs:93:  harmony.Patch(targetType, nameof(ConduitFlowVisualizer.Render), ...
```

Record the full list — every line is a candidate. If the list is empty, the name-only calls use a variable for the type (not `typeof`); run this additional grep:

```bash
grep -rn "harmony\.Patch(" "E:/projects/ONI/_onimods_upstream/FastTrack/" --include="*.cs" \
  | grep -v "GetMethodSafe\|GetMethod(\|AccessTools\|HarmonyMethod\|//\|\.bak" \
  | grep '"'
```

---

## Task 2: Enumerate overloaded methods in Assembly-CSharp.dll

**Files:**
- Read: `F:\SteamLibrary\steamapps\common\OxygenNotIncluded\OxygenNotIncluded_Data\Managed\Assembly-CSharp.dll`
- Produce: printed list of `TypeFullName::MethodName (N overloads)` where N > 1

**Interfaces:**
- Produces: list consumed by Task 3 synthesis

- [ ] **Step 1: Load the assembly and enumerate overloaded methods**

Run in PowerShell:

```powershell
$dll = "F:\SteamLibrary\steamapps\common\OxygenNotIncluded\OxygenNotIncluded_Data\Managed\Assembly-CSharp.dll"
$asm = [Reflection.Assembly]::LoadFile($dll)
$flags = [Reflection.BindingFlags]"Public,NonPublic,Instance,Static"
$results = $asm.GetTypes() | ForEach-Object {
    $t = $_
    $t.GetMethods($flags) |
        Group-Object Name |
        Where-Object { $_.Count -gt 1 } |
        ForEach-Object { "$($t.FullName)::$($_.Name) ($($_.Count) overloads)" }
}
$results | Sort-Object | Out-File "E:\projects\ONI\logs\assembly-overloads.txt"
Write-Host "Done. $($results.Count) overloaded method names found."
```

Expected: file written, count printed (will be in the hundreds — that's normal for a large game assembly).

- [ ] **Step 2: Spot-check the output**

```powershell
Get-Content "E:\projects\ONI\logs\assembly-overloads.txt" | Select-Object -First 20
```

Should show lines like `CircuitManager::SomeMethod (2 overloads)`. If the file is empty, the assembly failed to load — check the path.

---

## Task 3: Cross-reference and identify the ambiguous patch target(s)

**Files:**
- Read: results from Task 1 (grep output) and `E:\projects\ONI\logs\assembly-overloads.txt` (Task 2)
- Produce: a definitive list of `(SourceFile, LineNumber, TargetType, MethodName)` tuples that need fixing

**Interfaces:**
- Consumes: Task 1 candidate list, Task 2 overload list
- Produces: exact fix locations consumed by Task 4

- [ ] **Step 1: Extract type+method pairs from Task 1 grep output**

For each grep hit from Task 1, parse out the class name and method name. Example: `harmony.Patch(typeof(CircuitManager), nameof(CircuitManager.Refresh),` → `CircuitManager::Refresh`.

- [ ] **Step 2: Search the overload file for each candidate**

```powershell
$candidates = @(
  # Fill from Task 1 results — one entry per hit, format: "TypeName::MethodName"
  "CircuitManager::Refresh",
  "ConduitFlowVisualizer::Render"
  # ... all Task 1 entries
)
$overloads = Get-Content "E:\projects\ONI\logs\assembly-overloads.txt"
$candidates | Where-Object {
    $c = $_
    $overloads | Where-Object { $_ -match [regex]::Escape($c) }
} | ForEach-Object { "MATCH: $_" }
```

Expected: one or more `MATCH:` lines. Each match is an ambiguous patch target.

- [ ] **Step 3: If no matches found — pivot to Approach B**

If the cross-reference produces zero matches, the ambiguity may be in a variable-type call or a nested class. Skip to Task 7 (Approach B fallback). Otherwise continue to Task 4.

- [ ] **Step 4: For each match, read the source line and surrounding context**

Open the source file at the identified line. Read ±10 lines to understand what the patch does and what parameter types the target method should have.

```bash
grep -n "harmony\.Patch" "E:/projects/ONI/_onimods_upstream/FastTrack/<FILE>.cs" -A 5 -B 2
```

Identify the specific overload FT intends to patch (usually obvious from the patch body — it references parameters by name/type).

---

## Task 4: Apply the explicit overload fix

**Files:**
- Modify: whichever `.cs` file(s) Task 3 identified
- No new files

**Interfaces:**
- Consumes: exact `(file, line, TargetType, MethodName, intendedParamTypes)` from Task 3

- [ ] **Step 1: Determine the intended parameter types**

For each match from Task 3, read the patch method body (the prefix/postfix) to infer which overload FT targets. The patch's `__instance`, `__result`, and named parameters tell you the expected signature.

If the overloads in Assembly-CSharp.dll differ only by one parameter, check which the patch body uses.

- [ ] **Step 2: Replace the name-only Patch() call**

Before (name-only — ambiguous):
```csharp
harmony.Patch(typeof(SomeType), nameof(SomeType.MethodName),
    prefix: new HarmonyMethod(typeof(SomePatch), nameof(SomePatch.Prefix)));
```

After (explicit parameter types — unambiguous):
```csharp
var target = typeof(SomeType).GetMethodSafe(nameof(SomeType.MethodName), false,
    typeof(ParamType1), typeof(ParamType2));
harmony.Patch(target,
    prefix: new HarmonyMethod(typeof(SomePatch), nameof(SomePatch.Prefix)));
```

`GetMethodSafe` is a PLib extension already imported in every FT file. It returns `null` if not found (no throw), so add a null guard if the method isn't guaranteed to exist:
```csharp
if (target != null)
    harmony.Patch(target, prefix: new HarmonyMethod(typeof(SomePatch), nameof(SomePatch.Prefix)));
```

- [ ] **Step 3: Verify the edit compiles (quick sanity check)**

```bash
grep -n "GetMethodSafe\|harmony\.Patch" "E:/projects/ONI/_onimods_upstream/FastTrack/<MODIFIED_FILE>.cs" | head -20
```

Confirm the old name-only call is gone and the new `GetMethodSafe` call is present.

---

## Task 5: Rebuild FastTrack and deploy

**Files:**
- Read: `E:\projects\ONI\_onimods_upstream\FastTrack\FastTrack.csproj`
- Write: `C:\Users\jcgam\Documents\Klei\OxygenNotIncluded\mods\local\FastTrack\FastTrack.dll`

**Interfaces:**
- Consumes: fix from Task 4
- Produces: deployed DLL consumed by game launch in Task 6

- [ ] **Step 1: Confirm game is closed**

The DLL is locked while the game runs. Verify no `OxygenNotIncluded.exe` process is running before copying.

- [ ] **Step 2: Rebuild FastTrack**

```powershell
& "C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe" `
  "E:\projects\ONI\_onimods_upstream\FastTrack\FastTrack.csproj" `
  -t:Rebuild -p:Configuration=Release
```

Expected: `Build succeeded.` with `0 Error(s)`. If errors, fix them before proceeding — do not deploy a broken build.

- [ ] **Step 3: Verify build output exists**

```powershell
Get-Item "E:\projects\ONI\_onimods_upstream\FastTrack\bin\Mergedown\Release\netstandard2.1\FastTrack.dll" |
    Select-Object Name, LastWriteTime, Length
```

Confirm `LastWriteTime` is within the last minute.

- [ ] **Step 4: Deploy DLL**

```powershell
Copy-Item `
  "E:\projects\ONI\_onimods_upstream\FastTrack\bin\Mergedown\Release\netstandard2.1\FastTrack.dll" `
  "C:\Users\jcgam\Documents\Klei\OxygenNotIncluded\mods\local\FastTrack\FastTrack.dll" `
  -Force
```

---

## Task 6: Verify via game launch

**Files:**
- Read: `C:\Users\jcgam\AppData\LocalLow\Klei\Oxygen Not Included\Player.log` (after launch)
- Write: `E:\projects\ONI\logs\verify-<attempt>.log` (saved copy)

- [ ] **Step 1: Launch the game**

Launch ONI normally. Wait at least 15 seconds after the main menu appears. If the game stays up, that's a win. If it closes/crashes, note the behaviour.

- [ ] **Step 2: Save and inspect the log**

```powershell
Copy-Item "C:\Users\jcgam\AppData\LocalLow\Klei\Oxygen Not Included\Player.log" `
  "E:\projects\ONI\logs\verify-attempt-1.log"
```

```bash
grep -n "Loading MOD dll: FastTrack\|-- MAIN MENU --\|AmbiguousMatchException\|Restarting\|Cleanup current backend" \
  "E:/projects/ONI/logs/verify-attempt-1.log"
```

**Success criteria — all three must be true:**
1. `Loading MOD dll: FastTrack.dll` appears
2. `-- MAIN MENU --` appears after it
3. No `Cleanup current backend` within 10 log-lines of `-- MAIN MENU --`

**If the `AmbiguousMatchException` is gone but a new error appears:** read the surrounding log lines, identify the new crash type, and open a new investigation cycle starting from Task 3 with the updated crash info.

**If the `AmbiguousMatchException` still appears:** the fix targeted the wrong call. Re-examine Task 3 — look for additional matches or variable-type calls missed by the grep pattern.

- [ ] **Step 3: On success — update memory**

Update `C:\Users\jcgam\.claude\projects\E--projects-ONI\memory\fasttrack-018-aquatic-crash.md` to record:
- The specific method that was ambiguous
- The fix applied (explicit parameter types used)
- Date confirmed working on build 737790

---

## Task 7 (Approach B Fallback): Diagnostic binary disable

**Only run this task if Task 3 Step 3 triggers (zero cross-reference matches).**

**Files:**
- Modify: each FT source file containing `harmony.Patch(` calls, one at a time

**Goal:** Wrap each file's `Patch()` cluster in a `try/catch` that logs the throwing method name, then binary-search via game launches to identify the exact file.

- [ ] **Step 1: Add diagnostic wrapper to first candidate file**

Priority order: `ConduitPatches/ConduitFlowVisualizerPatches.cs`, then `GamePatches/ElectricalPatches.cs`, then `GamePatches/NoDiseasePatches.cs`, then `FastTrackMod.cs`.

For the first file, wrap all `harmony.Patch(typeof(...), "name",` calls in:
```csharp
try {
    harmony.Patch(typeof(SomeType), nameof(SomeType.Method), prefix: ...);
} catch (System.Reflection.AmbiguousMatchException ex) {
    Debug.LogError($"[FT-DIAG] AmbiguousMatch in {nameof(SomeType)}.{nameof(SomeType.Method)}: {ex.Message}");
}
```

- [ ] **Step 2: Rebuild and deploy (same as Task 5)**

- [ ] **Step 3: Launch game, save log, check for `[FT-DIAG]`**

```bash
grep -n "FT-DIAG\|AmbiguousMatchException" "E:/projects/ONI/logs/verify-attempt-b1.log"
```

If `[FT-DIAG]` appears with a method name → that is the ambiguous call. Remove the try/catch diagnostic wrapper, apply the `GetMethodSafe` fix from Task 4, and continue to Task 5.

If no `[FT-DIAG]` but `AmbiguousMatchException` still appears → the throw is in a different file. Move to the next file in the priority order and repeat from Step 1.

- [ ] **Step 4: Once the method is identified, apply Task 4's fix**

Remove all diagnostic wrappers before committing. Apply the `GetMethodSafe` explicit-type fix to the identified call only.
