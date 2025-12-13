# Next Steps: Systematische Aanpak naar Graphics

> **Status:** 13 December 2025 (Updated - Main Loop Verified!)
> **Doel:** Graphics in beeld krijgen voor Sly Cooper
> **Repository:** https://github.com/chrisking1981/sly1-recomp

---

## Huidige Situatie

```
┌─────────────────────────────────────────────────────────────────┐
│ PS2Recomp Executie Status                                       │
├─────────────────────────────────────────────────────────────────┤
│ Boot sequence:    ✅ Werkt (_start → SetupThread → main entry)  │
│ Functies executed: 54,000,000+ calls in 15 sec (stabiel!)       │
│ CD-ROM:           ✅ ISO reading geïmplementeerd                │
│ Game State Loop:  ✅ Bypassed (0x2701b8 patched)                │
│ Sound/IOP:        ✅ Gestubbed + WAV playback via Raylib        │
│ Main Loop:        ✅ Game draait in main() @ 0x1857c8           │
│ FlushFrames:      ✅ FIXED in PS2Recomp code generator!         │
│ MPEG Video:       ✅ Correct skipped (branch delay slot fix)    │
│ Branch Delay:     ✅ FIXED in PS2Recomp code generator!         │
│ Frame Rendering:  ✅ OpenFrame/CloseFrame worden aangeroepen    │
│ DMA Transfers:    ⚠️ Code draait, maar geen hardware emulatie   │
│ Graphics:         ❌ GS emulatie ontbreekt - geen pixels        │
│ Build:            ✅ Split files (19 bestanden, snelle rebuild) │
└─────────────────────────────────────────────────────────────────┘
```

### Game Flow (Verified 13 December 2025)

De game doorloopt nu succesvol de volgende flow:

```
_start (0x100008)
    ↓
SetupThread syscall → SP = 0x64C700
    ↓
SetupHeap syscall → heap configured
    ↓
_InitSys → syscall table setup
    ↓
Startup() → 29 startup functies
    ↓
main() @ 0x1857c8 ← WE ZIJN HIER!
    ↓
┌──────────────────────────────────────────┐
│ Main Loop (draait stabiel):              │
│   1. Check MPEG video (g_mpeg.oid_1)     │
│   2. Check transitions                   │
│   3. UpdateJoy(), UpdateGameState()      │
│   4. if (g_psw):                         │
│      - OpenFrame()                       │
│      - RenderSw(), DrawSw()              │
│      - CloseFrame()                      │
│   5. g_cframe++                          │
└──────────────────────────────────────────┘
    ↓
CloseFrame() → sceDmaSend() → GIF packets
    ↓
❌ DMA data gaat nergens heen (geen GS emulatie)
```

### Grote Fixes (13 December 2025):

**1. PS2Recomp Code Generator Loop Fix (Bug #6)**
Het probleem was dat de code generator `return;` genereerde na ELKE function call (JAL), ook als die call binnen een loop zat. Dit brak loops met syscalls.

**2. Branch Delay Slot Condition Bug (Bug #7)**
Een kritieke bug waarbij de branch conditie werd geëvalueerd NA de delay slot, in plaats van ERVOOR. Hierdoor werd de MPEG video check nooit correct uitgevoerd.

**Beide fixes in onze PS2Recomp fork:** https://github.com/chrisking1981/PS2Recomp

---

## Voortgang Overzicht

| Fase | Status | Resultaat |
|------|--------|-----------|
| Boot sequence | ✅ | Game start correct |
| Syscall handling | ✅ | Alle basis syscalls werken |
| Function dispatch | ✅ | Mid-function entries werken |
| Indirect jumps | ✅ | Range lookup werkt |
| CD-ROM | ✅ | ISO reading geïmplementeerd |
| Game state loop | ✅ | Bypassed, game vordert |
| Sound/IOP | ✅ | Gestubbed + WAV playback |
| VBlank | ✅ | Periodiek triggered |
| **Loop Fix** | ✅ | PS2Recomp code generator gefixed |
| **Branch Delay** | ✅ | PS2Recomp code generator gefixed |
| **MPEG Video** | ✅ | Correct skipped wanneer geen video |
| **Main Loop** | ✅ | Draait stabiel @ 0x1857c8 |
| **Frame Rendering** | ✅ | OpenFrame/CloseFrame aangeroepen |
| DMA Transfers | ⚠️ | Code draait, hardware ontbreekt |
| Graphics output | ❌ | GS emulatie ontbreekt |

---

## Huidige Bottleneck: Graphics Synthesizer (GS) Emulatie

De game draait nu in de main loop en maakt graphics calls:

### Wat er gebeurt:
1. Game roept `OpenFrame()` aan (0x15F2xx)
2. Game bouwt VIF/GIF packets in geheugen
3. Game roept `CloseFrame()` aan
4. `sceDmaSend(g_pdcGif, data)` wordt aangeroepen
5. **DMA data gaat nergens heen** - geen GS emulatie!

### Wat we nodig hebben:
- **DMA channel emulatie** - om GIF packets te ontvangen
- **GIF unpacking** - om primitives te extraheren
- **GS rendering** - om pixels te tekenen

---

## Volgende Stappen

### 1. DMA/GIF Logging (Quick Win)
Voeg logging toe aan `sceDmaSend` om te zien:
- Welke DMA channel (GIF, VIF0, VIF1)
- Hoeveel data
- Wat voor GIF tags

### 2. GS Register Monitoring
Log GS register writes om te zien welke GS commands de game doet.

### 3. Basis GS Emulatie
Keuzes:
- **parallel-gs** (reference/) - Volledige Vulkan implementatie
- **Software rasterizer** - Simpeler maar trager
- **Stub met logging** - Om te begrijpen wat nodig is

---

## Key Technical Discoveries

### Branch Delay Slot Bug (13 December 2025)

**Probleem:**
```cpp
// FOUT - delay slot overschrijft register VOOR branch check
SET_GPR_U32(ctx, 2, READ32(...));  // Load into $2
SET_GPR_U32(ctx, 2, 0x260000);     // Delay slot overwrites!
if (GPR_U32(ctx, 2) == 0) { ... }  // Uses NEW value!
```

**Fix:**
```cpp
uint32_t branch_cond = GPR_U32(ctx, 2);  // Save BEFORE delay slot
SET_GPR_U32(ctx, 2, 0x260000);            // Delay slot executes
if (branch_cond == 0) { ... }             // Use saved value
```

**Impact:** MPEG video check werkt nu correct - skipt als `oid_1 == 0`.

### Loop Fix (eerder vandaag)

**Probleem:** Na elke JAL werd `return;` gegenereerd, ook binnen loops.

**Fix:** Gebruik `goto label_XXXX;` voor interne function calls.

---

## Build Instructies

**Snelle rebuild (na kleine wijziging):**
```bash
cd PS2Recomp/build
export PATH="/c/msys64/mingw64/bin:/c/msys64/usr/bin:$PATH"
ninja ps2EntryRunner
```

**Na wijziging in code_generator.cpp:**
```bash
# 1. Rebuild recompiler
cd PS2Recomp/build && ninja ps2recomp

# 2. Recompile game (optioneel - alleen als je nieuwe functies wilt)
cd sly1-recomp
../PS2Recomp/build/ps2xRecomp/ps2recomp.exe config/sly1_config.toml

# 3. Rebuild runtime
cd ../PS2Recomp/build && ninja ps2EntryRunner
```

---

## Referenties

- `LESSONS_LEARNED.md` - Alle geleerde lessen
- `docs/sessions/2025-12-13-gs-logging.md` - Vandaag's sessie details
- `docs/mpeg-system.md` - MPEG video systeem documentatie
- `reference/parallel-gs/` - GS emulatie referentie

---

*Laatst bijgewerkt: 13 December 2025 - Main loop verified, ready for GS emulation*
