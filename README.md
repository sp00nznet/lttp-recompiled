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

[sp00nznet/gbarecomp@26522a1](https://github.com/sp00nznet/gbarecomp/commit/26522a1):
8. **Minimal SIO multiplayer transfer simulation.** On SIOCNT write with start bit (bit 7) high, immediately simulate transfer completion: SIOMULTI[0] = whatever the parent had in SIOMLT_SEND, SIOMULTI[1..3] = 0xFFFF (idle), clear start bit, set error bit (no slaves), raise SIO IRQ if enabled. Games that gate boot on "did a transfer complete and was there a slave" can now fall through to single-player cleanly.

[sp00nznet/gbarecomp@0a6908d](https://github.com/sp00nznet/gbarecomp/commit/0a6908d):
9. **ARM interpreter: data-processing no longer eats halfword/multiply encodings.** Real ARM v4 bug fix. The check `(insn & 0x0C000000) == 0` matched data-processing register form, **plus** all halfword transfers (`LDRH`/`STRH`/`LDRSB`/`LDRSH`) and multiply (`MUL`/`MLA`/etc.), because they all share bits 27-26 = 00. They're distinguished by bits 7 and 4 — but the data-processing check didn't look at those. So `STRH r0, [r3, #2]` (used by the cart's IRQ trampoline at ROM `0x080001B0` to ack IF after delivery) was being interpreted as a `BIC`-class data-processing op and never reached `bus_write16`'s write-1-to-clear path. IF bits stayed pinned high; IRQs delivered every scanline boundary in a storm. Fixed by gating data-processing on `!halfword && !multiply`. With this fix, LttP's IF acks land properly and the IRQ pacing matches real hardware — visible in the I/O trace as the DISPSTAT VCount-match bit toggling cleanly per VCount IRQ. This was a critical interpreter correctness bug affecting any game whose IRQ handler runs in copied-to-EWRAM code (almost all of them — gbas commonly relocate hot IRQ paths to RAM for speed).

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

LttP+Four Swords has at least three independent gates between "main loop running" and "title screen drawing":

- **Attract-loop guard at `func_0800B2CC`** — checks `KEYINPUT & 0xF == 0xF` (A+B+SELECT+START all released). If so, main loop branches back to label_0800B086 (top), re-running all four init functions every iteration. Cleared by simulating any one of those four buttons pressed. With START held the trace confirms `r0=0, KEYINPUT=0x03F7, state=0x00000080` — the chain works. But this is a regular user-input gate, not a hack — the original cart shows attract animation until input.
- **Gate 1 — SIOCNT.SD bit at `func_0800C54C`** — `(SIOCNT & 0x88) == 0x08`: bit 3 (Slave Detect "chain unbroken") must read 1. Game writes `0x6003` to SIOCNT during init; without the read-side override the readback has bit 3 = 0, function returns 0, outer state byte at IWRAM `0x03000BF0` never moves. Local override ORs in 0x08 on SIOCNT reads → state advances 0→0x80.
- **Gate 2 — struct+2 OR struct+3 (the real wall).** State-1 path checks `struct+2 != 0`; struct+2 is set only by `func_0800C6A8`'s tail: `struct+2 |= struct+3`. struct+3 has a bit set per "detected player slot" (4 slots) only when a 12-halfword sum from a per-slot table at `[struct+0x2C + slot*0x1C]` equals `-15` (signed). That's a checksum over received link-cable data: real-hardware SIO transfers fill the table with peer GBA data; with no peers and no SIO transfer simulation, the table is zero, sum is 0, never matches -15, struct+3 never gets a bit set, struct+2 stays 0, machine stays at state 1.
- **No timeout fallback found.** Counter at struct+B increments every call but wraps at 255 — no length-gated branch to a single-player path. Either the timeout is elsewhere, or the original cart genuinely refuses to boot LttP without seeing a coherent SIO bus.

### Full RE map of the Four Swords link-cable boot

After deep tracing, the protocol is mapped end-to-end. It's a multi-stage state machine driven by an SIO IRQ that the cart never triggers on its own without an external sync.

**The struct at EWRAM `0x02030790`** is the SIO multiplayer control block. Key fields:

| Offset | Purpose |
|---|---|
| +1 | Inner state byte (state machine in `func_0800C54C`) |
| +2 | Checksum result accumulator (`\|= struct+3` in `func_0800C6A8` tail) |
| +3 | Per-slot "detected" bits (set when a slot's 12-halfword sum == -15) |
| +5 | Sequence flag (set when `struct+0x18 == 0xB` during an SIO IRQ) |
| +0x14 | Error-path counter |
| +0x18 | Slot-write offset (initialized to 0xD, decremented to -1 on sync, then walks 0..0xB) |
| +0x1C | Pointer to per-slot tables region 0 |
| +0x20 | Pointer to per-slot tables region 1 |
| +0x24 | "Current write" buffer pointer (swaps with +0x28 on sync) |
| +0x28 | "Last written" buffer pointer |
| +0x2C | "Read for checksum" buffer pointer (swap of +0x28 after rotate fn) |

Verified at runtime: `struct+0x24` = 0x020307F8, `struct+0x2C` alternates between 0x020308D8 and 0x02030868 (double-buffered).

**Init path:** the SIO struct is initialized by code at ROM `0x0800C476-0x0800C494` (called once during boot). It writes `struct+0x14 = struct+0x18 = 0xD` and sets the five pointer fields to layouts within the same struct (slot region at `struct+0x148`).

**SIO IRQ handler at ROM `0x0800C7E8`** gets COPIED to EWRAM `0x02030590` during init (verified: subhandler[0] in IWRAM `0x03000B70` points to `0x02030591`, and a byte dump of EWRAM matches the ROM source). The handler runs through gbarecomp's interpreter on EWRAM targets.

Protocol decode:
- **Sync packet**: `SIOMULTI[0] == 0xFEFE`. Resets `struct+0x18` to -1, swaps `+0x24/+0x28` buffers, sets BIOS IF mirror bit 7. Then slot-copies the 4 SIOMULTI values into the new write buffer at slot-offset `struct+0x18 * 2 = -2`.
- **Data packet**: `SIOMULTI[0] != 0xFEFE`. Writes next halfword of a per-frame response buffer at `struct+0x20[counter*2]` into `SIOMLT_SEND` (so master's next transfer carries response). Increments `struct+0x14`. Then slot-copies SIOMULTI into the current write buffer at slot-offset `struct+0x18 * 2`.
- After 12 data packets, `struct+0x18` reaches 0xB → sets `struct+5 = 1`.

**The checksum gate at `func_0800C6A8`:**
1. Calls a function pointer at `[0x02030950]` (which is itself a copy of ROM `0x08000318`, an ARM "rotate buffers" function that swaps `+0x28 ↔ +0x2C` and returns the OLD value of `struct+5`).
2. If returned value is 0 → skip everything, exit. So `struct+5` must have been set to 1 by the SIO handler before `C6A8` runs.
3. Otherwise: for each of 4 slots, sum the 12 halfwords. If sum == -15 (signed 16-bit), set bit N in `struct+3`.
4. Tail: `struct+2 |= struct+3`.

**The gate at `func_0800C54C` state-1:** checks `struct+2 != 0`. If yes, increments `struct+8`; when `struct+8 > 7`, state advances to 2.

**The critical chain (all must be true for state to advance):**
1. Sync packet must arrive (resets state, swaps buffers)
2. 12 data packets must follow (fills the write buffer)
3. Final data packet must bring `struct+0x18` to 0xB → sets `struct+5 = 1`
4. After the buffer swap from next sync, `func_0800C6A8` runs the checksum
5. At least one slot's 12 halfwords sum to -15 → `struct+3` gets a bit set
6. `struct+2 |= struct+3` → gate 2 opens

**Why "force struct+2 = 1" doesn't work:** the state machine has dozens of other field dependencies after gate 2 — `func_0800B524` (VBlank wait), `func_0800D788` (sub-dispatcher), and downstream functions all read additional state set up by the proper protocol path. Bypassing gate 2 alone produces incorrect downstream state.

### What "full SIO peer simulation" means concretely

To execute the chain above, the runtime needs to play the role of a remote GBA in the multiplayer chain:

1. **Detect when the cart enters MultiPlayer mode** (SIOCNT bits 12-13 == 10, IRQE bit 14 = 1).
2. **Periodically inject a transfer-complete event** even though the cart isn't writing `SIOCNT.start = 1` (verified: the cart doesn't initiate; it expects to *receive* sync from a parent).
3. **Drive a 1-sync + 12-data packet cycle:**
   - On sync injection: `SIOMULTI[0] = 0xFEFE`, others = 0xFFFF.
   - On data injections: `SIOMULTI[0] = arbitrary != 0xFEFE`, `SIOMULTI[1..3]` crafted so that across 12 packets the per-slot 12-halfword sum equals -15 for slot 1 (e.g. 11× 0xFFFF + 1× 0xFFFC).
4. **Fire SIO IRQ after each injection** with IF bit 7, BIOS IF mirror bit 7, and IE bit 7 forced.
5. **Handle the back-channel:** after each non-sync data packet the cart's handler writes a response halfword to `SIOMLT_SEND`. A real peer would consume this; we ignore it (the cart is just talking to itself).

That's a couple of hundred lines of state-machine code in `runtime.c`, plus enough RE on the cart's *downstream* code paths (after gate 2 opens) to confirm the rest of init proceeds. Realistic estimate: 1-2 days of focused work to a verified title screen.

### SIO peer simulator built in `runtime.c` (latest session)

The full simulation has been written and verified working end-to-end except for the very final step. Implemented as a per-IRQ data injection inside `check_interrupts`:

- 13-packet cycle (1 sync + 12 data) state machine
- Sync packet: `SIOMULTI[0] = 0xFEFE`, others = `0xFFFF`
- Data packets: `SIOMULTI[0] = 0x0000`, `SIOMULTI[1]` = `0xFFFF` for first 11 + `0xFFFC` for packet 12 → slot 1 sum = `-15`
- One IF bit 7 set per frame; hard-cleared after each delivery (works around an apparent ARM interpreter bug where the trampoline's `STRH r0, [r3, #2]` at ROM `0x080001B0` doesn't actually trigger `bus_write16`'s write-1-to-clear path)

**Verified at runtime:**
- SIO IRQ delivery fires per-frame as expected
- Trampoline correctly dispatches to EWRAM SIO handler (byte-for-byte match with ROM source)
- `struct+0x18` cycles correctly: `-1 → 0 → 1 → ... → 0xB → 0xC → -1` per 13-packet cycle
- `struct+5 = 1` is set at exactly the right moment (data packet 12 when struct+0x18 = 0xB)
- `struct+0x24` (write buffer ptr) swaps on every sync packet, exactly per protocol
- Slot 1 region in the write buffer gets filled with `[0xFFFF * 11, 0xFFFC]` — sum = -15 ✓

**Remaining blocker — buffer rotation alignment:**
The checksum at `func_0800C6A8` reads from `struct+0x2C` (the read buffer). Even with the gate patched open, all four slot sums read as `0`. The rotate function in EWRAM swaps `struct+0x28`/`struct+0x2C` on every C6A8 call; with main loop running ~3700 iter/sec and SIO IRQ at 60Hz, the parity of swaps between when slot data is written and when checksum reads is wrong. Two issues to chase:
1. **Buffer rotation alignment**: trace which buffer `struct+0x2C` points at when struct+5 is set, vs which buffer holds the just-filled slot data. If different, find the protocol-correct moment for checksum to read.
2. ~~**ARM interpreter STRH→IF bug**~~ → **FIXED upstream as commit `0a6908d`.** The interpreter's data-processing check was inadvertently matching halfword transfer and multiply encodings (which share bits 27-26 = 00 with data-processing). `STRH r0, [r3, #2]` in the trampoline was being decoded as `BIC`-style data-processing instead. Fixed by gating the data-processing match on `!(halfword) && !(multiply)`. Critical generic interpreter correctness fix - affects any game with copied-to-RAM IRQ handlers (which is most of them).

The right next session can pick up at "find which one of these is the alignment fix" — should be a quick experiment with one of: (a) only swap buffers when struct+5 was 1 (mimic real game cycle timing), (b) call C6A8 only N times per frame instead of every iter, (c) re-read the rotate function's exact semantics.

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
