# Lecture 01 — x86 Architecture Overview

> **No code this lecture.** Pure theory. Covers how the CPU works, how it talks to memory and I/O, and why registers are faster than RAM. This is the mental model everything else in assembly is built on.

---

## The Big Picture: CPU, Memory, and I/O

Three main blocks on any computer:

- **CPU** — does all the computation
- **Memory (RAM)** — stores data and instructions
- **I/O Devices** — keyboard, monitor, mouse, etc. (many of these in practice)

These three communicate via three **buses**:

| Bus | Role |
|---|---|
| **Control Bus** | Synchronizes all devices — signals who's reading, who's writing, when |
| **Address Bus** | Carries the *location* of data/instructions (think: a pointer) |
| **Data Bus** | Carries the actual data or instructions being transferred |

> I/O devices often sit on a separate **I/O bus** in reality, but conceptually you can lump it with the data bus for now.

---

## Inside the CPU

Four main components:

### 1. ALU — Arithmetic Logic Unit
Does all math and logic: `add`, `sub`, `and`, `or`, `not`, comparisons. Any time the CPU "calculates" something, it goes through here.

### 2. Registers
- Storage locations that live **inside the CPU itself**
- Fastest memory you can access — we'll always prefer registers over RAM
- Direct access, ~1 clock cycle

### 3. Clock
- Alternates between 0 and 1 at a fixed frequency
- **One cycle** = the time between two falling edges (1 → 0)
- Everything in the CPU is synchronized to this clock — *something happens on every tick*
- Measured in **oscillations per second (Hz)**
- 1 GHz = 1 billion cycles/second → that's why CPUs are fast

> The clock is the CPU's heartbeat. Every beat, an operation progresses.

### 4. Control Unit
- Decodes instructions and directs what happens
- Looks at an instruction, figures out: what operation is this? what operands does it need? where do results go?
- Routes work to the ALU, memory, I/O as needed

---

## Fetch-Decode-Execute Cycle

Every instruction the CPU runs goes through these steps:

```
1. FETCH      — pull the next instruction from the instruction queue
2. DECODE     — control unit reads it, identifies the operation and operands
3. FETCH OPS  — retrieve operand values from registers or memory
4. EXECUTE    — ALU or other unit performs the operation
5. FLAGS      — update status flags (e.g. zero flag, carry flag)
6. STORE      — write result to destination register/memory (if needed)
```

**Example:** `ADD eax, ebx`
- Fetch: grab the ADD instruction
- Decode: it's an add, operands are `eax` and `ebx`
- Fetch ops: read values from `eax` and `ebx`
- Execute: ALU adds them
- Flags: set zero flag if result is 0, carry flag if overflow, etc.
- Store: result goes into `eax`

This loop is running **constantly**, billions of times per second.

---

## Reading from Memory (RAM) — Why It's Slow

Reading from RAM isn't direct. It takes multiple steps, each costing ~1 clock cycle:

```
1. Put target address on the Address Bus       (~1 cycle)
2. Assert the CPU's RD (read) pin              (~1 cycle)  ← tells CPU "I want to read"
3. Wait for memory to respond                  (~1 cycle)
4. Copy data from Data Bus to destination      (~1 cycle)
```

**Total: ~4 clock cycles** to read from RAM.

Compare that to **register access: ~1 clock cycle** (direct, no bus involved).

> RAM is ~4x slower than registers. This is why assembly code obsesses over keeping things in registers.

---

## Caching

Since RAM is slow, CPUs use **caches** — faster intermediate storage:

| Cache | Location | Speed | Type of RAM |
|---|---|---|---|
| **L1 Cache** | Inside the CPU | Very fast (close to register speed) | Static RAM (SRAM) |
| **L2 Cache** | Outside CPU, on high-speed bus | Faster than RAM, slower than L1 | Static RAM (SRAM) |
| **Main RAM** | DRAM | Slowest | Dynamic RAM (DRAM) |

**SRAM vs DRAM:**
- **DRAM (Dynamic RAM)** — needs constant refreshing to hold data. Cheaper, used for main memory.
- **SRAM (Static RAM)** — no refresh needed, much faster, but expensive. Used for caches.

The CPU automatically caches frequently-accessed memory in L1/L2 to reduce the cost of RAM reads.

---

## Mental Model Summary

```
Speed:   Registers > L1 Cache > L2 Cache > RAM
Cost:    Registers > L1 Cache > L2 Cache > RAM  (registers most expensive per byte)
Size:    Registers < L1 Cache < L2 Cache < RAM  (registers fewest bytes)
```

When you write assembly, you're manually managing the register layer. The goal is always: **keep your working data in registers**, spill to memory only when you have to.

---

## What's Next
The next lectures will cover the actual registers available in x86-64, their names, sizes, and conventions — then we get into writing NASM code.
