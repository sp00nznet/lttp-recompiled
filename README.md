# A Link to the Past Recompiled

**Static recompilation of *The Legend of Zelda: A Link to the Past & Four Swords* (GBA, 2002) to native Windows.**

Driving the development of [gbarecomp](https://github.com/sp00nznet/gbarecomp) - the first GBA static recompilation toolkit - by throwing a much larger, more complex game at it than the toolkit's first target (Advance Wars).

## Status

| Phase | State |
|-------|-------|
| ROM loaded | Done - 8 MB, AZLE, GBAZELDA |
| Static analysis | Done - 15,264 functions, 114,908 basic blocks |
| ARM/Thumb translation | Done - 15,264 functions across 153 source files (114 MB of C) |
| Stub generation | Done - 7,331 stubs + 833 trap stubs for unresolved targets |
| Native compile | **Done** - 20.8 MB AZLE.exe (MSVC /O1, MP, ~30 min) |
| Runtime init | **Done** - SDL2 window, flat memory, BIOS HLE all online |
| Boot frames | **Done** - 300+ frames render at 60 fps |
| Game past forced blank | Not yet - DISPCNT stays at 0x0080 (forced blank) |
| Title screen | Not yet |
| Gameplay | Not yet |

```
$ ./AZLE.exe game.gba
[runtime] GBA static recomp runtime initialized
[runtime] ROM: 8388608 bytes (game.gba)
[frame 1]   DISPCNT=0x0080 mode=0 BG=0000 OBJ=0
[frame 2]   DISPCNT=0x0080 mode=0 BG=0000 OBJ=0
...
[frame 300] DISPCNT=0x0080 mode=0 BG=0000 OBJ=0
[interp] step limit hit at 0x030059B0 (2000000 steps)
```

That last line is the next thing to fix. The game copied a routine into IWRAM at 0x030059B0 (game's crt0 does this for IRQ-handling code), and gbarecomp's runtime ARM/Thumb interpreter is spinning in it for 2M instructions before bailing out.

### Deeper dive on the IRQ spin

After adding diagnostics that dump bytes + IE/IF/IME state on step-limit-hit, the picture is:

```
[interp] step limit hit at 0x030059B0 (2000000 steps), pc=0x047A13BC
[interp]   bytes around target: 0000 0000 0000 0000 0000 0000 0000 0000 3301 E3A0 3C02 E283 2000 E593 10B8 E1D3
[interp]   bytes around final pc: 0000 0000 0000 0000 0000 0000 0000 0000 ...
[interp]   IE=0x2005 IF=0x0007 IME=1 DISPSTAT=0x0029 VCOUNT=205
```

Hex-searching the ROM for the first 16 bytes of the entry pinpoints the source: ROM `0x08000104` — the standard GBA IRQ handler stub the game's startup copies to IWRAM. Disassembling it:

```
MOV r3, #0x04000000        ; IE/IF/IME base
ADD r3, r3, #0x200
LDR r2, [r3, #0]           ; r2 = (IF<<16)|IE
LDRH r1, [r3, #8]          ; r1 = IME
MRS r0, SPSR
STMDB sp!, {r0, r1, r2, r3, lr}
MOV r0, #1; STRH r0, [r3, #8]   ; IME=1
AND r1, r2, r2 LSR #16     ; pending = IE & IF
... walk bits, find which IRQ fired (r12 = index*4)
0x080001D4: LDR r1, [pc, #0x38]   ; r1 = subhandler-table-base = 0x03000B70
0x080001D8: ADD r1, r1, r12       ; r1 = &handler[index]
0x080001DC: LDR r0, [r1]          ; r0 = handler function pointer
0x080001E0: STMDB sp!, {lr}
0x080001E4: ADD lr, pc, #0
0x080001E8: BX r0                 ; → bad pc 0x047A13BC
```

So: the runtime scheduler delivers a VBlank IRQ before the game has populated its subhandler table at IWRAM 0x03000B70. `BX r0` lands at uninitialized garbage, interpreter falls into walking zeros forever. This is a runtime-vs-game-init race that didn't bite Advance Wars because of timing differences.

### Two upstream gbarecomp fixes from this session

[sp00nznet/gbarecomp@5e8a680](https://github.com/sp00nznet/gbarecomp/commit/5e8a680):
1. **Stub generator no longer skips non-ROM call targets.** LttP's translator emitted a `BL` to `0x07EC7956` (OAM region — clearly wrong, post-undefined-instruction garbage). The unresolved-stub pass had `if ((addr >> 24) != 0x08) continue` and silently skipped it, breaking the link with no recovery path. Now every referenced address gets a trap stub.
2. **Generated CMakeLists uses /O1 + /MP and auto-copies runtime.c/display.c.** /O2 on the >2 MB generated translation units LttP produces (funcs_118.c is 2.5 MB) makes MSVC's regalloc burn 20+ minutes per file. /O1 with parallel cl.exe makes the build practical.

[sp00nznet/gbarecomp@f4a9910](https://github.com/sp00nznet/gbarecomp/commit/f4a9910):
3. **Runtime bails on ARM `BX Rm` to unmapped region** (i.e. not BIOS/EWRAM/IWRAM/ROM) instead of falling into a 2M-step walk through zero memory. Also added byte+register-state diagnostics on step-limit hits.

## What's done

| Component | How |
|-----------|-----|
| Analysis (recursive descent CFG) | gbarecomp - 7-phase pipeline finds 15,264 functions |
| ARM/Thumb → C translation | gbarecomp - emits 153 .c files at 114 MB total |
| Stubs + unresolved trap stubs | gbarecomp - covers every referenced address |
| Standalone runtime | gbarecomp/src/runtime.c - flat memory, DMA, timers, IRQ, BIOS HLE |
| Display | gbarecomp/src/display.c - SDL2, BG mode 0/1/3/4, OBJ |
| Native compile | MSVC + vcpkg SDL2 |
| Save (SRAM) | runtime.c - .sav auto-load/save |

## Open questions / next dives

- The new ARM-BX-to-unmapped guard didn't catch anything, so the bad `pc=0x047A13BC` arrives via a non-BX path (`LDR pc, [...]`, `LDM ... ^` IRQ-return form, or `MSR` mishandled and clobbering pc). The interpreter has special cases for MSR/MRS but the data-processing handler runs *first* in the if-chain and may misclassify MSR (rd=15, S=0) as TEQ.
- Even if we fix that path, the underlying issue is timing: the standalone runtime delivers VBlank/HBlank IRQs based on scanline ticks, but the game's startup writes to its IWRAM subhandler table at its own pace. Need to either (a) gate IRQ delivery on the game having written non-zero into the subhandler slot, or (b) install a "no-op handler if uninitialized" check inside the runtime.
- Once IRQ dispatch works, `DISPCNT` should clear forced-blank during init and the title screen logo should appear.

## Numbers

```
=== Analysis Summary ===
  Functions:          15264 (8762 leaf)
  Basic blocks:       114908 (726 with unresolved indirect branches)
  Jump tables:        0
  ARM instructions:   258
  Thumb instructions: 1080102
  Code coverage:
    ARM code:         1032 bytes (0.0%)
    Thumb code:       2160202 bytes (25.8%)
    Data:             0 bytes (0.0%)
    Unanalyzed:       6227374 bytes (74.2%)
========================

--- Translation ---
[translate] 7331 stub functions needed
[translate] 833 unresolved function stubs generated
Generated 157 files in: D:/recomp/gba/lttp/recomp_out
Functions translated: 15264
```

## What this project is for

Two parallel goals:

1. **Make A Link to the Past run natively on modern PCs** - no emulator underneath, just C compiled by MSVC/clang against SDL2.
2. **Push gbarecomp** - LttP exposes analysis gaps, translator bugs, missing BIOS calls, and runtime corner cases that Advance Wars never hit. Each fix lands upstream in [gbarecomp](https://github.com/sp00nznet/gbarecomp).

## Build

You provide the ROM. None of the game data is in this repo.

```bash
# Build gbarecomp first
cd ../gbarecomp
cmake -B build && cmake --build build --config Release

# Translate (output is regenerated each time, do not edit by hand)
cd ../lttp
../gbarecomp/build/Release/gbarecomp.exe translate \
  "Legend of Zelda, The - A Link to the Past & Four Swords (USA, Australia).gba" \
  -o recomp_out --multi

# Compile (gbarecomp >= 5e8a680 auto-emits CMakeLists with /O1 /MP and copies runtime.c/display.c)
cd recomp_out
cmake -B build -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake
cmake --build build --config Release

# Run
./build/Release/AZLE.exe game.gba
```

## Related

- [gbarecomp](https://github.com/sp00nznet/gbarecomp) - the toolchain
- [N64Recomp](https://github.com/N64Recomp/N64Recomp) - the architectural inspiration
- [snesrev/zelda3](https://github.com/snesrev/zelda3) - SNES LttP reverse-engineering reference (different platform, but useful for naming and structure)

## Legal

This repo contains no copyrighted game data. You must provide your own legally-obtained ROM. The recompilation tools and runtime are open source.

---

*Built with Claude Code.*
