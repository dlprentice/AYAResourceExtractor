---
name: aya-architecture
description: AYAResourceExtractor architecture and project structure. Extracts 3D models from Battle Engine Aquila .aya files to FBX format. Use when asking how the extractor works, understanding the project structure, data flow, adding new features, or working with AYA file formats.
---

# AYAResourceExtractor Architecture

Tool for extracting 3D models and textures from Battle Engine Aquila's proprietary .aya file format to standard FBX.

## Quick Start

- Main entry: `MainForm.cs` (WinForms UI)
- Core extraction: `AyaModelExtractor.cs` orchestrates the pipeline
- File format parsing: `AyaModelImporter.cs` reads binary .aya structure
- Output: `FbxModelExporter.cs` writes FBX using the Fbx library

## Project Structure

```
AYAResourceExtractor.sln
|
+-- AYAResourceExtractor/     [C# WinForms - Main Application]
|   +-- MainForm.cs           UI, batch extraction thread
|   +-- AyaModelExtractor.cs  Pipeline orchestrator
|   +-- AyaFileUncompressor.cs   Multi-part zlib decompression
|   +-- AyaModelImporter.cs   Binary .aya parser -> AyaModel
|   +-- AyaModel.cs           Data structures (vertices, indices)
|   +-- AyaTextureExtractor.cs   DDS texture -> PNG
|   +-- FbxModelExporter.cs   AyaModel -> FBX file
|   +-- Log.cs                Thread-safe logging singleton
|
+-- Fbx/                      [C# Library - FBX Read/Write]
|   +-- FbxIO.cs              Binary/ASCII FBX I/O
|   +-- FbxDocument.cs        FBX document model
|   +-- FbxNode.cs            FBX node structure
|
+-- ZLib/                     [Native C - zlib compression]
+-- ZLibWrapper/              [C++/CLI - Managed wrapper for ZLib]
+-- DDSTextureUncompress/     [C++/CLI - DDS texture decompression]
```

## Architecture Diagram

```
+------------------+
|    MainForm      |  WinForms UI
|  (UI Thread)     |  - File/folder selection
+--------+---------+  - Batch extraction control
         |
         | ExtractFile() / ExtractAll() [background thread]
         v
+------------------+
| AyaModelExtractor|  Pipeline Orchestrator
+--------+---------+
         |
    +----+----+--------------------+
    |         |                    |
    v         v                    v
+-------+ +----------+      +-------------+
|AyaFile| |AyaModel  |      |FbxModel     |
|Uncomp | |Importer  |      |Exporter     |
+---+---+ +----+-----+      +------+------+
    |          |                   |
    v          v                   v
+-------+  +-------+         +----------+
|ZLib   |  |AyaModel|        |Fbx Library|
|Wrapper|  |(structs)|       +-----+----+
+-------+  +-------+               |
                                   v
                          +----------------+
                          |AyaTexture      |
                          |Extractor       |
                          +-------+--------+
                                  |
                                  v
                          +----------------+
                          |DDSTexture      |
                          |Uncompress      |
                          +----------------+
```

## Data Flow Pipeline

```
.aya file (compressed)
       |
       v [AyaFileUncompressor]
       |  - Read 4-byte size headers
       |  - Decompress zlib chunks (1MB max each)
       |  - Concatenate into single byte[]
       v
Raw binary data
       |
       v [AyaModelImporter]
       |  - Parse tagged chunks (CMST, MSHT, MESP, PMVB, etc.)
       |  - Extract vertices, normals, UVs, colors
       |  - Apply part transforms (orientation matrix + position)
       |  - Handle REFR (part references for repeated geometry)
       v
AyaModel
  +-- List<AyaVertex> (position, normal, UV, color)
  +-- List<List<AyaIndexType>> (tri-strips per material)
  +-- List<string> Textures
       |
       v [FbxModelExporter]
       |  - Convert tri-strips to tri-lists
       |  - Populate FBX template (BoxWithTextures.fbx)
       |  - Set vertices, indices, normals, UVs, materials
       v
.fbx file (binary and/or ASCII)
       |
       +---> [AyaTextureExtractor] for each texture
                  |
                  v
             .aya texture (DDS in zlib)
                  |
                  v [AyaFileUncompressor]
                  v [DDSTextureUncompress]
                  v
             .png file
```

## Key Data Structures

```csharp
// Core vertex data
struct AyaVertex {
    AyaVector position;   // x, y, z, w
    AyaVector normal;     // x, y, z, w
    float U, V;           // texture coordinates
    uint colour;          // vertex color (ARGB)
}

// Index with material reference
struct AyaIndexType {
    int index;    // vertex index
    int texture;  // material/texture index
}

// Transform matrix for mesh parts
class AyaMatrix {
    AyaVector xAxis, yAxis, zAxis;
    AyaVector MultiplyWithVector(AyaVector v);
}

// Complete model representation
class AyaModel {
    List<AyaVertex> AyaVertices;
    List<List<AyaIndexType>> AyaIndices;  // tri-strips per material
    List<string> Textures;
}
```

## AYA Binary Format Tags

| Tag  | Purpose |
|------|---------|
| CMST | Material/texture count section |
| MSHT | Material header |
| TEXB | Texture binding |
| MESP | Mesh part header |
| CMSP | Part transform data |
| CHLD | Child part references |
| PRNT | Parent part reference |
| BBOX | Bounding box |
| PMVB | Part mesh vertex buffer |
| CMVB | Mesh vertex buffer header |
| MMPT | Material vertex/index buffer |
| IBUF | Index buffer |
| VBUF | Vertex buffer |
| TEXR | Texture reference indices |
| REFR | Reference to another part's geometry |
| BONE/BONW/BONS | Bone/skeletal data |

## Threading Model

- **UI Thread**: MainForm handles user interaction
- **Background Thread**: `ExtractAll()` runs batch extraction via `Thread`
- **Thread-safe Logging**: `Log` singleton uses `lock` and `Control.Invoke()` for cross-thread TextBox updates

## Adding New Features

### New Export Format
1. Create new exporter class (e.g., `ObjModelExporter.cs`)
2. Accept `AyaModel` as input
3. Call from `AyaModelExtractor.ExtractModel()` after import

### New File Format Support
1. Create importer that produces `AyaModel`
2. Reuse existing `FbxModelExporter` for output

### Additional Mesh Data
1. Extend `AyaVertex` or `AyaModel` with new fields
2. Parse new tags in `AyaModelImporter.ReadModelPart()`
3. Export new data in `FbxModelExporter`

## Dependencies

| Component | Type | Purpose |
|-----------|------|---------|
| ZLib | Native C | zlib decompression |
| ZLibWrapper | C++/CLI | Managed wrapper for ZLib |
| DDSTextureUncompress | C++/CLI | DDS format decompression |
| Fbx | C# Library | FBX file read/write |
| System.Drawing | .NET | Bitmap/PNG output |

## Build Configuration

- Platform: x86 (32-bit) - required for native DLLs
- Framework: .NET (WinForms)
- Dependencies: ZLib -> ZLibWrapper -> DDSTextureUncompress (build order matters)
