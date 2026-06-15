# Code Review — alibre-assimp-addon

- **Date:** 2026-06-15
- **Branch:** `review/2026-06-15-code-review` (branched from `the-tool-store` @ `415406c` — "cleanup push")
- **Reviewer:** Claude (Opus 4.8)
- **Scope:** Full repository review (single C# Alibre add-on host + Assimp conversion service; project/solution/manifest tooling; .github community files)

---

## 1. Summary

This is an Alibre Design add-on that adds an "Assimp Inside Alibre Design" ribbon menu for
importing and exporting many 3D mesh formats (OBJ, FBX, glTF, DAE, PLY, 3DS, STL, …) via the
[AssimpNet](https://www.nuget.org/packages/AssimpNet) wrapper around the native Assimp library.
The entire implementation lives in one file, [source/AlibreAddOn.cs](source/AlibreAddOn.cs)
(~544 lines): the COM-facing host (`AlibreAddOn` / `AddOnRibbon`), a hand-built menu tree
(`MenuManager`), the command handlers (`MenuItem`), and a thin Assimp wrapper (`AssimpService`).

The COM/menu plumbing is clean and reasonable, and is noticeably simpler than the sibling
shapes add-on (no installer scripts, no IronPython layer, no committed binaries). However, the
**central feature premise is broken**: the add-on's main value proposition is "convert mesh
formats to STEP and import into Alibre," but Assimp has **no STEP/B-Rep exporter** — every code
path that asks Assimp to export `"stp"` will fail at runtime. There is also a **version-pinned
Alibre install path**, an **unpinned/beta Assimp dependency with no native-binary deployment
story**, and a layer of **copy-paste cruft** ("stltostp" strings, a phantom `config.json`, an
`AlibreAddOn - Copy\**` exclusion, the file listed as both `<None>` and compiled).

**Overall:** Tidy scaffolding wrapped around a feature that cannot work as written. The Critical
item (STEP export) is a design problem, not a typo — it needs a real conversion strategy before
the import/export menu is anything but error dialogs. The repo is otherwise small and clean.

### Findings by severity

| Severity | Count |
|----------|-------|
| Critical | 1 |
| High     | 3 |
| Medium   | 4 |
| Low / Nit| 6 |

---

## 2. Critical

### C-1. Assimp cannot export STEP — the primary import/convert feature fails at runtime
**Files:** [AlibreAddOn.cs:140](source/AlibreAddOn.cs), [AlibreAddOn.cs:491-494](source/AlibreAddOn.cs), [AlibreAddOn.cs:412-414](source/AlibreAddOn.cs)

The whole import pipeline converts the chosen mesh to STEP, then hands the STEP file to Alibre:

```csharp
// ImportFile
string tempStepPath = Path.Combine(Path.GetTempPath(), $"{...}_temp.stp");
if (assimpService.ExportFile(scene, tempStepPath, "stp")) { ... alibreRoot.ImportSTEPFile(tempStepPath); ... }

// ConvertStlToStep -> ConvertFile(..., "stp")
public bool ConvertStlToStep(string stlPath, string stepPath) => ConvertFile(stlPath, stepPath, "stp");
```

Assimp is a *mesh* library. Its export format IDs are things like `stl`, `obj`, `ply`, `gltf2`,
`collada`, `fbx`, `3mf`, `assbin`, `assjson`, `x`, `pbrt` — there is **no `stp`/`step` B-Rep
exporter** (STEP is a boundary-representation/solid format, which Assimp does not produce).
`AssimpContext.ExportFile(scene, path, "stp")` will return `false` (unknown format id) or throw,
which `AssimpService.ExportFile`/`ConvertFile` swallow and return `false`
([AlibreAddOn.cs:484-488](source/AlibreAddOn.cs), [515-525](source/AlibreAddOn.cs)). The result:
every `Import 3dm/glTF/OBJ/FBX/DAE/PLY/3DS` menu item and the `Convert File to STEP` tool will
hit the "Failed to convert … to STEP" / "output STEP file was not created" error branch.
Meanwhile the menu *also* offers `Export ... STEP` ([line 412](source/AlibreAddOn.cs)) with the
same impossible `"stp"` id.

This is the headline feature of the add-on and it cannot work via Assimp alone.

**Fix:** decide on a real bridge to Alibre. Options, roughly in order of effort:
- Import meshes as meshes. Alibre can import STL directly — convert the Assimp scene to STL
  (a format Assimp *does* export) and call the appropriate Alibre STL-import API instead of
  `ImportSTEPFile`. This makes import actually work for all mesh inputs.
- If true solid/STEP output is required, mesh→B-Rep conversion needs a different engine
  (e.g. OpenCascade), not Assimp. Remove the STEP export menu entries until then.
- At minimum, remove/disable the STEP export menu item and stop advertising STEP conversion so
  the UI doesn't promise something that always errors.

---

## 3. High

### H-1. Hard-coded, version-pinned Alibre install path in `HintPath`s
**File:** [alibre-assimp-addon.csproj:31,35,39](source/alibre-assimp-addon.csproj)

```xml
<HintPath>C:\Program Files\Alibre Design 28.1.0.28223\Program\alibre-api.dll</HintPath>
<HintPath>C:\Program Files\Alibre Design 28.1.0.28223\Program\AlibreAddOn.dll</HintPath>
<HintPath>C:\Program Files\Alibre Design 28.1.0.28223\Program\AlibreX.dll</HintPath>
```

The build is pinned to one exact Alibre build (`28.1.0.28223`) at the default Program Files
location. It will not restore/build on any machine with a different Alibre version, a non-default
install path, or a localized Program Files. (Note this differs from the sibling repo, which
pinned `28.1.1.28227` — they are already out of sync.) Resolve the install root via an MSBuild
property/env var (e.g. `$(AlibreInstallDir)`) or the registry key the Alibre installer writes,
with the hard-coded path only as a last-resort fallback, and reference the DLLs relative to that.

### H-2. Beta Assimp dependency with no native-library deployment, and STEP-aside, no proof it loads
**File:** [alibre-assimp-addon.csproj:27](source/alibre-assimp-addon.csproj)

```xml
<PackageReference Include="AssimpNet" Version="5.0.0-beta1" />
```

Two problems:
- It pins a **pre-release** (`5.0.0-beta1`) of AssimpNet. Pre-releases can vanish or change API;
  prefer the latest stable `4.1.0` (or whatever stable build is validated against this code) and
  pin it deliberately.
- AssimpNet is a managed wrapper over a **native** `assimp` DLL (32/64-bit `Assimp*.dll` shipped
  in the package's `runtimes`/`build` folder). For a COM add-on that Alibre loads in-process, the
  native binary must end up next to the managed assembly *and* match the process bitness
  (the project targets `x64`, [csproj:12,15](source/alibre-assimp-addon.csproj) — good, but the
  native deployment still has to be verified). There is no `.adc`/packaging step that copies the
  Assimp native DLL into the add-on output, so the first call into `AssimpContext` may throw a
  `DllNotFoundException` at runtime. Confirm the native lib is emitted to the output directory and
  installed alongside the DLL.

### H-3. `AddOnInvoke` shows a stray `"stltostp.AddOnInvoke"` dialog; `Aboutmd` advertises the wrong tool
**Files:** [AlibreAddOn.cs:36-39](source/AlibreAddOn.cs), [AlibreAddOn.cs:222-226](source/AlibreAddOn.cs)

```csharp
public static void AddOnInvoke(...) { MessageBox.Show("stltostp.AddOnInvoke"); }
...
public IAlibreAddOnCommand? Aboutmd(IADSession session)
{
    MessageBox.Show("About stltostp - Version X.Y.Z\n(Details about the tool or link can be placed here)");
    ...
}
```

`AddOnInvoke` is a host-callable entry point; if Alibre ever invokes it (e.g. add-on activation)
the user sees a debug popup referencing a *different* add-on ("stltostp"). It should either do
nothing or perform the real activation, not pop a leftover debug `MessageBox`. `Aboutmd` is dead
copy-paste (the real About is `AboutAssimpAddon`, [line 228](source/AlibreAddOn.cs)) with a
placeholder "Version X.Y.Z" and the wrong product name — delete it.

---

## 4. Medium

### M-1. `.adc` manifest is terse / inconsistent with the in-code menu naming
**File:** [alibre-assimp-addon.adc:6-7](source/alibre-assimp-addon.adc)

```xml
<Menu text="assimp"/>
<Description>assimp</Description>
```

The ribbon root in code is `"Assimp Inside Alibre Design"` ([AlibreAddOn.cs:353](source/AlibreAddOn.cs))
and the About box calls it the same, but the manifest's `Menu`/`Description` are just `"assimp"`.
Users see one name in the add-on manager and another in the ribbon. Make the manifest text and
description match the product name for a coherent install/UX.

### M-2. Phantom `config.json` and `AlibreAddOn - Copy\**` cruft in the project file
**File:** [alibre-assimp-addon.csproj:18-21,47-49](source/alibre-assimp-addon.csproj)

```xml
<Compile Remove="AlibreAddOn - Copy\**" />   <!-- ... 3 more Remove lines -->
...
<None Update="config.json"><CopyToOutputDirectory>Always</CopyToOutputDirectory></None>
```

There is no `AlibreAddOn - Copy\` directory and no `config.json` in the repo. The `<None Update>`
(as opposed to `Include`) is a no-op when the item doesn't exist, so it won't break the build, but
together with the `- Copy\**` excludes these are clear leftovers from cloning another project and
signal the `.csproj` was never cleaned. Remove both. (The Assimp add-on does not appear to need a
`config.json` at all.)

### M-3. `<RootNamespace>` does not match the actual namespace
**File:** [alibre-assimp-addon.csproj:6](source/alibre-assimp-addon.csproj) vs [AlibreAddOn.cs:14](source/AlibreAddOn.cs)

```xml
<RootNamespace>AssimpInsideAlibreDesignAddon</RootNamespace>
```

The code's namespace is `AlibreAddOnAssembly`. `RootNamespace` only affects default namespaces for
*new* files and some resource naming, so it's not fatal, but the mismatch is confusing and is more
copy-paste residue (this repo also still carries `Microsoft.VisualBasic` and unused `using`s, see
L-1). Align it with `AlibreAddOnAssembly` or drop it.

### M-4. Export handlers are unimplemented stubs but presented as 18 working menu items
**Files:** [AlibreAddOn.cs:189-221](source/AlibreAddOn.cs), [AlibreAddOn.cs:381-429](source/AlibreAddOn.cs)

`ExportFile` only shows a "Note: Full export functionality … will be implemented" `MessageBox` and
never exports anything — yet `BuildMenus` wires up ~18 distinct export menu entries (3dm, glTF,
DAE, STL, OBJ, PLY, X, 3DS, JSON, ASSBIN, STEP, PBRT, glTF 1.0/2.0, 3MF, FBX). Every one of them
opens a save dialog and then does nothing. This is a large, misleading surface of dead UI. Either
implement export (Alibre-export-to-STL → Assimp-export-to-target is feasible for mesh formats) or
collapse the Export group to the formats you actually support and mark the rest clearly as TODO.

---

## 5. Low / Nits

### L-1. Unused / superfluous imports
[AlibreAddOn.cs:1-13](source/AlibreAddOn.cs): `Microsoft.VisualBasic` (line 4) and
`System.Windows.Forms` aliasing aside, `Microsoft.VisualBasic` is never used; `System.Windows`
(line 10) is only used for the `MessageBox` alias. Trim the unused usings.

### L-2. Class named identically to a namespace (`AlibreAddOn`)
[AlibreAddOn.cs:1,16](source/AlibreAddOn.cs): `public static class AlibreAddOn` lives next to
`using AlibreAddOn;` (the SDK namespace). It works here because the static class is only referenced
unqualified within its own namespace, but it's an avoidable ambiguity — rename to e.g.
`AddOnEntryPoint`. (Same smell flagged in the sibling repo.)

### L-3. `InvokeCommand` does an unguarded session lookup
[AlibreAddOn.cs:64-68](source/AlibreAddOn.cs): `_AlibreRoot.Sessions.Item(sessionIdentifier)` is
passed straight into the command with no null/exception guard; an unknown identifier can throw
across the COM boundary. Guard the lookup and return `null` on miss.

### L-4. `SubMenuItems` can return `null` to the host
[AlibreAddOn.cs:75-79](source/AlibreAddOn.cs): returns `null` (not an empty `Array`) for an unknown
id. Hosts generally prefer a non-null collection; return `Array.Empty<int>()` to be safe.

### L-5. Icon path `@"Icons\logo.ico"` is set on every menu item but never used by the host
[AlibreAddOn.cs:360-438](source/AlibreAddOn.cs): every `MenuItem` is given `@"Icons\logo.ico"`, and
`MenuIcon` ([line 83](source/AlibreAddOn.cs)) returns it, but it's a single generic logo for all
~30 items and a relative path that resolves against the process working directory, not the add-on
folder — so it likely won't load. Either resolve the icon path relative to the assembly location or
drop the per-item icon noise.

### L-6. Errors surface only as modal dialogs, never logged
Throughout `AssimpService` and the command handlers, failures are either `Debug.WriteLine`
([line 486](source/AlibreAddOn.cs) etc., invisible in Release) or a `MessageBox`. There's no
persistent log file, so field failures (e.g. the C-1 STEP failures, or a native-DLL load error)
can't be diagnosed without screenshots. Add a lightweight log to the add-on folder.

---

## 6. What looks good

- Single-file, dependency-light, easy to read; far less cruft and no committed build artifacts or
  installer scripts compared with the sibling shapes add-on (`.gitignore` is comprehensive and the
  working tree is clean).
- The COM/ribbon plumbing (`AddOnRibbon` implementing `IAlibreAddOn`, `MenuManager` building a
  registered id→item dictionary, `RootMenuItem`/`HasSubMenus`/`SubMenuItems`) is coherent and uses
  null-conditional lookups throughout.
- `AssimpService` correctly implements `IDisposable` and is consistently used in `using` blocks, so
  the native `AssimpContext` is released.
- Lifecycle (`AddOnLoad`/`AddOnUnload`) nulls the statics on unload.
- `x64` platform target is set for both Debug and Release — appropriate for an in-process 64-bit
  Alibre add-on loading a native Assimp binary.
- The `.github/` community files (README, CoC, contributing, issue/PR templates) and the
  standardized `source/ submodules/ documentation/` layout are in place.

---

## 7. Recommended fix order

1. **C-1** — decide the real mesh→Alibre path (import as STL, or drop STEP). Without this the
   import/convert feature only produces error dialogs. (blocking)
2. **H-2** — pin a stable AssimpNet and verify the native Assimp DLL deploys next to the add-on
   (x64), or nothing in `AssimpService` runs at all.
3. **H-1** — unpin the Alibre `HintPath`s so the project builds off this one machine.
4. **H-3 / M-4** — remove the leftover "stltostp" dialogs/`Aboutmd`, and either implement or prune
   the 18 stub export menu items so the UI stops promising features that no-op.
5. **M-1** — align the `.adc` menu/description with the real product name.
6. Sweep the **M-2 / M-3 / L-*** cleanups (phantom `config.json`, `- Copy\**` excludes, wrong
   `RootNamespace`, unused usings, icon path, logging) while touching the project.
