# A Link to the Past Recompiled

**Static recompilation of *The Legend of Zelda: A Link to the Past & Four Swords* (GBA, 2002) to native Windows.**

Driving the development of [gbarecomp](https://github.com/sp00nznet/gbarecomp) - the first GBA static recompilation toolkit - by throwing a much larger, more complex game at it than the toolkit's first target (Advance Wars).

## Why this game?

Advance Wars proved gbarecomp works on a 6.3K-function, mostly-Thumb GBA cart. **A Link to the Past** is roughly 2.4x bigger, mixes ARM and Thumb, and bundles the Four Swords multiplayer mode on top of the SNES port. If gbarecomp can handle this, it can handle most of the GBA library.

## Status

| Phase | State |
|-------|-------|
| ROM loaded | Done - 8 MB, AZLE, GBAZELDA |
| Static analysis | Done - 15,264 functions, 114,908 basic blocks discovered |
| Code coverage | 25.8% (Thumb), 0.0% (ARM) - 74.2% still unanalyzed (likely data + late-discovered code) |
| ARM/Thumb translation | Done - 15,264 functions emitted across 153 source files (114 MB of C) |
| Stub generation | 7,331 stub functions, 832 unresolved targets |
| Native compile | In progress |
| Game boot | Not started |
| Title screen | Not started |
| Gameplay | Not started |

Translation numbers from initial run (gbarecomp commit at project start):

```
Functions:          15264 (8762 leaf)
Basic blocks:       114908 (726 with unresolved indirect branches)
Jump tables:        0
ARM instructions:   258
Thumb instructions: 1080102
Stub functions:     7331 (832 unresolved)
Generated files:    157 (.c) at 114 MB
```

## What this project is for

Two parallel goals:

1. **Make A Link to the Past run natively on modern PCs** - no emulator underneath, just C compiled by MSVC/clang against SDL2.
2. **Push gbarecomp** - LttP will expose analysis gaps, translator bugs, missing BIOS calls, and runtime corner cases that Advance Wars never hit. Each fix lands upstream in [gbarecomp](https://github.com/sp00nznet/gbarecomp).

## Build

You provide the ROM yourself. None of the game data is included in this repo.

```bash
# Build gbarecomp first
cd ../gbarecomp
cmake -B build && cmake --build build --config Release

# Translate
cd ../lttp
../gbarecomp/build/Release/gbarecomp.exe translate "Legend of Zelda, The - A Link to the Past & Four Swords (USA, Australia).gba" -o recomp_out --multi

# Copy runtime + display (gbarecomp does not yet emit these automatically)
cp ../gbarecomp/src/runtime.c recomp_out/
cp ../gbarecomp/src/display.c recomp_out/

# Compile
cd recomp_out
cmake -B build -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake
cmake --build build --config Release

# Run
./build/Release/AZLE.exe
```

## Related

- [gbarecomp](https://github.com/sp00nznet/gbarecomp) - the toolchain
- [advancewars-recompiled](https://github.com/sp00nznet/advancewars-recompiled) - the first GBA game to come through gbarecomp
- [N64Recomp](https://github.com/N64Recomp/N64Recomp) - the architectural inspiration
- [pret/zelda3](https://github.com/snesrev/zelda3) - SNES LttP reverse-engineering reference (different platform, but invaluable for naming/structure)

## Legal

This repo contains no copyrighted game data. You must provide your own legally-obtained ROM. The recompilation tools and runtime are open source.

---

*Built with Claude Code.*
