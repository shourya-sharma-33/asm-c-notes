# Lecture 02 — Operation Modes & Registers

> **No code this lecture.** Covers the three CPU operation modes and a full walkthrough of x86 registers — their sizes, naming conventions, and what they're conventionally used for. Next lecture is the first actual NASM program.

---

## Operation Modes

x86 processors have three modes. You'll almost always be in **protected mode**, but know the others exist.

| Mode | Description | When you'd use it |
|---|---|---|
| **Protected Mode** | Default/native state. Multiple processes run isolated — each gets its own memory segment and can't touch another process's memory. All features available. | Always, unless you have a specific reason not to |
| **Real Address Mode** | Early x86 mode. Direct hardware access, no memory protection. | Low-level hardware interaction, bootloaders, early OS stages |
| **System Management Mode** | Chip-specific OS mechanisms — power management, security. | Writing firmware or something tightly coupled to a specific chip |

> **Protected mode is what we use.** The memory isolation is why your crashed program doesn't take down the whole OS — it's confined to its own memory region.

---

## x86 is 32-bit

Each register in x86 is **32 bits wide**. This is the fundamental unit.

---

## Register Naming & Partial Access

x86 lets you access different *portions* of a register using different names. This is unique and important to understand.

Using the `A` register as the example:

```
 31                16 15            8 7             0
 ┌──────────────────┬───────────────┬───────────────┐
 │    (upper 16)    │      AH       │      AL       │
 └──────────────────┴───────────────┴───────────────┘
 │◄──────────────── AX (16 bits) ──────────────────►│
 │◄────────────────── EAX (32 bits) ────────────────►│
```

| Name | Bits accessed | Meaning |
|---|---|---|
| `EAX` | All 32 bits | **E** = Extended (full register) |
| `AX` | Lower 16 bits | Drop the `E` |
| `AH` | Bits 8–15 (high byte of AX) | **H** = High |
| `AL` | Bits 0–7 (low byte of AX) | **L** = Low |

Same pattern applies to `B`, `C`, `D` registers: `EBX/BX/BH/BL`, `ECX/CX/CH/CL`, `EDX/DX/DH/DL`.

**Why does this matter?**

- Working with a **16-bit signed number**? Use `AX` not `EAX`. If you do arithmetic in `EAX`, the sign bit (bit 15) can spill into bit 16 and corrupt results. Using `AX` tells the CPU "this is 16-bit, keep operations within that range."
- **Two 8-bit values in one register?** Store one in `AH` and one in `AL` — you've packed two values into a single register. More efficient use of limited register space.

---

## General Purpose Registers

| Register | Conventional Use | Notes |
|---|---|---|
| `EAX` | **Accumulator** — multiply/divide results | Return value from functions also goes here |
| `EBX` | General purpose | No strong convention |
| `ECX` | **Loop counter** | `LOOP` instruction decrements ECX automatically |
| `EDX` | General purpose | Sometimes paired with EAX for 64-bit multiply results |
| `ESI` | **Source index** — high-speed memory transfer | Used in string/memory operations |
| `EDI` | **Destination index** — high-speed memory transfer | Used in string/memory operations |

---

## Stack & Frame Registers

Don't modify these directly — they're managed by the CPU and calling conventions.

| Register | Name | What it does |
|---|---|---|
| `ESP` | **Stack Pointer** | Points to the top of the stack (next available stack location). The stack is your program's RAM segment — where you store things that don't fit in registers (arrays, local variables, etc.) |
| `EBP` | **Base Pointer** | Points to the base of the current stack frame — used to reference function parameters and local variables. More detail when we get to functions. |

> Think of ESP as "where am I in stack memory right now." Every `push`/`pop` moves it.

---

## Special Purpose Registers

| Register | Name | What it does |
|---|---|---|
| `EIP` | **Instruction Pointer** | Points to the address of the **next instruction to execute**. The CPU auto-advances this. You don't usually touch it directly — jumps/calls modify it implicitly. |
| `EFLAGS` | **Flags Register** | Each bit is a status flag set by the last operation. |

### Common EFLAGS bits

| Flag | Name | Set when... |
|---|---|---|
| `CF` | Carry Flag | Operation produced a carry/borrow out of the MSB |
| `OF` | Overflow Flag | Signed arithmetic overflowed |
| `SF` | Sign Flag | Result was negative (MSB = 1) |
| `ZF` | Zero Flag | Result was zero |

These flags are how conditional jumps (`je`, `jne`, `jl`, etc.) work — they check EFLAGS after a `cmp` or arithmetic instruction.

---

## Register Summary Cheatsheet

```
EAX / AX / AH / AL   — accumulator, return values, mul/div
EBX / BX / BH / BL   — general purpose
ECX / CX / CH / CL   — loop counter
EDX / DX / DH / DL   — general purpose, mul/div overflow
ESI                   — source for memory ops
EDI                   — destination for memory ops
ESP                   — stack pointer (don't touch directly)
EBP                   — frame pointer (don't touch directly)
EIP                   — instruction pointer (auto-managed)
EFLAGS               — status flags (auto-managed)
```

---

## What's Next
Next lecture: first NASM program on Linux. We'll be using NASM as the assembler, compiling and running `.asm` files. Code + GDB starts here.
