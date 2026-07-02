# Assimp Inside Alibre Design

An Alibre Design add-on that imports common 3D mesh formats into Alibre using the Assimp (Open Asset Import) library.

The add-on loads a mesh with AssimpNet, exports it to a temporary STEP file, and brings that STEP into the active Alibre session through `IADRoot.ImportSTEPFile`. It adds a top-level "Assimp Inside Alibre Design" menu with Import, Export, and Tools groups. Import and the Convert-to-STEP tool work; the Export menu entries are stubs that display a placeholder notice.

It targets Alibre Design 29.0.0.29060, builds against .NET Framework 4.8.1 (x64), and is written in C#.

## Table Of Contents

- [What Is Here](#what-is-here)
- [Official Alibre Resources](#official-alibre-resources)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Installation](#installation)
- [Usage](#usage)
- [Key Files](#key-files)
- [Key Folders](#key-folders)
- [Notes](#notes)
- [License](#license)

## What Is Here

- A single-project C# add-on (`source/`) that registers the "Assimp Inside Alibre Design" menu.
- Import commands for 3DM, glTF (`.gltf`/`.glb`), OBJ, FBX, DAE (Collada), PLY, and 3DS. Each loads the file with Assimp, converts it to a temporary STEP file, and imports it into the current session.
- A "Convert File to STEP" tool that converts a chosen model (`.stl`, `.obj`, `.ply`, `.dae`, `.3ds`, `.fbx`, `.gltf`, `.glb`) to STEP and imports it.
- A "Supported Formats" tool that lists the import and export formats reported by the installed Assimp build, and an "About" dialog.
- Export menu entries (STL, OBJ, PLY, glTF, DAE, 3DS, 3MF, FBX, STEP, and others). These are wired into the menu but not implemented; selecting one shows a placeholder message.

## Official Alibre Resources

Alibre's official resources for API development and AI/LLM/agent workflows: <https://www.alibre.com/api/>

## Requirements

- Alibre Design 29.0.0.29060. The project references `alibre-api.dll`, `AlibreAddOn.dll`, and `AlibreX.dll` from `C:\Program Files\Alibre Design 29.0.0.29060\Program`.
- .NET Framework 4.8.1, built for x64.
- AssimpNet 5.0.0-beta1 (NuGet), which supplies the managed wrapper and the bundled native Assimp libraries.
- A Windows toolchain with MSBuild or Visual Studio.

## Quick Start

1. Open `source/alibre-assimp.sln` in Visual Studio.
2. Restore NuGet packages and build in Release for x64.
3. Deploy the build output alongside its manifest and native libraries, then load it in Alibre Design (see Installation).

## Installation

1. Build `source/alibre-assimp.sln` in Release for x64.
2. Copy the build output to a deployment folder, including `alibre-assimp-addon.dll`, the manifest `alibre-assimp-addon.adc`, `AssimpNet.dll`, the `runtimes` folder holding the native Assimp libraries, and the `Icons` folder.
3. Register the `.adc` manifest with Alibre Design's add-on loader. The manifest identifier is `{0CC91391-49C4-4836-B7B4-04119ED38D99}`.
4. Restart Alibre Design.

## Usage

1. Open or create a session in Alibre Design.
2. Open the "Assimp Inside Alibre Design" menu.
3. Under Import, choose a format and select a file to load it into the current session.
4. Under Tools, use "Convert File to STEP" to convert and import any supported model, or "Supported Formats" to review the formats available in your Assimp build.

## Key Files

| File | Purpose |
| --- | --- |
| `source/AlibreAddOn.cs` | Add-on entry points, the menu tree, and the `AssimpService` wrapper around AssimpNet import/export/convert. |
| `source/alibre-assimp-addon.csproj` | Build settings: `net481`, x64, AssimpNet package reference, and the three Alibre API references. |
| `source/alibre-assimp.sln` | Visual Studio solution for the add-on. |
| `source/alibre-assimp-addon.adc` | Add-on manifest: identifier GUID, managed DLL, icon, and menu text. |
| `source/Icons/logo.ico` | Menu and manifest icon. |
| `source/alibre.disclaimer.txt` | Project disclaimer and licensing note. |
| `LICENSE` | MIT license text. |

## Key Folders

| Folder | Purpose |
| --- | --- |
| `source/` | The C# add-on project, manifest, icon, and build output. |
| `.github/` | This README plus issue templates, the pull request template, contributing guide, and code of conduct. |
| `reviews/` | Dated code-review notes. |
| `documentation/` | Placeholder for documentation; currently empty. |
| `submodules/` | Placeholder for git submodules; currently empty. |

## Notes

- Import routes through STEP: Assimp reads the mesh, writes a temporary STEP file, and Alibre imports that STEP. Fidelity depends on both the Assimp STEP exporter and Alibre's STEP import.
- The Export menu is not implemented. Each export command opens a save dialog and then shows a placeholder message.
- The native Assimp libraries ship under `runtimes` for `win-x64`, `win-x86`, `linux-x64`, and `osx-x64`; the add-on itself runs in Alibre Design on Windows x64.
- The API hint paths are absolute and pinned to the 29.0.0.29060 install directory. A different install path or version requires updating the references in the `.csproj`.

## License

See [LICENSE](../LICENSE). Licensed under the MIT License.

Alibre, Alibre Design, and Alibre Script names and related materials belong to their respective owners.
