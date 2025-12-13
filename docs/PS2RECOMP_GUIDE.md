# PS2Recomp Guide

> Technical guide for using PS2Recomp - the PS2 static recompilation tool

---

## What is PS2Recomp?

PS2Recomp is a **static recompiler** that translates PS2 game code (MIPS R5900) to C++ that can run natively on PC.

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐
│ PS2 ELF     │ ──► │ PS2Recomp    │ ──► │ C++ code    │ ──► │ Native   │
│ (MIPS R5900)│     │ (translator) │     │ (generated) │     │ .exe     │
└─────────────┘     └──────────────┘     └─────────────┘     └──────────┘
     Game            Recompiler          ~30MB source        Windows/Linux
```

### Advantages over Emulation
- **Faster** - Native CPU instructions, no interpretation
- **Simpler** - No cycle-accurate CPU emulation needed
- **Moddable** - C++ source code is readable/modifiable

### Disadvantages
- **Per-game work** - Each game needs custom configuration
- **Incomplete** - Hardware must still be emulated (GS, DMA, etc.)
- **Complex** - Requires deep understanding of both platforms

### What PS2Recomp Actually Is

PS2Recomp is a **function execution framework**, NOT a full PS2 emulator. It provides:
- CPU instruction translation (MIPS → C++)
- Memory management (RDRAM, Scratchpad)
- Basic syscall handling
- File I/O

It does NOT provide:
- Graphics rendering (GS is stubbed)
- Audio (SPU2 not implemented)
- Real multi-threading
- Full hardware emulation

---

## Repository Structure

```
PS2Recomp/
├── ps2xRecomp/           # Main recompiler tool
│   ├── src/
│   │   ├── main.cpp              # Entry point
│   │   ├── ps2_recompiler.cpp    # Main recompilation logic
│   │   ├── r5900_decoder.cpp     # MIPS instruction decoder
│   │   ├── code_generator.cpp    # C++ code generation
│   │   └── elf_parser.cpp        # ELF file parsing
│   └── include/
│       └── ps2recomp/
│           ├── instructions.h    # All MIPS opcodes
│           ├── code_generator.h  # Translation declarations
│           └── types.h           # Data structures
│
├── ps2xRuntime/          # Runtime library (PS2 simulation)
│   ├── src/
│   │   ├── lib/
│   │   │   ├── ps2_runtime.cpp   # Core runtime + syscalls
│   │   │   ├── ps2_memory.cpp    # Memory management
│   │   │   └── ps2_stubs.cpp     # C library stubs
│   │   └── runner/
│   │       └── main.cpp          # Game runner entry point
│   └── include/
│       ├── ps2_runtime.h         # Core structures
│       └── ps2_runtime_macros.h  # MIPS operation macros
│
├── ps2xAnalyzer/         # ELF analysis tool
│   └── src/main.cpp              # Auto-generates config from ELF
│
└── CMakeLists.txt        # Build configuration
```

---

## Minimal Project Setup

For a new game, you need at minimum:

```
my-game-recomp/
├── config/
│   ├── game_config.toml      # Recompiler configuration
│   └── game_functions.json   # Function definitions
├── disc/
│   └── SLUS_XXX.XX           # Game ELF (from your disc)
└── output/                   # Generated code goes here
```

---

## Configuration File (TOML)

The config file tells PS2Recomp how to process your game:

```toml
[general]
# Input ELF file
input = "disc/SCUS_971.98"

# Output directory for generated C++
output = "output/"

# Put all functions in one file (faster compilation)
single_file_output = true

# Function definitions JSON
functions = "config/sly1_functions.json"

# Functions to stub (use C library instead)
stubs = [
    "printf", "sprintf", "vsprintf",
    "malloc", "free", "realloc",
    "memcpy", "memset", "memmove",
    "strlen", "strcpy", "strncpy", "strcmp",
    "fopen", "fclose", "fread", "fwrite"
]

# Functions to skip (not recompiled)
skip = [
    "abort", "exit", "_exit",
    "__start", "_ftext",
    "__do_global_dtors", "__do_global_ctors"
]
```

### Key Fields

| Field | Description |
|-------|-------------|
| `input` | Path to the game ELF file |
| `output` | Directory for generated C++ |
| `single_file_output` | `true` = one big file, `false` = file per function |
| `functions` | JSON file with function addresses/names |
| `stubs` | Functions to replace with C library calls |
| `skip` | Functions to not recompile |

---

## Functions JSON Format

The functions file defines all function entry points:

```json
[
  {
    "name": "_start",
    "address": "0x100008",
    "size": 92
  },
  {
    "name": "_InitSys",
    "address": "0x1f9358",
    "size": 1936
  },
  {
    "name": "entry_0x12345",
    "address": "0x12345",
    "size": 0
  }
]
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Function name (will be sanitized for C++) |
| `address` | Yes | Hex address of function start |
| `size` | Yes | Size in bytes (0 = auto-detect to next function) |

### Important Notes

1. **All JAL return addresses must be entry points** - When `JAL func` executes, it sets `$ra = PC + 8`. On return, that address must be a known function.

2. **Use size=0 for auto-detection** - The recompiler will extend the function to the next entry point.

3. **Name sanitization** - Reserved C++ names are automatically fixed:
   - `main` → `game_main`
   - `__name` → `fn___name`
   - `_Name` → `fn__Name`

---

## How Code Generation Works

### Input: MIPS Instruction
```mips
addiu $sp, $sp, -16    # Allocate stack frame
```

### Output: C++ Code
```cpp
SET_GPR_S32(ctx, 29, ADD32(GPR_U32(ctx, 29), 4294967280));
```

### Example Function Translation

**MIPS:**
```mips
my_func:
    addiu $sp, $sp, -16      # Allocate 16 bytes
    sw    $ra, 0($sp)        # Save return address
    jal   other_func         # Call other function
    nop                      # Delay slot
    lw    $ra, 0($sp)        # Restore return address
    jr    $ra                # Return
    addiu $sp, $sp, 16       # Restore stack (delay slot)
```

**Generated C++:**
```cpp
void my_func(uint8_t* rdram, R5900Context* ctx, PS2Runtime* runtime) {
    SET_GPR_S32(ctx, 29, ADD32(GPR_U32(ctx, 29), 4294967280));
    WRITE32(ADD32(GPR_U32(ctx, 29), 0), GPR_U32(ctx, 31));
    other_func(rdram, ctx, runtime);
    SET_GPR_U64(ctx, 31, READ64(ADD32(GPR_U32(ctx, 29), 0)));
    SET_GPR_S32(ctx, 29, ADD32(GPR_U32(ctx, 29), 16));
    ctx->pc = GPR_U32(ctx, 31); return;
}
```

---

## Functions That Must Be Stubbed

These functions should use native implementations instead of recompiled code:

### Memory Functions
```toml
stubs = ["malloc", "free", "realloc", "calloc"]
stubs = ["memcpy", "memset", "memmove", "memcmp"]
```

### String Functions
```toml
stubs = ["strlen", "strcpy", "strncpy", "strcmp", "strncmp"]
stubs = ["sprintf", "printf", "vsprintf"]
```

### Math Functions
```toml
stubs = ["sin", "cos", "tan", "sqrt", "pow", "fabs"]
```

### File I/O (if used)
```toml
stubs = ["fopen", "fclose", "fread", "fwrite", "fseek"]
```

---

## Runtime Capabilities (Detailed)

### ✅ What's Actually Implemented

| Category | Feature | Status |
|----------|---------|--------|
| **Memory** | 32MB RDRAM allocation | ✅ Works |
| **Memory** | 16KB Scratchpad | ✅ Works |
| **Memory** | 2MB IOP RAM | ✅ Works |
| **Memory** | Address translation with TLB | ✅ Works |
| **Memory** | Byte/Word/Quad-word read/write | ✅ Works |
| **File I/O** | fioOpen (host0:, cdrom0: mapping) | ✅ Works |
| **File I/O** | fioClose, fioRead, fioWrite | ✅ Works |
| **File I/O** | fioLseek (SEEK_SET/CUR/END) | ✅ Works |
| **Sync** | CreateSema, DeleteSema | ✅ Works |
| **Sync** | SignalSema, WaitSema, PollSema | ✅ Works |
| **Thread** | CreateThread (ID allocation only) | ⚠️ Partial |
| **Thread** | GetThreadId | ✅ Works |
| **Stdlib** | malloc, free, calloc, realloc | ✅ Works |
| **Stdlib** | memcpy, memset, memmove, memcmp | ✅ Works |
| **Stdlib** | strcpy, strcmp, strlen, strcat, etc. | ✅ Works |
| **CPU** | All 32 GPRs (128-bit) | ✅ Works |
| **CPU** | COP0 (System Control) | ✅ Works |
| **CPU** | COP1 (FPU) 32 float registers | ✅ Works |
| **CPU** | VU0 macro mode registers | ✅ Works |

### ❌ What's Stubbed/TODO

| Category | Feature | Status |
|----------|---------|--------|
| **Graphics** | GS register writes | ❌ Stub (prints only) |
| **Graphics** | Actual rendering | ❌ Not implemented |
| **DMA** | DMA transfers | ❌ Stub (prints only) |
| **DMA** | DMA chains/tags | ❌ Not implemented |
| **VU** | VU0/VU1 microprogram execution | ❌ Empty stub |
| **Thread** | Real multi-threading | ❌ Not implemented |
| **Thread** | SuspendThread, ResumeThread | ❌ TODO |
| **Sync** | Event Flags (8 functions) | ❌ TODO |
| **Sync** | Alarms (4 functions) | ❌ TODO |
| **IOP** | All SIF/RPC functions (10) | ❌ TODO |
| **Timer** | Interrupt handling | ❌ Stub |
| **Timer** | COP0 Count increment | ❌ Not implemented |
| **Audio** | SPU2 | ❌ Not even addressed |
| **Input** | Controller | ❌ Not implemented |

---

## Graphics: What Exists vs What's Missing

PS2Recomp has **infrastructure** for graphics but **no actual rendering**:

### ✅ What Already Exists

```cpp
// Data structures are defined in ps2_runtime.h:

struct GSRegisters {          // All 19 GS registers
    uint64_t pmode;           // Pixel mode
    uint64_t smode1, smode2;  // Sync modes
    uint64_t dispfb1, display1, dispfb2, display2;  // Display buffers
    uint64_t bgcolor, csr, imr;  // Background, status, interrupt mask
    // ... etc
};

struct VIFRegisters { ... };  // VIF0/VIF1 registers
struct DMARegisters { ... };  // 10 DMA channels
```

**Address ranges are recognized:**
- GS registers: `0x12000000 - 0x12001000`
- DMA registers: `0x10008000 - 0x1000F000`

**Raylib is included** as a dependency (`build/_deps/raylib-src`) - a graphics library ready for rendering.

**Syscalls exist:**
- `GsSetCrt()` - logs video mode (interlaced, NTSC/PAL)
- `GsGetIMR()` / `GsPutIMR()` - interrupt mask register

### ❌ What's Missing (The Hard Part)

| Component | What's Needed |
|-----------|---------------|
| **GS register logic** | Actually process writes, not just print them |
| **Framebuffer** | Allocate memory for pixels (640x448 or similar) |
| **GIF tag parsing** | Interpret drawing commands from DMA |
| **Primitive rendering** | Triangles, sprites, lines |
| **Texture handling** | Upload, sample, filter textures |
| **DMA transfers** | Move data from RAM to GS |
| **Raylib bridge** | Translate PS2 draw calls to Raylib/OpenGL |

### Current Flow (Broken)

```
Game writes to 0x12000000 (GS register)
         ↓
PS2Recomp: cout << "GS register write: 0x12000000 = ..."
         ↓
Nothing happens - no pixels, no framebuffer
```

### What Would Need to Happen

```
Game writes GS registers + DMA transfer
         ↓
PS2Recomp intercepts and parses GIF tags
         ↓
Extract triangles/textures from PS2 format
         ↓
Translate to Raylib draw calls
         ↓
Raylib renders to window
```

This is a significant undertaking - essentially writing a software PS2 GPU or HLE graphics layer.

---

## Runtime Architecture

### R5900 Context Structure

The CPU state is stored in a context structure:

```cpp
struct R5900Context {
    // 32 General Purpose Registers (128-bit each!)
    __m128i r[32];

    // Program Counter
    uint32_t pc;

    // Multiply/Divide results
    uint64_t hi, lo;      // Primary
    uint64_t hi1, lo1;    // Secondary (PS2-specific)

    // FPU Registers
    float f[32];
    uint32_t fcr31;

    // VU0 Registers (macro mode)
    __m128 vu0_vf[32];
    uint16_t vi[16];
};
```

### Run Loop

The game executes via a dispatch loop:

```cpp
while (!shouldExit) {
    auto func = lookupFunction(ctx->pc);
    if (func) {
        func(rdram, ctx, runtime);
    } else {
        printf("No func at 0x%x\n", ctx->pc);
        break;
    }
}
```

### Syscall Handling

PS2 syscalls are intercepted and emulated:

```cpp
void handleSyscall(R5900Context* ctx, uint8_t syscall_num) {
    switch (syscall_num) {
        case 0x3c: // SetupThread
            // Return stack pointer
            break;
        case 0x64: // FlushCache
            // No-op on PC
            break;
        // ... more syscalls
    }
}
```

---

## Build Process

### 1. Build PS2Recomp
```bash
cd PS2Recomp
mkdir build && cd build
cmake ..
cmake --build . --config Release
```

### 2. Generate Recompiled Code
```bash
./ps2xRecomp/ps2recomp ../my-game-recomp/config/game_config.toml
```

### 3. Build Runtime with Generated Code
```bash
cmake --build . --config Release
```

### 4. Run
```bash
./ps2xRuntime/ps2EntryRunner ../my-game-recomp/disc/GAME.ELF
```

---

## Common Problems & Solutions

### Problem: "No func at 0xXXXXXX"
**Cause:** Missing function entry point
**Solution:** Add the address to functions.json

### Problem: Compiler error about reserved identifier
**Cause:** Function name like `__name` or `main`
**Solution:** Already handled in our fork (sanitizeFunctionName)

### Problem: Stack corruption
**Cause:** SetupThread syscall returns wrong value
**Solution:** Must return stack pointer, not thread ID

### Problem: Delay slot not executing
**Cause:** Function boundary splits the delay slot
**Solution:** Run `fix_delay_slot_splits.py` script

### Problem: "undefined reference to \_mm\_set\_epi32"
**Cause:** Missing SSE flags
**Solution:** Add `-msse4.1` to compiler flags

---

## Tips for LLMs

When working with PS2Recomp:

1. **Always check ctx->pc** - This determines where execution continues
2. **JAL return = new entry point** - Every `JAL` creates a return address that needs an entry
3. **Delay slots are tricky** - The instruction after a branch executes BEFORE the branch
4. **128-bit registers** - PS2 has 128-bit GPRs, accessed via `__m128i`
5. **Syscalls are critical** - Wrong syscall implementation = crash
6. **Check the decomp** - Reference [TheOnlyZac/sly1](https://github.com/TheOnlyZac/sly1) for function names

---

## Reference

- **PS2Recomp repo:** [chrisking1981/PS2Recomp](https://github.com/chrisking1981/PS2Recomp) (fork with fixes)
- **Upstream:** [ran-j/PS2Recomp](https://github.com/ran-j/PS2Recomp)
- **N64Recomp (inspiration):** [Mr-Wiseguy/N64Recomp](https://github.com/Mr-Wiseguy/N64Recomp)

---

*This guide covers the PS2Recomp tool. See BUILD_GUIDE.md for build instructions and MY_SETUP.md for environment-specific details.*
