# Assimp Inside Alibre Design

An Alibre Design add-on that imports common 3D mesh formats into Alibre using the Assimp (Open Asset Import) library. Selected files are converted to STEP via Assimp and brought into the active session through Alibre's STEP import.

## Features
- Adds an "Assimp Inside Alibre Design" menu with Import, Export, and Tools groups.
- Imports mesh formats and loads them into Alibre as STEP geometry: glTF (`.gltf`/`.glb`), OBJ, FBX, DAE (Collada), PLY, 3DS, and 3DM.
- "Convert File to STEP" tool converts a chosen 3D model (`.stl`, `.obj`, `.ply`, `.dae`, `.3ds`, `.fbx`, `.gltf`, `.glb`) to STEP and imports it.
- "Supported Formats" tool lists the import and export formats reported by the installed Assimp build.
- Export menu entries are present but not yet implemented (they display a placeholder notice).

## Requirements
- Alibre Design 29.0.0.29060 (the project references the API from this install).
- .NET Framework 4.8.1 (x64).
- AssimpNet 5.0.0-beta1 and its bundled native Assimp libraries.

## Installation
1. Build the project in `source/` (`alibre-assimp.sln`) in Release for x64.
2. Copy the build output to a folder, including `alibre-assimp-addon.dll`, the add-on manifest `alibre-assimp-addon.adc`, the native Assimp libraries, and the `Icons` folder.
3. Register the `.adc` manifest with Alibre Design's add-on loader so it loads at startup.
4. Restart Alibre Design.

## Usage
1. Open or create a session in Alibre Design.
2. Open the "Assimp Inside Alibre Design" menu.
3. Choose a format under Import and select a file to load it into the current session, or use Tools > Convert File to STEP to convert and import any supported model.
4. Use Tools > Supported Formats to review the formats available in your Assimp build.

## License
See [LICENSE](../LICENSE).
