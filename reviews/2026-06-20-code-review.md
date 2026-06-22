# alibre-assimp-addon (Assimp Inside Alibre Design Add-on) — Code Review (Correctness)

**Date:** 2026-06-20
**Scope:** Second-opinion review, code only (correctness bugs).

**Summary: 3 bugs — 1 High, 2 Medium**

## High

- **`AlibreAddOn.cs:63` — `RootMenuItem => _menuManager.GetRootMenuItem().Id`**
  If the `MenuManager` constructor threw in the `AddOnRibbon` constructor (line 56), the exception is caught and swallowed (lines 58-61), leaving `_menuManager` as `null`. Any subsequent call to `RootMenuItem`, `InvokeCommand`, `HasSubMenus`, `SubMenuItems`, `MenuItemText`, etc. will throw a `NullReferenceException` because none of them null-check `_menuManager`. The swallowed-exception pattern hides the real failure and converts it into a later, harder-to-diagnose crash when the host calls into the interface.

## Medium

- **`AlibreAddOn.cs:138-153` — temp STEP file leaked on the Alibre import failure path.**
  The temp file written by `assimpService.ExportFile` to `tempStepPath` is only deleted inside the `if (alibreRoot != null)` success branch (lines 152-153). If `alibreRoot` is null, or if `ImportSTEPFile` throws (caught at line 161), the temp `.stp` file is never deleted, leaking a file in the temp directory on every failed import. There is no `finally` cleanup.

- **`AlibreAddOn.cs:196` — `DefaultExt = filterExtensions.Split(';')[0].Replace("*", "")` produces an invalid default extension.**
  For an input like `"*.gltf"`, this yields `".gltf"` (leading dot), but `SaveFileDialog.DefaultExt` expects the extension without the leading dot (e.g. `"gltf"`); the leading `.` causes the default extension to be applied incorrectly (a filename can end up as `name..gltf` or have the extension not appended as intended). It should strip the `*.` prefix, not just the `*`.

## Notes (not flagged as bugs but worth awareness)

- `ConvertFile`/`ExportFile`/`ImportFile` in `AssimpService` rely on `_assimpContext.ImportFile` returning null on failure; AssimpNet typically throws `AssimpException` rather than returning null, but those calls are wrapped in try/catch, so behavior is acceptable (the `scene == null` checks are merely redundant, not incorrect).
- The `RunCmd` path writes the STEP output next to the input file (`Path.ChangeExtension`), which could overwrite an existing `.stp` — this is by design, not a clear correctness bug.

The High finding (unguarded `_menuManager` after swallowed constructor exception) is the one that can cause a real crash in the host application.

---

## Fixes applied — 2026-06-20

- **[High] `AlibreAddOn.cs`** — removed the swallowing try/catch in the `AddOnRibbon` constructor so a `MenuManager` failure propagates; the object can no longer end up half-initialized with a null `_menuManager`.
- **[Medium] `AlibreAddOn.cs`** — temp STEP file at `tempStepPath` is now deleted in a `finally` (guarded by its own try/catch), so it is cleaned up on the success, null-root, and exception paths.
- **[Medium] `AlibreAddOn.cs`** — `DefaultExt` now strips the `*.` prefix (`.Replace("*.", "").Replace("*", "")`) so the value has no leading dot.

*Caveat: changes applied to source; not verified by build.*
