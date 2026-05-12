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
| Boot frames | **Done** - 600+ frames render at 60 fps, IRQ handler runs cleanly |
| Game past forced blank | **Done** - DISPCNT moves from 0x0080 -> 0x0040 (forced blank cleared) |
| Main loop running | **Done** - state-machine dispatcher at func_0800D788 cycles at ~3700 iter/sec |
| Title screen | Not yet - BG/OBJ layers not enabled in DISPCNT yet (still 0x0040) |
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

[sp00nznet/gbarecomp@76c26af](https://github.com/sp00nznet/gbarecomp/commit/76c26af):
4. **ARM interpreter handles `MSR`/`MRS` before data-processing.** Hunting the IRQ spin further with LDR/LDM/DataProc-pc instrumentation surfaced the real root cause: `MSR CPSR_cf, r3` (`0xE129F003`) was being decoded as `TEQ` (opcode 9) by the data-processing handler because MSR's encoding overlaps with TST/TEQ/CMP/CMN-without-S. Then with Rd=15 the data-processing path dropped the XOR result into pc, producing the garbage `0x047A13BC` we'd been chasing. Moving the MSR/MRS checks above data-processing fixed it; LttP now runs 600+ frames cleanly through its IRQ handler.

[sp00nznet/gbarecomp@ab5cab7](https://github.com/sp00nznet/gbarecomp/commit/ab5cab7):
5. **`cpu_bx` falls back to the interpreter for ROM targets not in the dispatch table.** With a per-call-site tracer wired through `func_0800B084` (the game's main), I bisected the hang to a state-machine dispatcher at `func_0800D788` that calls handlers via a function-pointer table starting at ROM `0x084273EC`. The state-0 handler at `0x0800D890` is a trivial `MOV r0,#0; BX lr` but **gbarecomp's static analyzer never discovered it as a function entry** (it's only reached via the table). `cpu_bx(0x0800D891)` therefore fell through to `r[15] = target` with no execution; the dispatcher saw r0 still holding the function-pointer value (nonzero) and spun forever. Now `cpu_bx` routes ROM-not-in-table targets through `run_iwram_function`, so any leaf handler missed by analysis is still executed via the interpreter. Effect on LttP: main loop reaches ~3700 iter/sec and forced blank clears (DISPCNT 0x80 → 0x40).

[sp00nznet/gbarecomp@2c88f05](https://github.com/sp00nznet/gbarecomp/commit/2c88f05):
6. **EEPROM_V (8KB) emulation.** LttP polls `0x0D000000` for an EEPROM "ready" bit after issuing read/write commands; without emulation the runtime returned ROM-mirror data forever. Implemented the bit-serial GBATEK protocol: detection by `"EEPROM_V"` SDK marker scan, 8KB storage, state machine over `bus_read16`/`bus_write16` in the `0x0D...` region (`"11"`+addr+stop for reads, `"10"`+addr+data+stop for writes, idle reads return ready=1). Persists to a `.eep` file alongside the ROM. Replaces the local short-circuit hack from the previous session — `func_08135E74` now returns through the real DMA-then-poll cycle.

[sp00nznet/gbarecomp@cdace91](https://github.com/sp00nznet/gbarecomp/commit/cdace91):
7. **SIOMULTI[0..3] default = 0xFFFF (no link cable).** Found while tracing why LttP's outer state byte at IWRAM `0x03000BF0` never advances past 0. `func_0800C54C` reads a 32-bit value from `0x04000128` (SIOCNT+SIOMLT_SEND) plus an EWRAM control byte; with `io_regs` zero-init, it sees "link cable possibly connected, no idle pattern". Real hardware drives unconnected SIO pins high via pull-ups, so SIOMULTI reads as 0xFFFF. Set that as the runtime default. Didn't actually unblock LttP (the Four Swords state machine has more gates beyond SIO data), but matches hardware and is generically correct for every other GBA title.

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

The Four Swords link-cable state machine has at least 3 gates. With a SIOCNT-read override that forces bit 3 = 1 (a local hack, not pushed upstream), I drove the machine through the first one:

- **Gate 1 (cleared with SIOCNT.SD = 1 hack):** `func_0800C54C` state-0 path checks `(SIOCNT & 0x88) == 0x08`. Game writes `0x6003`; reading it back returns `0x6003` with bit 3 = 0. Override fixes this: state+1 advances 0→1, return value becomes 0x80.
- **Gate 2 (current block):** state-1 path checks `struct+2 != 0` before incrementing the counter at `struct+8`. struct+2 = `0x02030792` stays at 0; nothing in our run writes it. Probably set by an SIO recv interrupt handler that runs only when a remote unit actually transmits data.
- **Gate 3 (anticipated):** once struct+8 increments past 7, state advances to 2. That path is in `func_0800C5FC` → calls `func_0800C6A8`. Haven't disassembled it yet.

The real fix here is full SIO multiplayer handshake simulation: respond to SIOCNT writes that initiate transfers with synthesized "no remote unit detected → fall back to single-player" or "fake one connected slave" responses. That's days of work. For LttP specifically there's probably also a single-player code path triggered by a timer counter, but I haven't found it yet.

- **Function-pointer table discovery (broader gbarecomp improvement).** The cpu_bx fallback works but it's a *runtime* fix. The deeper improvement is for the analyzer to discover state-machine handler tables and recompile their entries — would eliminate interpreter overhead for hot dispatch paths.

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
