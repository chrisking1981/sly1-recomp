# Build Guide - Sly Cooper PS2 Recompilation

> General build instructions for any platform. For my specific setup, see MY_SETUP.md.

---

## Supported Platforms

| Platform | Status | Compiler |
|----------|--------|----------|
| Windows | Tested | MSVC, MinGW/GCC |
| Linux | Should work | GCC, Clang |
| macOS | Untested | Clang |

---

## Prerequisites

### Required Software

| Software | Version | Purpose |
|----------|---------|---------|
| Git | Any | Clone repositories |
| CMake | 3.16+ | Build system |
| C++ Compiler | C++17 | MSVC, GCC 9+, or Clang 10+ |
| Python | 3.8+ | Helper scripts |

### Required Files

| File | Source | Notes |
|------|--------|-------|
| Game ELF | Your PS2 disc | `SCUS_971.98` for Sly Cooper |

### Windows-Specific

Choose ONE of:
- **MSVC** - Visual Studio 2019/2022 with C++ workload
- **MinGW** - Via MSYS2 (`pacman -S mingw-w64-x86_64-gcc`)

### Linux-Specific

```bash
# Ubuntu/Debian
sudo apt install build-essential cmake git python3

# Fedora
sudo dnf install gcc-c++ cmake git python3

# Arch
sudo pacman -S base-devel cmake git python
```

---

## Build Steps

### 1. Clone Repositories

```bash
# Create working directory
mkdir ~/ps2-recomp && cd ~/ps2-recomp

# Clone this project (Sly-specific config)
git clone https://github.com/chrisking1981/sly1-recomp

# Clone PS2Recomp tool (fork with bug fixes)
git clone https://github.com/chrisking1981/PS2Recomp
```

### 2. Build PS2Recomp Tool

```bash
cd PS2Recomp
mkdir build && cd build

# Configure
cmake ..

# Build
cmake --build . --config Release
```

**Executables created:**
- `ps2xRecomp/ps2recomp` - The recompiler tool
- `ps2xRuntime/ps2EntryRunner` - The game runner
- `ps2xAnalyzer/ps2analyzer` - ELF analyzer

### 3. Copy Your Game ELF

Extract `SCUS_971.98` from your Sly Cooper disc and place it:

```bash
cp /path/to/SCUS_971.98 ~/ps2-recomp/sly1-recomp/disc/
```

### 4. Generate Recompiled Code

```bash
cd ~/ps2-recomp/PS2Recomp/build

# Run recompiler with Sly config
./ps2xRecomp/ps2recomp ../../sly1-recomp/config/sly1_config.toml
```

This generates ~30MB of C++ code in `sly1-recomp/output/`.

### 5. Rebuild with Generated Code

The generated code needs to be compiled into the runtime:

```bash
# Reconfigure to pick up generated code
cmake ..

# Rebuild
cmake --build . --config Release
```

### 6. Run

```bash
./ps2xRuntime/ps2EntryRunner ../../sly1-recomp/disc/SCUS_971.98
```

---

## Commands Reference

### One-liner: Full Rebuild

```bash
cd PS2Recomp/build && \
./ps2xRecomp/ps2recomp ../../sly1-recomp/config/sly1_config.toml && \
cmake --build . --config Release && \
./ps2xRuntime/ps2EntryRunner ../../sly1-recomp/disc/SCUS_971.98
```

### Just Recompile (no code regeneration)

```bash
cd PS2Recomp/build && cmake --build . --config Release
```

### Regenerate Functions JSON

```bash
cd sly1-recomp/scripts
python analyze_jal_returns.py ../disc/SCUS_971.98 ../config/sly1_functions.json
```

### Fix Delay Slot Boundaries

```bash
cd sly1-recomp/scripts
python fix_delay_slot_splits.py ../disc/SCUS_971.98 ../config/sly1_functions.json
```

---

## Project Structure

```
~/ps2-recomp/
├── PS2Recomp/                  # Recompiler tool
│   ├── build/                  # Build output
│   │   ├── ps2xRecomp/         # Recompiler executable
│   │   ├── ps2xRuntime/        # Runner executable
│   │   └── ps2xAnalyzer/       # Analyzer executable
│   ├── ps2xRecomp/src/         # Recompiler source
│   └── ps2xRuntime/src/        # Runtime source
│
└── sly1-recomp/                # Game-specific project
    ├── config/
    │   ├── sly1_config.toml    # Recompiler config
    │   └── sly1_functions.json # Function definitions
    ├── disc/
    │   └── SCUS_971.98         # Game ELF (you provide)
    ├── scripts/                # Python helpers
    ├── docs/                   # Documentation
    └── output/                 # Generated C++ (created by recompiler)
```

---

## Troubleshooting

### CMake can't find compiler

**Windows (MSVC):**
```cmd
# Open "Developer Command Prompt for VS 2022"
# Then run cmake from there
```

**Windows (MinGW):**
```bash
# Use MSYS2 MinGW64 shell, not MSYS2 MSYS shell
cmake -G "MinGW Makefiles" ..
```

### Compiler errors about \_\_m128i

Add SSE4.1 flag:
```bash
cmake -DCMAKE_CXX_FLAGS="-msse4.1" ..
```

### "No func at 0x..." at runtime

The address needs to be added to `sly1_functions.json`. Run:
```bash
python scripts/analyze_jal_returns.py disc/SCUS_971.98 config/sly1_functions.json
```

### Python script errors

Make sure you have the required packages:
```bash
pip install pyelftools
```

### Build takes forever

Enable parallel builds:
```bash
cmake --build . --config Release -j8
```

Or use `single_file_output = true` in config (already set for Sly).

---

## Platform-Specific Notes

### Windows with MSVC

```cmd
mkdir build && cd build
cmake -G "Visual Studio 17 2022" -A x64 ..
cmake --build . --config Release
```

### Windows with MinGW (MSYS2)

```bash
# Use MinGW64 shell
mkdir build && cd build
cmake -G "MinGW Makefiles" ..
cmake --build .
```

### Linux

```bash
mkdir build && cd build
cmake ..
cmake --build . -j$(nproc)
```

### Cross-compilation (Linux → Windows)

Not tested yet. Would need MinGW cross-compiler.

---

## Next Steps

After building successfully:

1. Read `PROJECT_STATUS.md` for current progress
2. Read `PS2RECOMP_GUIDE.md` for technical details
3. Check `AI_WORKFLOW.md` for development methodology

---

*For my specific environment and exact commands, see MY_SETUP.md*
