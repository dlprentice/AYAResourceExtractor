---
name: aya-file-format
description: Parse and understand AYA binary files from Battle Engine Aquila. Explains chunked zlib compression, tag-based format structure, mesh data layout, vertex formats, and texture references. Use when working with .aya files, parsing binary mesh data, understanding AYA tags like CMST/MSHT/MESP/VBUF/IBUF, or extracting 3D models.
---

# AYA Binary File Format

Reference for the AYA resource format used by Battle Engine Aquila (2003).

## Quick Start

1. Decompress chunked zlib data (see [Compression](#compression))
2. Parse tag-based structure starting at offset 8
3. Read mesh hierarchy via MESP/CMSP tags
4. Extract vertices from VBUF, indices from IBUF

## Compression

AYA files use chunked zlib compression with a 1MB limit per chunk:

```
[4 bytes: compressed chunk size]
[zlib compressed data]
[4 bytes: next chunk size]  (if file > 1MB uncompressed)
[zlib compressed data]
...
```

Most files have a single chunk. Larger files (complex meshes) have multiple chunks that must be concatenated after decompression.

## File Header

After decompression, the binary structure begins:

| Offset | Size | Description |
|--------|------|-------------|
| 0 | 8 | Unknown header bytes |
| 8 | 4 | Unknown |
| 12 | 4 | Number of textures |
| 16 | 28 | Reserved (7 x uint32) |
| 44 | 300 | Mesh name (null-terminated string) |
| 344 | 12 | Unknown |
| 356 | 4 | Number of parts |
| 360 | 20 | Reserved (5 x uint32) |

## Tag-Based Structure

All data sections use 8-byte headers:
- 4 bytes: ASCII tag name (e.g., "CMST", "MESP")
- 4 bytes: Section length (uint32, little-endian)

Tags are read sequentially. Some tags are optional depending on mesh features.

See [TAGS.md](TAGS.md) for complete tag reference.

## Vertex Formats

Two vertex formats exist:

### Standard Vertex (36 bytes)
| Offset | Type | Field |
|--------|------|-------|
| 0 | float3 | Position (x, y, z) |
| 12 | float3 | Normal (x, y, z) |
| 24 | uint32 | Vertex color (ARGB) |
| 28 | float | U texture coordinate |
| 32 | float | V texture coordinate |

### Skinned Vertex (48 bytes)
| Offset | Type | Field |
|--------|------|-------|
| 0 | float3 | Position (x, y, z) |
| 12 | float3 | Bone weights (3 floats) |
| 24 | float3 | Normal (x, y, z) |
| 36 | uint32 | Vertex color (ARGB) |
| 40 | float | U texture coordinate |
| 44 | float | V texture coordinate |

The vertex chunk size in CMVB determines which format is used.

## Primitive Type

Meshes use **triangle strips** (primitive type stored in CMVB). Index buffers contain strip indices that must be converted to triangles for standard rendering.

## Part Hierarchy

Each MESP defines a mesh part with:
- Local and base orientation matrices (3x4 floats each)
- Position offsets
- Child/parent relationships (CHLD, PRNT tags)
- Optional bone data (BONE, BONW, BONS)

Parts can reference other parts via REFR tag to share geometry.

## Texture System

Textures are stored separately in `.aya` files under `dxtntextures/meshtex%`:
- Format: `meshtex%<name>(0)A1R5G5B5.aya` or `A8R8G8B8.aya`
- Contains DDS-compressed texture data after zlib decompression

## Key Implementation Notes

1. **Byte order**: Little-endian throughout
2. **String encoding**: UTF-8, null-terminated
3. **Matrix layout**: Row-major, 3x4 (no perspective row)
4. **Coordinate system**: Y-up, right-handed
5. **Index type**: uint16 (limits to 65535 vertices per buffer)

## Related Files

- See [TAGS.md](TAGS.md) for complete tag reference with sizes and purposes
