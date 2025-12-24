# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AYAResourceExtractor is a Windows Forms application that extracts 3D models from Battle Engine Aquila (2003 PC version) `.aya` files and converts them to FBX format. The tool handles zlib decompression, binary parsing of a proprietary format, DXT texture decompression, and FBX export.

## Build Commands

Open solution in Visual Studio 2022:
```
Code\AyaResourceExtractor\AYAResourceExtractor.sln
```

Build from command line (requires VS 2022 Build Tools):
```bash
msbuild Code\AyaResourceExtractor\AYAResourceExtractor.sln /p:Configuration=Release /p:Platform=x86
```

**Requirements:**
- Visual Studio 2022 with C++ desktop development workload
- .NET 6.0 SDK
- Platform must be x86 (native C++/CLI dependencies)

## Architecture

```
MainForm.cs (UI)
    │
    ▼
AyaModelExtractor.cs (orchestrator)
    │
    ├── AyaFileUncompressor.cs ──► ZLibWrapper (C++/CLI) ──► ZLib (native)
    │
    ├── AyaModelImporter.cs (binary parser) ──► AyaModel.cs (data container)
    │
    └── FbxModelExporter.cs ──► Fbx library (C#)
            │
            └── AyaTextureExtractor.cs ──► DDSTextureUncompress (C++/CLI)
```

### Projects in Solution

| Project | Type | Purpose |
|---------|------|---------|
| AYAResourceExtractor | C# WinForms | Main application |
| Fbx | C# Library | FBX file read/write |
| ZLib | C++ Static Lib | zlib compression |
| ZLibWrapper | C++/CLI DLL | .NET wrapper for ZLib |
| DDSTextureUncompress | C++/CLI DLL | DDS/DXT texture decoder |

### Key Data Flow

1. **Decompression**: `.aya` files use chunked zlib (1MB limit per chunk). Each chunk has 4-byte size header followed by compressed data.

2. **Binary Parsing**: Tag-based format with 4-char ASCII tags. Key tags:
   - `CMST`, `MSHT`, `TEXB` - texture definitions
   - `MESP`, `CMSP` - mesh parts with transformation matrices
   - `CMVB`, `MMPT`, `VBUF`, `IBUF` - vertex/index buffers
   - `TEXR` - texture references per material
   - `REFR` - geometry instancing (copies from another part)

3. **FBX Export**: Template-based approach using `BoxWithTextures.fbx`. Modifies nodes for vertices, normals, UVs, materials. Converts triangle strips to triangle lists.

4. **Textures**: Located at `dxtntextures\meshtex%{name}(0){format}.aya`. DXT2/3/5 compressed, exported as 32-bit PNG.

### Core Data Structures (AyaModel.cs)

```csharp
AyaVertex: position (xyz), normal (xyz), UV, colour (ARGB packed uint)
AyaIndexType: vertex index + texture/material ID
AyaMatrix: 3x4 transformation matrix
AyaModel: List<AyaVertex>, List<List<AyaIndexType>> (tri-strips), List<string> Textures
```

### Coordinate System

FBX export flips: Z-position negated, Z-normal negated, V-texture negated (left-hand to right-hand conversion).

## Known Issues

- Some normals point wrong direction
- Max 24 textures per model (template limitation)
- Vertex colors parsed but not exported
- Bones/animations parsed but not exported
- Bug in `DDSTextureUncompress.cpp:28`: uses `height * height` instead of `width * height`

## File Locations

- Models: `{game}\data\resources\meshes\*.aya`
- Textures: `{game}\data\resources\dxtntextures\meshtex%*.aya`
- FBX Template: `BoxWithTextures.fbx` (required at runtime)

## Task Tracking

Issues, bugs, and improvements are tracked in: `tasks/todo.md`

## Skills (Auto-Discovered Knowledge)

The following skills provide detailed knowledge for specific tasks:

| Skill | Triggers When |
|-------|---------------|
| `aya-file-format` | Parsing .aya files, binary format, AYA tags |
| `aya-architecture` | Project structure, data flow, adding features |
| `aya-development` | Building, debugging, VS setup, common issues |
| `fbx-export` | FBX output, coordinate conversion, Fbx library |
| `texture-extraction` | DDS/DXT decoding, texture pipeline, PNG export |

Skills are located in `.claude/skills/` and are automatically loaded based on context.
