# Next Steps: Systematische Aanpak naar Graphics

> **Status:** 12 December 2024
> **Doel:** Graphics in beeld krijgen voor Sly Cooper

---

## Huidige Situatie

```
┌─────────────────────────────────────────────────────────────────┐
│ PS2Recomp Executie Status                                       │
├─────────────────────────────────────────────────────────────────┤
│ Boot sequence:    ✅ Werkt (_start → SetupThread → main entry)  │
│ Functies executed: 138+ calls                                    │
│ Status:           🔄 Game draait in SignalSema loop             │
│ Blocker:          Syscall implementatie (SignalSema/VSync)      │
│ Graphics:         ❌ Nog niet bereikt                            │
│ DMA:              ❌ Nog niet bereikt                            │
│ Build:            ✅ Werkt (zie BUILD.md voor instructies)       │
└─────────────────────────────────────────────────────────────────┘
```

### Entry Points Gefixed Deze Sessie:
- 0x185bb0 ✅
- 0x1857f4 ✅
- 0x15f240 ✅
- 0x15f258 ✅

---

## Laatste Voortgang

### Crash Fix (0x185bb0)
- **Probleem:** Game crashte op call #110 met "No func at 0x185bb0"
- **Oorzaak:** JAL naar midden van bestaande functie `entry_185ba0`
- **Oplossing:** Functie gesplit in `entry_185ba0` (size: 16) en `entry_185bb0` (size: 20)
- **Bestand:** `sly1-recomp/config/sly1_functions.json`
- **Status:** Recompiled, wacht op herbouw runtime

### Build Environment Fix
- **Probleem:** CMake/Ninja builds faalden met git https helper errors
- **Oorzaak:** MSYS2 PATH niet compleet in Claude Code shell
- **Oplossing:** Beide `/c/msys64/mingw64/bin` EN `/c/msys64/usr/bin` in PATH
- **Zie:** `docs/BUILD.md` voor volledige instructies

---

## Systematische Aanpak

### Fase 1: Data Verzamelen ✅ VOLTOOID

**Resultaat:**
- Trace geanalyseerd met `tools/trace_analyzer.py`
- Crash geïdentificeerd op 0x185bb0
- Fix toegepast

### Fase 2: Crash Fixen ⏳ IN PROGRESS

**Volgende stappen:**
1. Rebuild PS2Recomp met fix
2. Run game opnieuw
3. Analyseer nieuwe crash (indien aanwezig)
4. Herhaal tot we verder komen

**Commando's:**
```bash
# Vanaf project root
cd PS2Recomp/build

# Configure (alleen als clean build nodig)
PATH="/c/msys64/mingw64/bin:/c/msys64/usr/bin:$PATH" \
  /c/msys64/mingw64/bin/cmake.exe -G Ninja \
  -DFETCHCONTENT_UPDATES_DISCONNECTED=ON \
  -DCMAKE_BUILD_TYPE=Release ..

# Build
PATH="/c/msys64/mingw64/bin:/c/msys64/usr/bin:$PATH" \
  /c/msys64/mingw64/bin/ninja.exe ps2EntryRunner

# Run
cd ps2xRuntime
./ps2EntryRunner.exe "../../../sly1-decomp/disc/SCUS_971.98"
```

### Fase 3: IO Stubbing voor Stabiliteit

**Doel:** Game kan doordraaien zonder te crashen op IO

**Aanpak:**
- Alle GS register writes loggen maar niet crashen
- DMA transfers "voltooien" (busy bit clearen)
- VIF data accepteren maar negeren
- Interrupts stubben

**Test:** Game draait door tot main loop (zelfs zonder graphics)

### Fase 4: Eerste Graphics Output

**Doel:** Iets op scherm krijgen

**Opties:**

#### Optie A: Raylib (Simpel)
PS2Recomp heeft al Raylib als dependency:
1. Open een window met Raylib
2. Log GS framebuffer writes
3. Teken een placeholder

#### Optie B: Parallel-GS Integratie (Complex)
Integreer de parallel-gs GS emulator:
1. Voeg Vulkan dependency toe
2. Route GS calls naar parallel-gs interface
3. Krijg echte graphics output

**Aanbeveling:** Start met Optie A, migreer later naar B

### Fase 5: Iteratief Verbeteren

Zodra we iets zien:
1. Identificeer wat er mis is
2. Debug specifieke GS feature
3. Implementeer/fix
4. Herhaal

---

## Benodigde Tools

| Tool | Doel | Status |
|------|------|--------|
| `trace_analyzer.py` | Analyseer execution logs | ✅ Gemaakt |
| Execution logger | Uitgebreide IO logging | ⚠️ Deels in PS2Recomp |
| PCSX2 trace | Vergelijk met echte emulator | ❌ Nog niet |
| GS register decoder | Begrijp GS writes | ❌ Nog niet |

---

## Deterministische Test Cases

### Test 1: Boot verder dan 110 calls
**Input:** Sly Cooper ELF
**Expected:** Game komt verder dan call #110 na fix
**Actual:** TBD - wacht op rebuild

### Test 2: GS registers worden geschreven
**Input:** Game executie
**Expected:** We zien welke GS registers worden geschreven
**Actual:** TBD (via trace_analyzer)

### Test 3: DMA transfers starten
**Input:** Game executie
**Expected:** We zien DMA transfer attempts
**Actual:** TBD (via trace_analyzer)

### Test 4: Window opent
**Input:** Game met Raylib window
**Expected:** Window opent, blijft open
**Actual:** TBD

### Test 5: Eerste pixels
**Input:** Game met basis GS handling
**Expected:** Iets zichtbaar op scherm
**Actual:** TBD

---

## Volgende Actie

**NU:** Rebuild PS2Recomp en test of de 0x185bb0 fix werkt

```bash
# Zie docs/BUILD.md voor volledige instructies
cd PS2Recomp/build
PATH="/c/msys64/mingw64/bin:/c/msys64/usr/bin:$PATH" ninja ps2EntryRunner
cd ps2xRuntime
./ps2EntryRunner.exe "../../../sly1-decomp/disc/SCUS_971.98" 2>&1 | tee ../../../trace_new.log
```

---

## Referenties

- `reference/parallel-gs/BLOG_POST.md` - Hoe GS emulatie werkt
- `reference/ps2-recompiler-Cactus/` - Threading/syscall referentie
- `docs/ps2recomp-analysis.md` - Component status overzicht
- `docs/BUILD.md` - Build instructies

---

*Dit document wordt bijgewerkt naarmate we vorderen*
