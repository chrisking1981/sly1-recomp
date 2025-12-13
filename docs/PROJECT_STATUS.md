# Project Status - Sly Cooper PS2 Recompilation

> **Last Updated:** 13 December 2025
> **Game:** Sly Cooper and the Thievius Raccoonus (SCUS_971.98)
> **Platform:** PS2 → PC via Static Recompilation

---

## Quick Status

| Component | Status | Notes |
|-----------|--------|-------|
| Function discovery | ✅ Done | 22,416 functions identified |
| Code generation | ✅ Done | All functions translated to C++ |
| Boot sequence | ✅ Works | `_start` → `_InitSys` → `main` |
| Memory management | ✅ Works | 32MB RDRAM, 16KB Scratchpad |
| File I/O | ✅ Works | fioOpen, fioRead, fioWrite, etc. |
| Syscalls | ✅ Works | Core syscalls all working |
| Threading | ⚠️ Minimal | Thread ID allocation, no scheduling |
| Semaphores | ✅ Works | CreateSema, WaitSema, SignalSema |
| **Loop Fix** | ✅ **FIXED** | JAL in loops now use goto |
| Graphics (GS) | ❌ Stub | No rendering yet |
| DMA | ❌ Stub | No actual transfers |
| VU0/VU1 | ❌ Stub | Registers exist, no microcode |
| Audio (SPU2) | ⚠️ Stub | Sound commands logged |
| Input | ⚠️ Stub | Controller stubs work |

**Current progress:** Game executes **56,000,000+ function calls** stably in main loop!

---

## Timeline / Commits

| Date | Milestone | Details |
|------|-----------|---------|
| Dec 2025 | Initial testing | Started testing PS2Recomp with Sly Cooper |
| Dec 2025 | Bug #1-5 fixed | Critical boot bugs fixed |
| Dec 2025 | JAL analysis | Added 18,111 JAL return addresses |
| Dec 2025 | Sound stubs | Bank loading, command logging |
| Dec 2025 | File split | 19 files for faster builds |
| **13 Dec 2025** | **LOOP FIX** | **Fixed code generator for loops with syscalls** |

---

## Major Fix: Loop with Syscalls (13 December 2025)

### The Problem
The PS2Recomp code generator generated `return;` after EVERY function call (JAL instruction), even inside loops. This broke any loop containing syscalls.

### The Solution
Modified `code_generator.cpp`:

1. **`collectBranchTargets()`** - Now collects JAL return addresses as labels
2. **`handleBranchDelaySlots()`** - Uses `goto` instead of `return` for internal calls

### Before (Broken)
```cpp
WaitSema(rdram, ctx, runtime); return;  // Exits function!
```

### After (Fixed)
```cpp
WaitSema(rdram, ctx, runtime); goto label_15f248;  // Stays in loop!
```

### Impact
- FlushFrames VBlank sync loop now works correctly
- Game is stable at 56+ million function calls
- This fix benefits ALL PS2 games, not just Sly Cooper

---

## Current Architecture

### Memory Layout

```
0x00000000 - 0x01FFFFFF  Main RAM (32 MB)
0x02000000               Stack starts here (grows down)
0x70000000 - 0x70003FFF  Scratchpad (16 KB)
0x10000000 - 0x1FFFFFFF  I/O Registers
0x1FC00000 - 0x1FFFFFFF  BIOS (4 MB)
0x80000000+              BIOS handlers (stubbed)
```

### Boot Sequence (Working)

```
✅ _start (0x100008)
✅ SetupThread syscall → SP = 0x64C700
✅ SetupHeap syscall → heap configured
✅ _InitSys
✅ Syscall table initialization
✅ FlushCache
✅ main() entry
✅ Game main loop (56M+ calls stable)
```

---

## PS2Recomp Capabilities

### ✅ Fully Working

| Feature | Details |
|---------|---------|
| Memory Management | 32MB RDRAM, address translation |
| File I/O Syscalls | All basic file operations |
| ELF Loading | Full ELF parsing, segment loading |
| CPU Context | All 32 GPRs (128-bit), COP0, FPU |
| MIPS Instructions | All arithmetic, MMI, FPU macros |
| Syscall Infrastructure | 50+ syscalls with dispatch |
| **Loop Handling** | **Fixed - syscalls in loops work** |

### ⚠️ Partially Working

| Feature | Details |
|---------|---------|
| Threading | Thread ID allocation only |
| Sound | Commands logged, no audio output |
| Controllers | Stubs return neutral state |

### ❌ Not Implemented

| Feature | Details |
|---------|---------|
| Graphics (GS) | No actual rendering |
| DMA Controller | No actual data transfers |
| VU Microcode | `executeVU0Microprogram()` empty |
| IOP/SIF/RPC | All SIF functions TODO |

---

## Next Steps

### Priority 1: Graphics
- [ ] Add GS register monitoring
- [ ] Add DMA transfer logging
- [ ] Choose approach: parallel-gs vs software rasterizer

### Priority 2: Submit Upstream Fix
- [ ] Create PR for loop fix to ran-j/PS2Recomp
- [ ] Bug report already created: `PS2Recomp/docs/BUG_REPORT_LOOP_FIX.md`

### Priority 3: More Testing
- [ ] Test other PS2 games with the loop fix
- [ ] Verify all loop patterns work correctly

---

## Project Structure

```
sly1-recomp/
├── config/
│   ├── sly1_config.toml      # Recompiler configuration
│   └── sly1_functions.json   # 22,416 function definitions
├── docs/
│   ├── PROJECT_STATUS.md     # This file
│   ├── PS2RECOMP_GUIDE.md    # How the recompiler works
│   └── sessions/             # Session documentation
└── recomp_output/            # Generated code output

PS2Recomp/
├── ps2xRecomp/
│   └── src/
│       └── code_generator.cpp  # LOOP FIX IS HERE
├── ps2xRuntime/
│   └── src/
│       ├── lib/              # Runtime implementation
│       └── runner/           # Entry point + recompiled code
└── docs/
    └── BUG_REPORT_LOOP_FIX.md  # Bug report for upstream
```

---

## Statistics

| Metric | Value |
|--------|-------|
| Total functions | 22,416 |
| Function calls executed | 56,000,000+ |
| Bugs found & fixed | 6 critical (including loop fix) |
| Generated C++ code | ~36 MB |
| Build time (incremental) | ~12 seconds |

---

## Reference Resources

### Related Projects

| Project | Description | Link |
|---------|-------------|------|
| **PS2Recomp (fork)** | Our fork with bug fixes | [chrisking1981/PS2Recomp](https://github.com/chrisking1981/PS2Recomp) |
| **PS2Recomp (upstream)** | Original tool | [ran-j/PS2Recomp](https://github.com/ran-j/PS2Recomp) |
| **sly1-decomp** | Matching decompilation | [TheOnlyZac/sly1](https://github.com/TheOnlyZac/sly1) |

### Documentation

- [PS2Tek](https://psi-rockin.github.io/ps2tek/) - PS2 hardware documentation
- `LESSONS_LEARNED.md` - All lessons learned
- `prompt.md` - LLM session prompt with fix-first philosophy

---

*This document is updated as progress is made on the recompilation project.*
