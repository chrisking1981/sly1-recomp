# Project Status - Sly Cooper PS2 Recompilation

> **Last Updated:** December 2024
> **Game:** Sly Cooper and the Thievius Raccoonus (SCUS_971.98)
> **Platform:** PS2 → PC via Static Recompilation

---

## Quick Status

| Component | Status | Notes |
|-----------|--------|-------|
| Function discovery | ✅ Done | 22,416 functions identified |
| Code generation | ✅ Done | All functions translated to C++ |
| Boot sequence | ✅ Works | `_start` → `_InitSys` → `main` |
| Syscalls | ⚠️ Partial | Core syscalls working (SetupThread, SetupHeap, etc.) |
| Graphics (GS) | ❌ TODO | Graphics Synthesizer emulation needed |
| Audio (SPU2) | ❌ TODO | Sound Processing Unit not implemented |
| DMA | ❌ TODO | Direct Memory Access controller needed |
| Input | ❌ TODO | Controller input not implemented |

**Current progress:** Game executes **108+ function calls** before hitting unimplemented hardware.

---

## Timeline / Commits

| Date | Milestone | Details |
|------|-----------|---------|
| Dec 2024 | Initial testing | Started testing PS2Recomp with Sly Cooper |
| Dec 2024 | Bug #1 fixed | `jr $ra` not setting `ctx->pc` |
| Dec 2024 | Bug #2 fixed | SetupThread syscall returning wrong value |
| Dec 2024 | Bug #3 fixed | Reserved C++ identifiers not sanitized |
| Dec 2024 | Bug #4 fixed | Delay slots split across function boundaries |
| Dec 2024 | Bug #5 fixed | Missing syscalls (0x5A/0x5B, 0x74) |
| Dec 2024 | JAL analysis | Added 18,111 JAL return addresses as entry points |
| Dec 2024 | PR submitted | [PR #3](https://github.com/ran-j/PS2Recomp/pull/3) to upstream PS2Recomp |

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

### Boot Sequence (Currently Working)

```
✅ _start (0x100008)
✅ BSS clearing loop
✅ SetupThread syscall → SP = 0x64C700
✅ SetupHeap syscall → heap configured
✅ _InitSys
✅ supplement_crt0
✅ GetSystemCallTableEntry
✅ InitAlarm
✅ InitThread
✅ InitExecPS2
✅ Syscall table registration (8 syscalls)
✅ FlushCache
⏳ main() entry... → stops after 108 calls
```

---

## Known Issues

### Current Blocker
- **Missing hardware emulation**: Game tries to access GS (Graphics Synthesizer) and DMA registers
- Last PC before crash: `0x185bc8`
- Error: "No func at 0x185bc8" (missing function entry point)

### Workarounds Applied
1. **JAL return addresses**: Script adds all 18,111 return addresses as function entries
2. **BIOS handler stubs**: Addresses >= 0x80000000 are stubbed instead of crashing
3. **Reserved identifier sanitization**: `__name` → `fn___name`, `main` → `game_main`

---

## Next Steps

### Priority 1: Hardware Emulation
- [ ] **Graphics Synthesizer (GS)** - Most critical, enables visuals
- [ ] **DMA Controller** - Needed for data transfers
- [ ] **VIF (Vector Interface)** - Unpacks data for VU

### Priority 2: More Syscalls
- [ ] Threading syscalls (CreateThread, StartThread, etc.)
- [ ] Semaphore syscalls (full implementation)
- [ ] Timer/Alarm syscalls

### Priority 3: I/O
- [ ] Controller input
- [ ] Audio (SPU2)
- [ ] Memory card

---

## Project Structure

```
sly1-recomp/
├── config/
│   ├── sly1_config.toml      # Recompiler configuration
│   └── sly1_functions.json   # 22,416 function definitions
├── scripts/
│   ├── analyze_jal_returns.py    # Find JAL return addresses
│   ├── fix_delay_slot_splits.py  # Fix function boundaries
│   ├── add_entry.py              # Add single entry point
│   └── iterate_fixes.py          # Automated test loop
├── docs/
│   ├── PROJECT_STATUS.md         # This file
│   ├── PS2RECOMP_GUIDE.md        # How the recompiler works
│   ├── BUILD_GUIDE.md            # General build instructions
│   ├── MY_SETUP.md               # My specific environment
│   ├── AI_WORKFLOW.md            # AI-assisted development
│   └── backup/                   # Previous documentation
└── output/                       # Generated code output
```

---

## Reference Resources

### Related Projects

| Project | Description | Link |
|---------|-------------|------|
| **PS2Recomp (fork)** | Our fork with bug fixes | [chrisking1981/PS2Recomp](https://github.com/chrisking1981/PS2Recomp) |
| **PS2Recomp (upstream)** | Original tool | [ran-j/PS2Recomp](https://github.com/ran-j/PS2Recomp) |
| **sly1-decomp** | Matching decompilation | [TheOnlyZac/sly1](https://github.com/TheOnlyZac/sly1) |
| **N64Recomp** | Inspiration project | [Mr-Wiseguy/N64Recomp](https://github.com/Mr-Wiseguy/N64Recomp) |

### Documentation

- [PS2Tek](https://psi-rockin.github.io/ps2tek/) - PS2 hardware documentation
- [PS2 EE Syscalls](https://israpps.github.io/ps2tek/PS2/BIOS/EE_Syscalls.html) - Syscall reference
- [PS2SDK](https://github.com/ps2dev/ps2sdk) - PS2 development kit
- [PCSX2](https://pcsx2.net/) - PS2 emulator (reference implementation)

---

## Statistics

| Metric | Value |
|--------|-------|
| Total functions | 22,416 |
| Original functions | 4,682 |
| JAL return addresses added | 18,111 |
| Function calls executed | 108+ |
| Bugs found & fixed | 5 critical |
| Generated C++ code | ~30 MB |

---

*This document is updated as progress is made on the recompilation project.*
