# AYAResourceExtractor (fork): agent guide

Read this before changing anything. It is the only instruction file in the repo; `CLAUDE.md` just points here.

## What this is

David's fork of Stuart Gillam's `stuart73/AYAResourceExtractor`: a Windows tool that extracts the models in the
PC *Battle Engine Aquila* `.aya` resource files and writes FBX files plus PNG textures. `README.md` is Stuart's
description of what it does and its limits (static single objects, no animation or bones). MIT (`LICENSE.txt`).

One Visual Studio 2022 solution, `Code/AyaResourceExtractor/AYAResourceExtractor.sln`, holds five projects:
`AYAResourceExtractor` (C# WinForms, `net6.0-windows`: the app, `.aya` uncompressor, model and texture
extractors, FBX exporter), `Fbx` (C#, hamish-milne), `ZLib` (C, Mark Adler) and the C++/CLI glue `ZLibWrapper`
and `DDSTextureUncompress` (DDS reader by Juho Peltonen). `BoxWithTextures.fbx` is a sample output. The only
build instructions that exist are the README's: open the solution in Visual Studio 2022 on Windows. WinForms
and C++/CLI do not build on Linux, so nothing here is built or run on this laptop.

Remotes: `origin` is `dlprentice/AYAResourceExtractor` (default branch `main`), `upstream` is
`stuart73/AYAResourceExtractor` (branch `master`). The fork is upstream `4e04952`, David's `53b10b0` (fixes the
DDS copy loop for non-square textures in `Code/DDSTextureUncompress/DDSTextureUncompress.cpp`) and this guide.

## How it is used

`~/Projects/game-dev/Onslaught-Career-Editor` vendors this fork as the submodule `references/AYAResourceExtractor`,
pinned at `53b10b0`, as its source reference for the `.aya` format. Its `tools/aya_extractor_source_audit.py`
refuses to run unless the submodule is at `EXTRACTOR_PIN` (and `references/Onslaught` at `ONSLAUGHT_PIN`), then
hashes every tracked file and checks for expected functions; the write-up is
`reverse-engineering/source-code/aya-resource-extractor-source-audit.md`. `npm run test:safety` scans it too.

## Rules

- Treat this as upstream code: no reformatting, renaming or cleanup. The two `AYAResourceExtractor - Backup*.csproj`
  files are stray but inventoried by the audit (`PROJECT_SUPPORT`); removing them is a pin bump, not a tidy-up.
- Useful feature branches and repo-local worktrees are allowed. Push small verified changes to the existing
  origin; do not force-push. If the authorized task also adopts changed
  extractor source in Onslaught-Career-Editor, update its submodule pointer and `EXTRACTOR_PIN`, then re-run
  the audit, `tools/aya_extractor_source_audit_tests.py` and `npm run test:safety`. A guide-only change needs
  a diff check, not a consumer pin bump or Windows build. Do not advance reference pins as a side effect.
- Never rewrite history here: the pins must always resolve. Take upstream changes with `git fetch upstream` and a
  merge of `upstream/master` into `main`, not a rebase.
- Keep `LICENSE.txt` and the third-party attributions in `README.md`. Nothing here needs `local-data/`.
