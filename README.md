# Sly Cooper - Static Recompilation Project

> Native PC port of Sly Cooper and the Thievius Raccoonus via static recompilation

[![PS2Recomp](https://img.shields.io/badge/tool-PS2Recomp-blue)](https://github.com/chrisking1981/PS2Recomp)
[![Game](https://img.shields.io/badge/game-Sly%20Cooper-orange)]()
[![AI Assisted](https://img.shields.io/badge/AI-Claude%20Opus%204.5-purple)]()

---

## New to Decomp/Recomp? Start Here!

**[GETTING_STARTED.md](docs/GETTING_STARTED.md)** - Learn decomp/recomp using AI as your learning partner

I built this project with **zero prior knowledge** of decompilation. Using AI (Claude Code) as a learning partner, I went from knowing nothing to finding and fixing critical bugs in a few hours.

**Why AI + Decomp/Recomp works so well:** Both are *deterministic* - you get clear, measurable output at every step. Does the code match? Does the game run? How many functions executed? AI can read crash logs, trace register values, and systematically debug. No expertise required - just curiosity and an AI assistant.

---

## What is This?

This project aims to run **Sly Cooper and the Thievius Raccoonus** (PS2) natively on PC using static recompilation. Instead of emulating the PS2 hardware, we translate the original MIPS machine code to C++ that runs directly on your computer.

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐
│ PS2 ELF     │ ──► │ PS2Recomp    │ ──► │ C++ code    │ ──► │ Native   │
│ (MIPS R5900)│     │ (translator) │     │ (generated) │     │ .exe     │
└─────────────┘     └──────────────┘     └─────────────┘     └──────────┘
```

## Project Status

| Component | Status | Notes |
|-----------|--------|-------|
| Function discovery | ✅ Done | 22,416 functions identified |
| Code generation | ✅ Done | All functions translated to C++ |
| Boot sequence | ✅ Works | _start → _InitSys → main |
| Memory management | ✅ Works | 32MB RDRAM, 16KB Scratchpad |
| File I/O | ✅ Works | fioOpen, fioRead, fioWrite, fioLseek |
| Syscalls | ⚠️ Partial | Core syscalls working, many TODO |
| Graphics (GS) | ❌ Stub | Struct exists, handlers only print debug |
| DMA | ❌ Stub | Struct exists, no actual transfer |
| Audio (SPU2) | ❌ None | Not implemented |

**Current progress:** Game executes 108+ function calls before hitting unimplemented GS/DMA hardware.

**Note:** PS2Recomp is a *function execution framework*, not a full emulator. CPU/memory/file I/O work, but graphics and audio require implementation.

## Documentation

See `docs/` directory:

| Document | Description |
|----------|-------------|
| `GETTING_STARTED.md` | **Start here!** Learn decomp/recomp with AI |
| `PROJECT_STATUS.md` | Current status, timeline, known issues |
| `PS2RECOMP_GUIDE.md` | How the recompiler works, configuration |
| `BUILD_GUIDE.md` | General build instructions (any platform) |
| `MY_SETUP.md` | My specific setup (for LLMs in my environment) |
| `AI_WORKFLOW.md` | AI-assisted development methodology |

**For LLMs**: Use `MY_SETUP.md` if working in my environment, or `BUILD_GUIDE.md` for general setup.

## Project Structure

```
sly1-recomp/
├── config/
│   ├── sly1_config.toml      # Recompiler configuration
│   └── sly1_functions.json   # 22,416 function definitions
├── scripts/
│   ├── analyze_jal_returns.py    # Find JAL return addresses
│   ├── fix_delay_slot_splits.py  # Fix function boundaries
│   └── iterate_fixes.py          # Automated test loop
├── docs/
│   ├── PROJECT_STATUS.md         # Project status & timeline
│   ├── PS2RECOMP_GUIDE.md        # Technical recompiler guide
│   ├── BUILD_GUIDE.md            # General build instructions
│   ├── MY_SETUP.md               # My specific environment
│   ├── AI_WORKFLOW.md            # AI-assisted workflow
│   └── backup/                   # Previous documentation
└── output/                   # Generated code goes here
```

## Related Projects

| Project | Description | Link |
|---------|-------------|------|
| **PS2Recomp** | The recompilation tool (our fork with fixes) | [chrisking1981/PS2Recomp](https://github.com/chrisking1981/PS2Recomp) |
| **sly1-decomp** | Matching decompilation project | [TheOnlyZac/sly1](https://github.com/TheOnlyZac/sly1) |
| **Original PS2Recomp** | Upstream tool | [ran-j/PS2Recomp](https://github.com/ran-j/PS2Recomp) |

## Prerequisites

- **PS2Recomp** - Clone from [chrisking1981/PS2Recomp](https://github.com/chrisking1981/PS2Recomp)
- **Sly Cooper ELF** - Extract `SCUS_971.98` from your own game disc
- **Build tools** - CMake, MinGW/GCC or MSVC
- **Python 3** - For helper scripts

## Quick Start

### 1. Clone the repos

```bash
cd ~/decompilations  # or wherever you keep projects

# This repo (Sly-specific config)
git clone https://github.com/chrisking1981/sly1-recomp

# PS2Recomp tool
git clone https://github.com/chrisking1981/PS2Recomp
```

### 2. Build PS2Recomp

```bash
cd PS2Recomp
mkdir build && cd build
cmake ..
cmake --build . --config Release
```

### 3. Copy your game ELF

Copy `SCUS_971.98` from your Sly Cooper disc to `sly1-recomp/disc/`

### 4. Generate recompiled code

```bash
cd PS2Recomp
./build/ps2xRecomp/ps2recomp ../sly1-recomp/config/sly1_config.toml
```

### 5. Build and run

```bash
cd build
cmake --build . --config Release
./ps2xRuntime/ps2EntryRunner ../sly1-recomp/disc/SCUS_971.98
```

## Bugs We Fixed in PS2Recomp

During testing, we discovered and fixed several critical bugs in PS2Recomp:

1. **`jr $ra` not setting `ctx->pc`** - Run loop dispatch was broken
2. **SetupThread syscall** - Returned thread ID instead of stack pointer
3. **Reserved C++ identifiers** - `__is_pointer`, `main`, etc. not sanitized
4. **Delay slot handling** - Split across function boundaries
5. **Missing syscalls** - GetSyscallHandler, SetSyscall, etc.

See [docs/backup/ps2recomp-bug-report.md](docs/backup/ps2recomp-bug-report.md) for details.

## AI-Assisted Development

This project was developed using **Claude Code with Opus 4.5** for AI-assisted reverse engineering. The AI helped with:

- Systematic debugging and crash analysis
- Tracing register values and execution flow
- Researching PS2 SDK documentation
- Generating fix code and test scripts
- Creating documentation

See [docs/AI_WORKFLOW.md](docs/AI_WORKFLOW.md) for the full workflow.

## Contributing

Contributions welcome! Areas that need work:

- [ ] Graphics Synthesizer (GS) emulation
- [ ] DMA controller implementation
- [ ] Audio (SPU2) support
- [ ] Controller input
- [ ] More syscall implementations

## License

This project contains no copyrighted game code or assets. You must provide your own legally obtained copy of Sly Cooper.

## Credits

- **Chris King** ([@chrisking1981](https://github.com/chrisking1981)) - Recompilation project & bug fixes
- **ran-j** - Original [PS2Recomp](https://github.com/ran-j/PS2Recomp) tool
- **TheOnlyZac** - [Sly Cooper decompilation](https://github.com/TheOnlyZac/sly1) project
- **Claude (Anthropic)** - AI-assisted development

---

*This is an experimental project. The goal is educational - understanding how PS2 games work and exploring static recompilation techniques.*
