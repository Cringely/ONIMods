# Fast Track Fix Design — 2026-06-18

## Problem

Fast Track 0.18.0.0 (locally built from PeterHan master) crashes ONI Aquatic (build 737790) with a hard exit ~0.3 seconds after reaching the main menu.

**Root cause confirmed by crash log (`logs/crash-ft-loaded.log`):**

```
[ERROR] [PLib/FastTrack] System.Reflection.AmbiguousMatchException Ambiguous match found.
  at System.Type.GetMethod(string name, BindingFlags bindingAttr)
  at PeterHan.PLib.Core.ExtensionMethods.Patch(Harmony, Type, string, HarmonyMethod, HarmonyMethod)
```

PLib's `Patch()` extension calls `Type.GetMethod(name, flags)` with a method name only (no parameter types). The Aquatic DLC added a new overload to some method FT patches this way, making resolution ambiguous. The ambiguous call throws, the critical patch is skipped, and the game hard-crashes during main menu initialization.

**What was previously investigated (no longer the primary issue):**
The earlier thread-safety crashes (geyser CACHE race, `ThreadAbortException` in template pre-load) were patched in `_onimods_upstream`. Those fixes are still in place and correct. The current crash is a separate, newer issue introduced by the Aquatic DLC adding method overloads.

## Approach A: Source Audit + Assembly Inspection (Primary)

### Agent 1 — FT source map

Reads every `.cs` file under `E:/projects/ONI/_onimods_upstream/FastTrack/`.

Extracts every call matching the pattern:
```
harmony.Patch(typeof(SomeType), "MethodName", ...)
```
or equivalently:
```
harmony.Patch(typeof(SomeType), nameof(SomeType.MethodName), ...)
```
where the second argument is a plain string (not a `GetMethodSafe` call).

Output: list of `(FullTypeName, MethodName, SourceFile, LineNumber)` tuples.

### Agent 2 — Assembly overload finder

Loads `F:/SteamLibrary/steamapps/common/OxygenNotIncluded/OxygenNotIncluded_Data/Managed/Assembly-CSharp.dll` via PowerShell reflection and enumerates all types. For each type, groups public methods by name and identifies any name that has more than one overload.

Output: list of `(FullTypeName, MethodName, OverloadCount)` tuples where `OverloadCount > 1`.

### Synthesis

Cross-reference Agent 1 and Agent 2 output. The intersection is the ambiguous patch target(s).

### Fix agent

For each matched `(Type, MethodName)` tuple:
- Open the source file at the identified line
- Replace the name-only `harmony.Patch(typeof(T), "Name", ...)` call with one that resolves the specific overload using explicit parameter types:
  ```csharp
  harmony.Patch(typeof(T).GetMethodSafe("Name", false, typeof(Param1), typeof(Param2)),
      prefix: ..., postfix: ...);
  ```
- Minimal change — only the ambiguous resolution, nothing else

## Approach B: Instrumented Binary Disable (Fallback)

If Approach A does not produce a match (agents can't resolve from source + assembly alone):

Add explicit `try/catch` with `Debug.Log` around each `Patch()` call cluster, grouped by source file. Rebuild and deploy. One game launch per cluster identifies which file contains the ambiguous call. Once the file is known, the method is known.

Binary search order (start with files most likely to patch Aquatic-new APIs):
1. `ConduitPatches/ConduitFlowVisualizerPatches.cs`
2. `GamePatches/ElectricalPatches.cs`
3. `GamePatches/NoDiseasePatches.cs`
4. `FastTrackMod.cs`
5. Remaining files alphabetically

## Rebuild + Deploy

```
"C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe"
  "E:\projects\ONI\_onimods_upstream\FastTrack\FastTrack.csproj"
  -t:Rebuild -p:Configuration=Release
```

Output DLL: `E:/projects/ONI/_onimods_upstream/FastTrack/bin/Mergedown/Release/netstandard2.1/FastTrack.dll`

Deploy to: `C:/Users/jcgam/Documents/Klei/OxygenNotIncluded/mods/local/FastTrack/FastTrack.dll`

## Verification

Launch ONI. In `Player.log`, check for:
1. `Loading MOD dll: FastTrack.dll` — DLL in memory
2. `-- MAIN MENU --` — game reached menu
3. No shutdown sequence within 10 seconds of the menu line

Success = all three. If the game exits within 10 seconds of the menu line, there is a second crash point to investigate.

## Loop Exit Conditions

- **Win:** game survives 10 seconds at main menu with FT loaded
- **Dead-end:** all identified ambiguous calls fixed, game still crashes → update `fasttrack-018-aquatic-crash` memory, re-park FT in `_FastTrack.DISABLED_TEST`, wait for Klei/PeterHan

## Files

- Crash log: `E:/projects/ONI/logs/crash-ft-loaded.log` (line 830 — `AmbiguousMatchException`)
- FT source: `E:/projects/ONI/_onimods_upstream/FastTrack/`
- Game assemblies: `F:/SteamLibrary/steamapps/common/OxygenNotIncluded/OxygenNotIncluded_Data/Managed/`
- Deploy target: `C:/Users/jcgam/Documents/Klei/OxygenNotIncluded/mods/local/FastTrack/`
