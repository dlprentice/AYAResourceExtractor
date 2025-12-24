# AYAResourceExtractor - Task Tracking

## Known Bugs

- [ ] **DDSTextureUncompress.cpp:28** - Uses `height * height` instead of `width * height` for non-square textures
  - File: `Code/DDSTextureUncompress/DDSTextureUncompress.cpp`
  - Line 28: `for (int i = 0; i < image->height * image->height * 4; i++)`
  - Should be: `image->width * image->height * 4`

## Known Limitations

- [ ] Some normals point in wrong direction (root cause unknown)
- [ ] Max 24 textures per model (hardcoded in FBX template)
- [ ] Vertex colors parsed but not exported to FBX
- [ ] Bones/animation data parsed but not exported
- [ ] No multi-texture blending support
- [ ] Models exported as single static mesh (no part hierarchy preserved)

## Potential Improvements

- [ ] Support Steam version of Battle Engine Aquila (untested)
- [ ] Add progress bar for batch extraction
- [ ] Persist settings between sessions (currently empty Settings.Designer.cs)
- [ ] Export as separate parts instead of merged mesh
- [ ] Export bone/skeleton data
- [ ] Export animation data

## Completed

(None yet)
