---
name: texture-extraction
description: Extract and decode textures from AYA game resource files. Covers zlib decompression, DDS/DXT texture decoding, file naming conventions, and PNG output. Use when working with texture files, DDS decompression, DXT formats, texture extraction, or PNG export from AYA resources.
---

# AYA Texture Extraction Pipeline

## Quick Start

Textures are extracted in three stages:
1. Locate the `.aya` file using the naming convention
2. Decompress chunked zlib data
3. Decode DDS/DXT texture to BGRA, output as 32-bit ARGB PNG

## File Naming Convention

Texture files are located at:
```
dxtntextures/meshtex%{name}(0){format}.aya
```

Where `{format}` is one of:
- `A1R5G5B5` - 16-bit color with 1-bit alpha
- `A8R8G8B8` - 32-bit color with 8-bit alpha (fallback)

**Lookup order**: Try `A1R5G5B5` first, then `A8R8G8B8`.

Example paths:
```
dxtntextures/meshtex%tank_body(0)A1R5G5B5.aya
dxtntextures/meshtex%soldier_face(0)A8R8G8B8.aya
```

## Compression Format

AYA files use **chunked zlib compression** (same as model files):

```
[4-byte size][zlib chunk][4-byte size][zlib chunk]...
```

Each chunk decompresses to max 1MB. Multiple chunks are concatenated.

**Implementation**: `AyaFileUncompressor.cs`

## DDS/DXT Texture Format

After zlib decompression, data is a standard DDS file containing DXT-compressed texture.

### Supported DXT Formats
| FourCC | Format | Notes |
|--------|--------|-------|
| DXT1 | BC1 | 1-bit alpha or no alpha |
| DXT2 | BC2 | Premultiplied alpha (uses DXT3 decoder) |
| DXT3 | BC2 | Explicit 4-bit alpha |
| DXT5 | BC3 | Interpolated alpha |

### RGB Formats
Also supports uncompressed RGB with `DDPF_RGB` flag:
- 16-bit (A1R5G5B5)
- 32-bit (A8R8G8B8)

**Implementation**: `DDSTextureUncompress` (C++/CLI wrapper)

## Output Format

- **Pixel format**: 32-bit ARGB PNG
- **Color order**: BGRA from decoder, converted to ARGB for PNG
- **Alpha**: Preserved from source

## Known Bug

**File**: `DDSTextureUncompress.cpp` line 28

```cpp
// BUG: Uses height*height instead of width*height
for (int i = 0; i < image->height * image->height * 4; i++)
```

Should be:
```cpp
for (int i = 0; i < image->width * image->height * 4; i++)
```

This causes incorrect data copy for non-square textures.

## Key Files

| File | Purpose |
|------|---------|
| `Code/AyaResourceExtractor/AyaTextureExtractor.cs` | Main extraction logic |
| `Code/AyaResourceExtractor/AyaFileUncompressor.cs` | Chunked zlib decompression |
| `Code/DDSTextureUncompress/DDSTextureUncompress.cpp` | C++/CLI DDS decoder wrapper |
| `Code/DDSTextureUncompress/DDSReader.cpp` | DXT block decompression |
| `Code/DDSTextureUncompress/DDSreader.h` | Image struct definition |

## Extraction Flow

```
AyaTextureExtractor.ExtractorTexture()
    |
    v
Locate file: dxtntextures/meshtex%{name}(0){A1R5G5B5|A8R8G8B8}.aya
    |
    v
AyaFileUncompressor.Uncompress() -> byte[]
    |
    v
DDSTextureUncompress.Uncompress() -> BGRA pixels + width/height
    |
    v
Bitmap.SetPixel() loop: BGRA -> ARGB
    |
    v
Bitmap.Save() as PNG
```

## DXT Decoding Details

The DDS reader uses FreeImage-derived code:

1. **Block-based**: DXT textures are 4x4 pixel blocks
2. **Color decoding**: RGB565 expanded to RGB888
3. **Alpha handling**:
   - DXT1: 1-bit alpha (transparent if color0 <= color1)
   - DXT3: 4-bit explicit alpha per pixel
   - DXT5: Interpolated 3-bit alpha indices

**Color order on little-endian (Windows)**: BGR order in memory, converted via `FI_RGBA_*` macros.
