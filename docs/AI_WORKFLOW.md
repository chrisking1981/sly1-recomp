# AI-Assisted Development Workflow

> **Author:** Chris King (@chrisking1981)
> **AI Tool:** Claude Code with Claude Opus 4.5
> **Project:** Sly Cooper PS2 Static Recompilation

---

## Overview

This document describes how I use AI (specifically Claude Code with Opus 4.5) to systematically debug and fix complex reverse engineering projects.

### Results Achieved

| Metric | Before | After |
|--------|--------|-------|
| Function calls | 0 | 108+ |
| Boot sequence | Immediate crash | Completes |
| Bugs found | - | 5 critical |
| Scripts created | 0 | 6 |
| Time spent | - | ~4-6 hours |
| Equivalent manual time | - | Est. 40-80 hours |

---

## Workflow Phases

### Phase 1: Setup & Exploration

**Goal:** Understand the project structure and tools

```
Prompt: "Explore the ps2recomp repository and explain what each component does"
```

The AI will:
- Read README files and documentation
- Examine source code structure
- Identify key files and their purposes
- Report back with a summary

**Don't skip this phase** - understanding the codebase prevents guessing later.

### Phase 2: Build & Test

**Goal:** Get something running, even if it crashes

```
Prompt: "Build the project and run it"
```

The AI can:
- Run cmake/make/ninja
- Capture compiler errors
- Identify missing dependencies
- Suggest fixes for build issues

### Phase 3: Systematic Debugging

**Goal:** Find and fix bugs methodically

```
Prompt: "The game crashes. Think like a real reverse engineer -
        test everything systematically so we know what's working"
```

This triggers methodical analysis:
1. **Identify the crash point** - What address? What function?
2. **Examine register state** - Are values correct?
3. **Trace backwards** - What led to this state?
4. **Form hypotheses** - Why might this happen?
5. **Test hypotheses** - Add debug output, check assumptions
6. **Fix and verify** - Apply fix, confirm it works

### Phase 4: Iterative Fixing

**Goal:** Fix bugs one at a time

```
Test → Crash → Analyze → Fix → Test → New Crash → ...
```

Example progression from this project:
```
Crash #1: "No func at 0x0"
  → Fix: jr $ra must set ctx->pc

Crash #2: "SP = 0x00000001" (corrupt stack)
  → Fix: SetupThread must return stack pointer, not thread ID

Crash #3: Compiler error "__is_pointer reserved"
  → Fix: Sanitize reserved C++ identifiers

Crash #4: Stack not restored after function call
  → Fix: Delay slots split across function boundaries

Crash #5: "No func at 0x185bc8"
  → Analysis: Missing entry points, need JAL return addresses
```

### Phase 5: Automation

**Goal:** Script repetitive tasks

When we identified 18,000+ missing JAL return addresses, Claude created Python scripts:

```python
# analyze_jal_returns.py - Scans ELF for all JAL instructions
# fix_delay_slot_splits.py - Fixes function boundary issues
# iterate_fixes.py - Automated test-fix-rebuild loop
```

This automates work that would take hours manually.

### Phase 6: Documentation

**Goal:** Document findings for others

```
Prompt: "Create a detailed bug report for the ps2recomp maintainer"
```

This produces:
- Exact description of each bug
- Code snippets showing before/after
- Explanation of root cause
- Test results proving the fix works

---

## Scripts Created

| Script | Purpose |
|--------|---------|
| `analyze_jal_returns.py` | Scans ELF for JAL instructions, extracts return addresses, adds them to functions.json |
| `fix_delay_slot_splits.py` | Identifies functions that end with branches and extends them to include delay slots |
| `add_entry.py` | Manually adds a single entry point to functions.json |
| `iterate_fixes.py` | Automated loop: test → find crash → add entry → rebuild |

### analyze_jal_returns.py
```
Input:  ELF file, existing functions.json
Output: Updated functions.json with JAL return addresses

Found 18,111 new entry points from JAL instructions
Total functions: 4,682 → 22,416
```

### fix_delay_slot_splits.py
```
Input:  ELF file, functions.json
Output: Fixed functions.json with adjusted boundaries

Fixed 10 functions where delay slots were split
```

---

## Effective Prompts

### For Exploration
```
"Explore the codebase and explain the architecture"
"What does this function do? Read it and explain"
"Search for all places where X is used"
```

### For Debugging
```
"Think like a reverse engineer - what could cause this?"
"Test systematically, don't guess"
"What's the value of SP at this point? Trace it back"
```

### For Fixing
```
"Fix this bug, show me the exact code change"
"Create a script to automate this"
"What's the minimal change needed?"
```

### For Documentation
```
"Document what we found for the maintainer"
"Create a bug report with all the details"
"Explain this so someone else can understand"
```

---

## Best Practices

### DO:
- Give clear, specific goals
- Ask AI to "think systematically"
- Let AI read actual source code (not just describe)
- Build iteratively - fix one thing at a time
- Ask for documentation of findings
- Create scripts for repetitive tasks
- Verify fixes with tests

### DON'T:
- Expect AI to guess without information
- Skip the exploration phase
- Try to fix everything at once
- Ignore AI's questions or suggestions
- Forget to test after each fix
- Assume the AI remembers previous sessions (it doesn't)

---

## Example Session: Finding the SetupThread Bug

Here's a real example of how we found a critical bug:

### 1. Observed Problem
```
=== Call #2 ===
  SP: 0xfffffff1   ← This is wrong! Should be ~0x2000000
```

### 2. AI Analysis
```
The stack pointer is corrupt. Let me trace where SP gets set...

Looking at _start:
  0x100064: syscall        ; SetupThread
  0x100068: daddu $sp, $v0, $0  ; SP = return value of syscall!

So SP is set from $v0, which comes from SetupThread syscall.
```

### 3. Found the Bug
```cpp
// Current code (WRONG):
case 0x3c: // SetupThread
    setReturnS32(ctx, 1);  // Returns thread ID = 1
```

### 4. Researched Correct Behavior
```
AI searched PS2 SDK documentation and found:
- SetupThread returns STACK POINTER, not thread ID
- Formula: SP = stack + stack_size (or 0x02000000 - stack_size if stack == -1)
```

### 5. Applied Fix
```cpp
// Fixed code:
case 0x3c: // SetupThread
    uint32_t stack_ptr = (stack == -1)
        ? 0x02000000 - stack_size
        : stack + stack_size;
    setReturnU32(ctx, stack_ptr);  // Returns stack pointer
```

### 6. Verified
```
=== Call #2 ===
  SP: 0x64c700   ← Correct!
```

---

## Lessons Learned

### 1. Systematic beats guessing
Random changes waste time. Following the execution flow methodically finds bugs faster.

### 2. Documentation is valuable
Writing up findings forces clear thinking and helps others (including future AI sessions).

### 3. Scripts save hours
Automating repetitive tasks (like finding 18k entry points) is worth the upfront investment.

### 4. AI + Human = Best combo
- **Human:** Domain knowledge, direction, verification
- **AI:** Code reading, pattern matching, automation

### 5. Test after every fix
A fix that isn't tested might break something else.

---

## Reproducibility

To reproduce this workflow on a new game:

1. **Setup**
   ```
   Clone PS2Recomp and your game-specific config repo
   Build PS2Recomp
   Extract game ELF from disc
   ```

2. **Initial Analysis**
   ```
   Run ps2analyzer on ELF to generate initial functions.json
   Configure stubs/skips in config.toml
   ```

3. **First Run**
   ```
   Generate recompiled code
   Build runtime
   Run - expect crashes
   ```

4. **Debug Loop**
   ```
   For each crash:
     - Note the PC/address
     - Add missing entry point OR
     - Fix bug in runtime/recompiler
     - Rebuild and test
   ```

5. **Automate**
   ```
   Once patterns emerge, create scripts
   Run scripts to find all instances of the pattern
   ```

6. **Document**
   ```
   Create bug reports for upstream
   Document workflow for future reference
   ```

---

## Contact & Resources

- **GitHub:** [@chrisking1981](https://github.com/chrisking1981)
- **This project:** [sly1-recomp](https://github.com/chrisking1981/sly1-recomp)
- **PS2Recomp fork:** [PS2Recomp](https://github.com/chrisking1981/PS2Recomp)
- **AI Tool:** [Claude Code](https://claude.ai/code) with Opus 4.5

---

*This workflow was developed using Claude Code (Opus 4.5). Feel free to adapt it for your own projects.*
