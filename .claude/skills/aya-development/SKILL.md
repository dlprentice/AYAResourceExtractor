---
name: aya-development
description: Build, develop, debug, and run AYAResourceExtractor. Use when building the project, setting up development environment, running the application, debugging issues, or asking about build configuration, msbuild, Visual Studio setup, or test data.
---

# AYAResourceExtractor Development

## Quick Start

```powershell
# Build from VS Developer Command Prompt (x86)
msbuild Code/AyaResourceExtractor/AYAResourceExtractor.sln /p:Configuration=Release /p:Platform=x86

# Run the application
./Code/AyaResourceExtractor/bin/Release/net6.0-windows/AYAResourceExtractor.exe
```

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Visual Studio 2022 | Community edition or higher |
| C++ Desktop Workload | "Desktop development with C++" in VS Installer |
| C++/CLI Support | Individual component: "C++/CLI support for v143 build tools" |
| .NET 6.0 SDK | Windows desktop runtime |
| Platform | **x86 only** - native C++/CLI dependencies require 32-bit |

## Solution Structure

**Solution:** `Code/AyaResourceExtractor/AYAResourceExtractor.sln`

| Project | Type | Purpose |
|---------|------|---------|
| AYAResourceExtractor | C# WinForms (.NET 6) | Main application |
| Fbx | C# (.NET 6) | FBX file read/write |
| ZLib | C++ (native) | Compression library |
| ZLibWrapper | C++/CLI | .NET wrapper for ZLib |
| DDSTextureUncompress | C++/CLI | DXT texture decompression |

## Build Commands

### From VS Developer Command Prompt

Always use x86 Developer Command Prompt or specify platform explicitly:

```powershell
# Debug build
msbuild Code/AyaResourceExtractor/AYAResourceExtractor.sln /p:Configuration=Debug /p:Platform=x86

# Release build
msbuild Code/AyaResourceExtractor/AYAResourceExtractor.sln /p:Configuration=Release /p:Platform=x86

# Clean and rebuild
msbuild Code/AyaResourceExtractor/AYAResourceExtractor.sln /t:Clean /p:Configuration=Release /p:Platform=x86
msbuild Code/AyaResourceExtractor/AYAResourceExtractor.sln /t:Rebuild /p:Configuration=Release /p:Platform=x86
```

### From Visual Studio

1. Open `Code/AyaResourceExtractor/AYAResourceExtractor.sln`
2. Set Solution Platform to **x86** (not x64 or Any CPU)
3. Build > Build Solution (Ctrl+Shift+B)

## Output Locations

| Configuration | Output Path |
|---------------|-------------|
| Debug | `Code/AyaResourceExtractor/bin/Debug/net6.0-windows/` |
| Release | `Code/AyaResourceExtractor/bin/Release/net6.0-windows/` |

## Runtime Requirements

The following file must be present alongside the executable:

- `BoxWithTextures.fbx` - Template file for FBX generation (located at repo root)

Copy it to output directory if missing:
```powershell
copy BoxWithTextures.fbx Code/AyaResourceExtractor/bin/Release/net6.0-windows/
```

## Running the Application

1. Launch `AYAResourceExtractor.exe`
2. Select Battle Engine Aquila game folder
3. Optionally change output folder
4. Either:
   - Extract individual `.aya` file from `data/resources/meshes/`
   - Select "Extract all" for batch processing

## Debugging

### In Visual Studio

1. Set AYAResourceExtractor as startup project
2. Set breakpoints
3. Press F5 (Debug > Start Debugging)
4. Use immediate window for runtime inspection

### Common Debug Scenarios

**Texture issues:** Set breakpoints in DDSTextureUncompress project
**Decompression issues:** Set breakpoints in ZLibWrapper project
**FBX output issues:** Debug the Fbx project

## Test Data

**Game folder structure expected:**
```
<BEA Install>/
  data/
    resources/
      meshes/      <- .aya model files
      textures/    <- texture resources
```

**Output structure:**
```
<Output Folder>/
  MeshTextures/    <- Extracted textures (.png)
  *.fbx            <- Converted models (ascii or binary)
```

## Common Issues

### Build fails: "The imported project was not found"

Missing C++ workload or C++/CLI support. Install via VS Installer:
- Desktop development with C++
- C++/CLI support for v143 build tools

### Build fails: Platform mismatch

Native C++ projects only support Win32 (x86). Ensure:
- Solution Platform is set to x86
- msbuild uses `/p:Platform=x86`

### Runtime error: BoxWithTextures.fbx not found

Copy template file to output directory:
```powershell
copy BoxWithTextures.fbx Code/AyaResourceExtractor/bin/Release/net6.0-windows/
```

### DLL not found at runtime

Ensure all native DLLs are in output:
- ZLib.dll
- ZLibWrapper.dll
- DDSTextureUncompress.dll

If missing, rebuild solution - project dependencies should copy them automatically.

### Application crashes on startup

1. Verify .NET 6.0 Windows Desktop Runtime installed
2. Run from Debug config to see exception details
3. Check Event Viewer > Windows Logs > Application

## FBX Output Formats

| Format | Use Case |
|--------|----------|
| ASCII | Windows 3D Viewer compatibility |
| Binary | Blender import |

## Dependencies

- **ZLib** (Mark Adler) - .aya file decompression
- **DSSReader** (Juho Peltonen) - DXT texture decompression
- **FBX** (hamish-milne) - FBX file manipulation
