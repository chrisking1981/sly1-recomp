# Getting Started - Learning Decomp/Recomp with AI

> **For beginners who want to learn decompilation and recompilation using AI as a learning partner**

---

## My Story

I started this project with **zero knowledge** of decompilation or recompilation. I didn't know MIPS assembly, didn't understand how PS2 games work, and had never done reverse engineering before.

What I did have:
- Curiosity about how games work
- Access to Claude Code (AI assistant)
- Willingness to ask "dumb" questions

**6 hours later:** I had found and fixed 5 critical bugs in PS2Recomp, the game was executing 108+ functions, and I understood concepts I thought would take months to learn.

---

## Why AI + Decomp/Recomp Works So Well

### The Key Insight

Decompilation and recompilation are **deterministic/binary** problems:
- Code either matches or it doesn't
- The game either runs or crashes
- Register values are either correct or wrong

This is **perfect for AI** because:

1. **AI can read the output** - Crash logs, compiler errors, and debug output are text that AI understands perfectly

2. **Clear success criteria** - "Does the code match?" is a yes/no question, not subjective

3. **Systematic iteration** - AI excels at methodical debugging:
   ```
   Test → Crash → Read error → Hypothesize → Fix → Test again
   ```

4. **AI wants a goal** - Give AI a binary target ("make this function match" or "fix this crash") and it will systematically work toward it

---

## How I Learn With AI

### My Approach

Instead of trying to learn everything first, I:

1. **Set a small goal**: "Let's try to run this game with PS2Recomp"
2. **Ask AI to explain each step**: "What is this doing? Why did it crash?"
3. **Let AI read the actual code/output**: Not describe it, but paste the real thing
4. **Ask follow-up questions**: "Why does SetupThread return a stack pointer?"

### Example Conversation

```
Me: The game crashes with "SP = 0x00000001". What's wrong?

AI: The stack pointer (SP) should be around 0x02000000, not 1.
    Let me trace where SP gets set...

    Looking at _start:
      0x100064: syscall        ; SetupThread
      0x100068: daddu $sp, $v0, $0  ; SP = return value!

    The SetupThread syscall is returning 1 (thread ID) but the game
    expects it to return the stack pointer address.

Me: Why does it expect that?

AI: According to PS2 SDK docs, SetupThread/InitMainThread returns
    the stack pointer, not a thread ID. The formula is:
    - If stack == -1: SP = 0x02000000 - stack_size
    - Else: SP = stack + stack_size

    This is a bug in PS2Recomp. Here's the fix...
```

In one exchange, I learned:
- What a stack pointer is
- How PS2 syscalls work
- How to read MIPS assembly
- How to trace execution flow

---

## What You Need

### Tools

| Tool | Purpose | Get it from |
|------|---------|-------------|
| **Claude Code** | AI assistant | [claude.ai/code](https://claude.ai/code) |
| **Git** | Version control | Your package manager |
| **Python 3** | Helper scripts | python.org |
| **MSYS2** (Windows) | Build environment | msys2.org |

### Files

| File | Purpose | Notes |
|------|---------|-------|
| **Game ELF** | The game executable | Extract from your own disc |
| **This repo** | Sly-specific configs | Clone from GitHub |
| **PS2Recomp** | The recompiler tool | Clone from GitHub |

### Mindset

- **Ask "dumb" questions** - There are no dumb questions when learning
- **Be specific** - "It crashes" → "It crashes with 'No func at 0x185bc8'"
- **Paste actual output** - Don't summarize, give AI the real error
- **One thing at a time** - Fix one bug before moving to the next

---

## Your First Session

### Step 1: Give AI Context

Start your AI session with:

```
I'm learning PS2 decompilation/recompilation. I have:
- Sly Cooper PS2 game (SCUS_971.98)
- PS2Recomp tool
- sly1-recomp config files

I want to understand how this works while getting the game to run.
Please explain each step as we go.

Here are the docs: [paste MY_SETUP.md contents]
```

### Step 2: Build and Run

Ask AI to help you build:

```
Help me build PS2Recomp and run it with Sly Cooper.
Explain what each command does.
```

### Step 3: Debug Together

When it crashes (it will!), paste the output:

```
It crashed with this output:
[paste full output]

What does this mean? How do we fix it?
```

### Step 4: Learn From Each Fix

After each fix, ask:

```
Can you explain why that fix worked?
What was the underlying concept I should understand?
```

---

## What You'll Learn

By working through crashes and fixes, you'll naturally learn:

| Concept | You'll learn it when... |
|---------|------------------------|
| **MIPS assembly** | Reading crash addresses and tracing code |
| **Stack/heap** | Debugging memory corruption |
| **Syscalls** | Implementing OS functions |
| **Function calling conventions** | Understanding register usage |
| **Delay slots** | Fixing branch-related bugs |
| **ELF format** | Understanding game structure |

You don't need to study these first - you'll learn them **in context** when they matter.

---

## Tips for Effective AI Learning

### DO:

- **Give AI the actual output** - Paste full error messages, not summaries
- **Ask "why"** - Understanding beats just fixing
- **Set binary goals** - "Make it run 10 more functions" is better than "improve it"
- **Let AI read code** - AI can understand assembly/C++ if you show it
- **Document as you go** - Ask AI to explain what you learned

### DON'T:

- **Don't expect AI to guess** - Give it concrete information
- **Don't skip exploration** - Let AI understand the codebase first
- **Don't try everything at once** - One bug at a time
- **Don't ignore AI questions** - If it asks for clarification, provide it

---

## Project Structure for AI

Give these docs to your AI:

| Document | When to give it |
|----------|-----------------|
| `MY_SETUP.md` | First - your environment setup |
| `PS2RECOMP_GUIDE.md` | When AI needs to understand the tool |
| `PROJECT_STATUS.md` | To show current progress |
| `AI_WORKFLOW.md` | For debugging methodology |
| `BUILD_GUIDE.md` | For build issues |

### Pro tip: Start sessions with context

```
I'm continuing work on Sly Cooper PS2 recompilation.
Here's my setup: [paste MY_SETUP.md]
Current status: [paste relevant section from PROJECT_STATUS.md]
```

---

## Why This Approach Works

### Traditional Learning
```
1. Read book about MIPS assembly (weeks)
2. Study PS2 architecture (weeks)
3. Learn reverse engineering (months)
4. Try to apply knowledge (frustrating)
```

### AI-Assisted Learning
```
1. Set goal: "Run this game"
2. Hit obstacle: "It crashes"
3. Learn concept: "This is why it crashed"
4. Apply immediately: "Here's the fix"
5. Repeat with next obstacle
```

You learn **what you need, when you need it**, with immediate practical application.

---

## Community

- **This project**: [github.com/chrisking1981/sly1-recomp](https://github.com/chrisking1981/sly1-recomp)
- **PS2Recomp**: [github.com/ran-j/PS2Recomp](https://github.com/ran-j/PS2Recomp)
- **Sly decomp**: [github.com/TheOnlyZac/sly1](https://github.com/TheOnlyZac/sly1)

---

## Final Thoughts

You don't need to be an expert to start. You need:
- Curiosity
- An AI assistant
- Willingness to ask questions

The AI will help you understand concepts as they become relevant. Each crash is a learning opportunity. Each fix teaches you something new.

**Start small. Ask questions. Learn by doing.**

---

*This guide was written by someone who learned decomp/recomp from scratch using AI. If I can do it, you can too.*
