# My Setup - Chris's Development Environment

> **For LLMs:** This document contains exact paths and commands for my environment.
> Use this when working in my setup. For general instructions, see BUILD_GUIDE.md.

---

## Environment Overview

| Component | Value |
|-----------|-------|
| **Host OS** | Windows 11 |
| **Build Environment** | MSYS2/MinGW64 |
| **Compiler** | GCC 13 (MinGW-w64) |
| **AI Tool** | Claude Code (Opus 4.5) in PowerShell |
| **IDE** | VS Code |

### How It Works

```
┌──────────────────────────────────────────────────────────┐
│  Windows 11                                               │
│  ┌────────────────────┐    ┌──────────────────────────┐  │
│  │ PowerShell         │    │ MSYS2 MinGW64 Shell      │  │
│  │ (Claude Code runs  │    │ (Build commands)         │  │
│  │  here)             │    │                          │  │
│  │                    │    │ GCC, CMake, Make         │  │
│  │ Windows paths:     │    │ Unix paths:              │  │
│  │ C:\Users\User\...  │    │ /c/Users/User/...        │  │
│  └────────────────────┘    └──────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## Project Locations

### Main Projects

| Project | Windows Path | Description |
|---------|--------------|-------------|
| **sly1-recomp** | `C:\Users\User\Documents\decompilations\sly1-ps2-decomp\sly1-recomp` | This project (Sly-specific config) |
| **PS2Recomp** | `C:\Users\User\Documents\decompilations\sly1-ps2-decomp\PS2Recomp` | Recompiler tool (our fork) |
| **sly1-decomp** | `C:\Users\User\Documents\decompilations\sly1-ps2-decomp\sly1-decomp` | TheOnlyZac's matching decomp |

### GitHub Repos

| Repo | URL |
|------|-----|
| sly1-recomp | https://github.com/chrisking1981/sly1-recomp |
| PS2Recomp (fork) | https://github.com/chrisking1981/PS2Recomp |
| PS2Recomp (upstream) | https://github.com/ran-j/PS2Recomp |
| sly1-decomp | https://github.com/TheOnlyZac/sly1 |

---

## Path Conventions

### Windows Paths (PowerShell / Claude Code)
```
C:\Users\User\Documents\decompilations\sly1-ps2-decomp\
├── sly1-recomp\
├── PS2Recomp\
└── sly1-decomp\
```

### MSYS2/MinGW Paths (Build)
```
/c/Users/User/Documents/decompilations/sly1-ps2-decomp/
├── sly1-recomp/
├── PS2Recomp/
└── sly1-decomp/
```

### Important Files

| File | Windows Path |
|------|--------------|
| Game ELF | `C:\Users\User\Documents\decompilations\sly1-ps2-decomp\sly1-recomp\disc\SCUS_971.98` |
| Config TOML | `C:\Users\User\Documents\decompilations\sly1-ps2-decomp\sly1-recomp\config\sly1_config.toml` |
| Functions JSON | `C:\Users\User\Documents\decompilations\sly1-ps2-decomp\sly1-recomp\config\sly1_functions.json` |
| Recompiler | `C:\Users\User\Documents\decompilations\sly1-ps2-decomp\PS2Recomp\build\ps2xRecomp\ps2recomp.exe` |
| Runner | `C:\Users\User\Documents\decompilations\sly1-ps2-decomp\PS2Recomp\build\ps2xRuntime\ps2EntryRunner.exe` |

---

## Exact Build Commands

### Full Rebuild (PowerShell)

```powershell
cd C:\Users\User\Documents\decompilations\sly1-ps2-decomp\PS2Recomp\build

# Generate recompiled code
.\ps2xRecomp\Release\ps2recomp.exe ..\..\sly1-recomp\config\sly1_config.toml

# Rebuild
cmake --build . --config Release

# Run
.\ps2xRuntime\Release\ps2EntryRunner.exe ..\..\sly1-recomp\disc\SCUS_971.98
```

### Full Rebuild (MSYS2 MinGW64)

```bash
cd /c/Users/User/Documents/decompilations/sly1-ps2-decomp/PS2Recomp/build

# Generate recompiled code
./ps2xRecomp/ps2recomp ../../sly1-recomp/config/sly1_config.toml

# Rebuild
cmake --build . -j8

# Run
./ps2xRuntime/ps2EntryRunner ../../sly1-recomp/disc/SCUS_971.98
```

### Python Scripts (PowerShell)

```powershell
cd C:\Users\User\Documents\decompilations\sly1-ps2-decomp\sly1-recomp\scripts

# Analyze JAL returns
python analyze_jal_returns.py ..\disc\SCUS_971.98 ..\config\sly1_functions.json

# Fix delay slots
python fix_delay_slot_splits.py ..\disc\SCUS_971.98 ..\config\sly1_functions.json
```

### Git Commands (PowerShell)

```powershell
# sly1-recomp
cd C:\Users\User\Documents\decompilations\sly1-ps2-decomp\sly1-recomp
git add -A && git commit -m "message" && git push

# PS2Recomp (fork)
cd C:\Users\User\Documents\decompilations\sly1-ps2-decomp\PS2Recomp
git add -A && git commit -m "message" && git push
```

---

## LLM Checklist

When working in my environment:

### Use Windows Paths For:
- [x] File reads/writes (`C:\Users\User\...`)
- [x] PowerShell commands
- [x] Git operations

### Use MSYS2 Paths For:
- [x] Build commands in MSYS2 shell (`/c/Users/User/...`)
- [x] CMake operations (if using MinGW)

### Before Building:
- [ ] Check current directory
- [ ] Verify ELF exists at `disc/SCUS_971.98`
- [ ] Run recompiler to regenerate code if config changed

### After Code Changes:
1. Rebuild: `cmake --build . --config Release`
2. Test: Run `ps2EntryRunner`
3. Check output for "No func at 0x..." errors

### When Adding Functions:
1. Edit `config/sly1_functions.json`
2. Regenerate: Run recompiler
3. Rebuild: CMake build
4. Test: Run game

---

## Tools Installed

### MSYS2 Packages
```bash
pacman -S mingw-w64-x86_64-gcc
pacman -S mingw-w64-x86_64-cmake
pacman -S mingw-w64-x86_64-ninja
pacman -S git
```

### Python Packages
```powershell
pip install pyelftools
```

### GitHub CLI
```powershell
# Installed via winget
winget install GitHub.cli

# Authenticated as chrisking1981
gh auth status
```

---

## Common Issues in My Setup

### Problem: "command not found" in MSYS2
**Solution:** Use MinGW64 shell, not MSYS shell

### Problem: PowerShell escaping issues
**Solution:** Use single quotes or backticks for special characters

### Problem: Path too long errors
**Solution:** Enable long paths in Windows:
```powershell
# Run as admin
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1
```

### Problem: Git push fails
**Solution:** Check GitHub auth:
```powershell
gh auth status
gh auth login  # if needed
```

---

## Reference Decomp Access

When I need function names or code structure, check the decomp:

```powershell
# Open decomp folder
cd C:\Users\User\Documents\decompilations\sly1-ps2-decomp\sly1-decomp

# Search for function
grep -r "FunctionName" src/
```

Or browse: https://github.com/TheOnlyZac/sly1

---

*This document is for my specific setup. For general build instructions, see BUILD_GUIDE.md.*
