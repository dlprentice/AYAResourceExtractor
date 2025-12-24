---
name: fbx-export
description: FBX model export system for AYA game resources. Template-based approach using BoxWithTextures.fbx, converts triangle strips to triangle lists, handles coordinate system conversion (left-hand to right-hand). Use when working with FBX files, mesh export, coordinate conversions, the Fbx C# library, or triangle strip/list conversion.
---

# FBX Export System

## Quick Start

The FBX exporter uses a template-based approach:
1. Load template `BoxWithTextures.fbx` (pre-configured with 24 material slots)
2. Replace geometry data (vertices, indices, normals, UVs)
3. Update texture references
4. Write binary or ASCII FBX output

## Architecture

### Key Files

| File | Purpose |
|------|---------|
| `Code/AyaResourceExtractor/FbxModelExporter.cs` | Main export logic |
| `Code/Fbx/FbxIO.cs` | Static read/write methods |
| `Code/Fbx/FbxDocument.cs` | Top-level document node |
| `Code/Fbx/FbxNode.cs` | Node with properties and children |
| `Code/Fbx/FbxNodeList.cs` | Base class with node traversal |
| `BoxWithTextures.fbx` | Template file (project root) |

### Template Approach

The Fbx library only supports read/modify/write operations - it cannot create FBX structures from scratch. The template `BoxWithTextures.fbx` provides:
- Pre-configured geometry structure
- 24 material/texture slots (hardcoded limit)
- Proper FBX node hierarchy

## Coordinate System Conversion

**Source:** Left-handed (AYA game engine)
**Target:** Right-handed (FBX standard)

### Conversions Applied

```csharp
// Position Z: negate
vertices[vv++] = ver.position.x;
vertices[vv++] = ver.position.y;
vertices[vv++] = -ver.position.z;  // Negated

// Normal Z: negate
normals.Add(vertex.normal.x);
normals.Add(vertex.normal.y);
normals.Add(-vertex.normal.z);     // Negated

// Texture V: negate
UV.Add(vertex.U);
UV.Add(-vertex.V);                 // Negated
```

## Triangle Strip to Triangle List Conversion

AYA models use triangle strips for efficiency. FBX requires triangle lists.

### Algorithm

```csharp
private List<AyaIndexType> GenerateTriListIndicesFromTriStrip(AyaModel model)
{
    List<AyaIndexType> newIndices = new();

    foreach (List<AyaIndexType> indylist in model.AyaIndices)
    {
        int i = 0;
        while (i < indylist.Count)
        {
            AyaIndexType a, b, c;

            if (i == 0)
            {
                // First triangle: use first 3 vertices
                a = indylist[0];
                b = indylist[1];
                c = indylist[2];
                i += 3;
            }
            else
            {
                // Subsequent triangles: reuse 2 previous vertices
                a = indylist[i - 1];
                b = indylist[i - 2];
                c = indylist[i];

                // Winding correction: alternate vertex order
                if (i % 2 == 1)
                {
                    (b, a) = (a, b);
                }
                i += 1;
            }

            // Skip degenerate triangles
            if (a.index != b.index && a.index != c.index && b.index != c.index)
            {
                newIndices.Add(a);
                newIndices.Add(b);
                c.index = -(c.index + 1);  // Negative marks polygon end
                newIndices.Add(c);
            }
        }
    }
    return newIndices;
}
```

### FBX Polygon Encoding

FBX uses negative indices to mark polygon boundaries:
- `-4` means "end of polygon, vertex index 3"
- Formula: `actualIndex = -(encodedIndex + 1)`

## Fbx Library Usage

### Reading FBX

```csharp
FbxDocument documentNode = FbxIO.ReadBinary(fbxTemplateFilename);
// or
FbxDocument documentNode = FbxIO.ReadAscii(path);
```

### Navigating Nodes

```csharp
// By name (returns first match)
FbxNode objectsNode = documentNode["Objects"];
FbxNode geometryNode = objectsNode["Geometry"];

// Nested access
FbxNode normalsNode = geometryNode["LayerElementNormal"]["Normals"];

// By index
FbxNode textureNode = objectsNode.Nodes[index];

// Path-based
FbxNode node = documentNode.GetRelative("Objects/Geometry/Vertices");
```

### Setting Values

```csharp
// Set first property (Value shorthand)
geometryNode["Vertices"].Value = vertices;  // double[]
geometryNode["PolygonVertexIndex"].Value = indices;  // int[]

// Set specific property by index
node.Properties[1] = "Video::" + textureName;

// Nested property access
videoNode["Properties70"]["P"].Properties[4] = texturePath;
```

### Writing FBX

```csharp
FbxIO.WriteAscii(documentNode, outputPath);
FbxIO.WriteBinary(documentNode, outputPath);
```

## Data Structures

### AyaModel (source)

```csharp
class AyaModel
{
    List<AyaVertex> AyaVertices;           // Vertex data
    List<List<AyaIndexType>> AyaIndices;   // Triangle strips per sub-mesh
    List<string> Textures;                 // Texture paths
}

struct AyaVertex
{
    AyaVector position;  // x, y, z, w
    AyaVector normal;    // x, y, z, w
    float U, V;          // Texture coordinates
    uint colour;         // Vertex color (not exported to FBX)
}

struct AyaIndexType
{
    int index;    // Vertex index
    int texture;  // Material/texture index
}
```

## Limitations

- **Max 24 textures:** Hardcoded in template; logs error if exceeded
- **No vertex colors:** `AyaVertex.colour` is not exported
- **Template dependency:** Requires `BoxWithTextures.fbx` at runtime
- **Single geometry:** One geometry node per export

## Texture Handling

```csharp
int maxTextures = 24;
foreach (String texture in model.Textures)
{
    // Template has pre-configured Texture and Video nodes
    FbxNode TextureNode = objectsNode.Nodes[2 + maxTextures + texNumber];
    FbxNode VideoNode = objectsNode.Nodes[2 + maxTextures + maxTextures + texNumber];

    // Update references
    TextureNode["Media"].Value = "Video::" + textureName;
    TextureNode["FileName"].Value = fullPath;
    TextureNode["RelativeFilename"].Value = fullPath;

    VideoNode.Properties[1] = "Video::" + textureName;
    VideoNode["Filename"].Value = fullPath;
    VideoNode["RelativeFilename"].Value = fullPath;
    VideoNode["Properties70"]["P"].Properties[4] = fullPath;

    // Extract texture from game resources
    AyaTextureExtractor.ExtractorTexture(texture, resourcesPath, outputPath);
}
```

## Common Tasks

### Debug FBX Structure

Uncomment to write ASCII version of template:
```csharp
FbxIO.WriteAscii(documentNode, @"BoxWithTextures_ascii.fbx");
```

### Add More Texture Slots

1. Edit `BoxWithTextures.fbx` in a 3D application
2. Add more materials with textures
3. Update `maxTextures` constant in code
4. Ensure node indices align with template structure
