# Sessie: 12 December 2024 - Build Fixes & Entry Point Issues

## Samenvatting

Deze sessie richtte zich op het fixen van de build environment en het oplossen van missing function entry points in PS2Recomp.

---

## Problemen Opgelost

### 1. Build Environment (MSYS2 PATH)

**Probleem:** CMake/Ninja builds faalden met cryptische errors:
- `remote helper 'https' aborted session`
- `basename: opdracht niet gevonden`
- Compilers leken te falen zonder error message

**Oorzaak:** De Claude Code shell had alleen `/mingw64/bin` in PATH, maar MSYS2 git vereist ook `/usr/bin` voor helper scripts.

**Oplossing:**
```bash
export PATH="/c/msys64/mingw64/bin:/c/msys64/usr/bin:$PATH"
```

**Documentatie:** `docs/BUILD.md` aangemaakt met volledige instructies.

### 2. Missing Entry Point 0x185bb0

**Probleem:** Game crashte op call #110 met "No func at 0x185bb0"

**Analyse:** 0x185bb0 viel binnen functie `entry_185ba0` (address: 1596320, size: 36). Een JAL sprong naar het midden van deze functie.

**Oplossing:** Functie gesplit in `sly1-recomp/config/sly1_functions.json`:
```json
// Oud:
{"name": "entry_185ba0", "address": 1596320, "size": 36}

// Nieuw:
{"name": "entry_185ba0", "address": 1596320, "size": 16},
{"name": "entry_185bb0", "address": 1596336, "size": 20}
```

**Resultaat:** Game komt nu tot call #132 (was #110).

### 3. Missing Entry Point 0x1857f4

**Probleem:** Game crashte op call #132 met "No func at 0x1857f4"

**Analyse:** 0x1857f4 valt 4 bytes na `entry_1857f0` - slechts 1 MIPS instructie.

**Oplossing toegepast:**
```json
// Oud:
{"name": "entry_1857f0", "address": 1595376, "size": 20}

// Nieuw:
{"name": "entry_1857f0", "address": 1595376, "size": 4},
{"name": "entry_1857f4", "address": 1595380, "size": 16}
```

**Status:** NIET OPGELOST - Functie is correct toegevoegd en geregistreerd, maar runtime vindt het nog steeds niet. Dit wijst op een dieper probleem in de function lookup.

---

## Documentatie Updates

### Nieuwe Bestanden
- `docs/BUILD.md` - Volledige build instructies voor MSYS2/MinGW64
- `docs/sessions/` - Sessie logs directory
- `docs/sessions/2024-12-12-build-fixes.md` - Dit bestand

### Geüpdatet
- `docs/NEXT_STEPS.md` - Status update, crash info, build commando's
- `reference/AI_CONTEXT.md` - Build quick reference toegevoegd

---

## Technische Details

### Build Workflow
1. Edit `sly1-recomp/config/sly1_functions.json`
2. Run recompiler: `ps2recomp.exe sly1_config.toml`
3. Remove object files: `rm ps2xRuntime/CMakeFiles/.../ps2_recompiled_functions.cpp.obj`
4. Remove register file: `rm ps2xRuntime/CMakeFiles/.../register_functions.cpp.obj`
5. Rebuild: `ninja ps2EntryRunner`
6. Test: `./ps2EntryRunner.exe "../../../sly1-decomp/disc/SCUS_971.98"`

### Function Lookup Issue
De functie `entry_1857f4` is:
- Correct in `sly1_functions.json` (address: 1595380)
- Correct in `ps2_recompiled_functions.cpp`
- Correct in `register_functions.cpp` met `runtime.registerFunction(0x1857f4, entry_1857f4);`

Maar de runtime geeft nog steeds "No func at 0x1857f4". Mogelijke oorzaken:
1. Function table niet correct geïnitialiseerd
2. Address masking issue
3. Lookup bug in runtime

---

## Volgende Stappen

~~1. **Debug function registration** - Print alle geregistreerde functies bij startup~~
~~2. **Check lookup code** - Bekijk `ps2_runtime.cpp` hoe functies worden opgezocht~~
~~3. **Vergelijk met vorige build** - Was dit probleem er al?~~

**OPGELOST!** Het probleem was dat de recompiler output naar `recomp_output/` gaat, maar de runtime compiled van `ps2xRuntime/src/runner/`. We moesten de bestanden kopiëren:

```bash
# Na elke recompile:
cp recomp_output/ps2_recompiled_functions.cpp PS2Recomp/ps2xRuntime/src/runner/
cp recomp_output/ps2_recompiled_functions.h PS2Recomp/ps2xRuntime/include/
cp recomp_output/register_functions.cpp PS2Recomp/ps2xRuntime/src/runner/
```

## Voortgang na de Fix

- **Call #132**: 0x1857f4 fix werkt nu!
- **Call #133**: Crash op 0x15f240 -> Fixed
- **Call #136**: Crash op 0x15f258 -> Fixed
- **Call #138**: SignalSema syscall loop - game draait door!

De game is nu voorbij de crashes en draait in een syscall loop (SignalSema). Dit betekent dat de boot sequence werkt en de game wacht op iets (waarschijnlijk VSync of een timer).

## Wat zijn "Calls"?

Elke "call" is een functie-aanroep in de recompiled game. De game bestaat uit ~22.000 functies die elkaar aanroepen.

**Boot milestones:**
- Call 1-10: _start, SetupThread, SetupHeap
- Call 10-100: System initialization, syscall table setup
- Call 100-200: Game-specifieke initialisatie
- Call 200+: Begin van main game code

Nu we bij call #138 zijn en de game in een loop draait (niet crasht), betekent dit dat de basale execution werkt. Het volgende probleem is waarschijnlijk:
1. Graphics (GS) niet geïmplementeerd
2. DMA niet geïmplementeerd
3. VSync/Timer niet geïmplementeerd

## Script voor Entry Point Fixes

`tools/fix_entry_point.py` gemaakt om het splitsen van functies te automatiseren:

```bash
python tools/fix_entry_point.py 0x15f240
# Output: splits functie, updates json, geeft instructies
```

---

## Commands Used

```bash
# Build met correcte PATH
PATH="/c/msys64/mingw64/bin:/c/msys64/usr/bin:$PATH"

# Configure (clean build)
cmake -G Ninja -DFETCHCONTENT_UPDATES_DISCONNECTED=ON -DCMAKE_BUILD_TYPE=Release ..

# Build recompiler
ninja ps2recomp

# Recompile game
cd sly1-recomp/config
../../PS2Recomp/build/ps2xRecomp/ps2recomp.exe sly1_config.toml

# Build runtime
ninja ps2EntryRunner

# Run game
./ps2EntryRunner.exe "../../../sly1-decomp/disc/SCUS_971.98"
```

---

## Decomp vs Recomp

De `sly1-decomp` directory bevat handmatige decompilatie:
- 160 C bestanden, ~15.700 regels code
- ~760 functies handmatig reverse engineered
- Leesbare code met echte functienamen

De `PS2Recomp` genereert automatisch:
- 22.000+ functies
- Gegenereerde namen (entry_XXXXX)
- Volledige game code, maar minder leesbaar

De decomp is nuttig voor het begrijpen van de game structuur, maar de recomp is nodig om de game te draaien.

---

*Sessie duur: ~2 uur*
*AI: Claude Opus 4.5*
*Status bij einde sessie: Game draait tot call #138 in SignalSema loop*
