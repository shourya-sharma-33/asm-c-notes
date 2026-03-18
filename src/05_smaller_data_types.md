# Lecture 05 — Working with Smaller Data Types

> Covers what happens when you load sub-32-bit values into 32-bit registers, why bytes get packed together in memory, and how to use partial register names (`bl`, `bh`, `cl`, etc.) to load exactly the size you want.

---

## The Problem: Bytes Packed in Memory

When you declare multiple bytes in `.data`, they get stored **adjacent** in memory — packed into the same 32-bit slot:

```nasm
section .data
    num     db  1       ; 1 byte, value 1
    num2    db  2       ; 1 byte, value 2
```

In memory it looks like this:

```
Address:    0x804a000    0x804a001
Value:          0x01         0x02
                ↑            ↑
               num          num2
```

Both bytes sit in the same 32-bit memory slot: `0x00000201`

x86 is doing this deliberately — **memory efficiency**. Why waste a full 32-bit slot on a single byte when you can pack four bytes in?

---

## Why Naive Load Fails

```nasm
mov ebx, [num]      ; intention: load 1
                    ; reality: loads all 32 bits at 0x804a000 = 0x00000201 = 513
```

`ebx` = 32-bit register. `[num]` tells x86 to load 32 bits from that address. It doesn't know you only wanted 1 byte — it grabs the whole slot, which includes `num2` sitting right next to it.

### GDB proof

```bash
gdb data
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi                     ; execute mov ebx, [num]
(gdb) info registers ebx
# ebx = 0x201    513            ← wrong, got both bytes
```

Inspect the raw memory:
```
(gdb) x/x 0x804a000
# 0x804a000:    0x00000201      ← 0x01 = num, 0x02 = num2, packed together
```

---

## The Fix: Partial Register Names

Tell the CPU how many bits to load by using the right register name.

### Register size breakdown (using B register as example):

```
 31              16 15          8 7            0
 ┌────────────────┬─────────────┬──────────────┐
 │   (upper 16)   │     BH      │      BL      │
 └────────────────┴─────────────┴──────────────┘
 │◄────────────── BX (16 bits) ───────────────►│
 │◄──────────────── EBX (32 bits) ────────────►│
```

| Name | Bits | Use when... |
|---|---|---|
| `ebx` | 32 | Working with 32-bit values |
| `bx` | 16 | Working with 16-bit values |
| `bl` | 8 (low) | Working with 8-bit values |
| `bh` | 8 (high) | Packing two 8-bit values in one register |

Same pattern: `eax/ax/al/ah`, `ecx/cx/cl/ch`, `edx/dx/dl/dh`

---

## Correct Program

```nasm
section .data
    num     db  1
    num2    db  2

section .text
    global _start

_start:
    mov bl, [num]       ; load 1 byte from num into low 8 bits of B register
    mov cl, [num2]      ; load 1 byte from num2 into low 8 bits of C register
    mov eax, 1
    int 0x80
```

### GDB session

```bash
gdb data
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi                     ; execute mov bl, [num]
(gdb) info registers bl
# bl = 0x1    1                 ← correct, got exactly 1 byte

(gdb) info registers ebx
# ebx = 0x1    1                ← consistent: lower 8 bits = 1, rest zeroed

(gdb) stepi                     ; execute mov cl, [num2]
(gdb) info registers cl
# cl = 0x2    2                 ← correct

(gdb) info registers ecx
# ecx = 0x2    2                ← same value, lower 8 bits
```

When you write to `bl`, the rest of `ebx` stays as-is (usually zero from program start). Reading `ebx` after writing `bl` with value `1` gives `1` because bit 0 = 1 and all higher bits = 0.

---

## `BL` vs `BH` — Why It Matters

```nasm
mov bh, [num]       ; puts value 1 into the HIGH byte of BX
mov ch, [num2]      ; puts value 2 into the HIGH byte of CX
```

### GDB session

```bash
(gdb) stepi
(gdb) info registers bh
# bh = 0x1    1                 ← bh alone looks correct

(gdb) info registers ebx
# ebx = 0x100    256            ← NOT 1
```

Why `256`? Because the bit is now in position 8 (the high byte), not position 0. `2^8 = 256`.

```bash
(gdb) info registers ecx
# ecx = 0x200    512            ← 2 × 2^8 = 512
```

### Visualization

```
BL = 1:    EBX = 0x00000001 = 1
BH = 1:    EBX = 0x00000100 = 256
```

**The same bit pattern has a different numeric value depending on where in the register it sits.**

---

## Quick Reference

| You have | Use this register name |
|---|---|
| 32-bit value | `eax`, `ebx`, `ecx`, `edx` |
| 16-bit value | `ax`, `bx`, `cx`, `dx` |
| 8-bit value (lower) | `al`, `bl`, `cl`, `dl` |
| 8-bit value (upper) | `ah`, `bh`, `ch`, `dh` |

**Default to `_l` (low) for 8-bit unless you're intentionally packing two values into one register.**

---

## Key Takeaways

- Bytes pack adjacently in memory — a naive 32-bit load grabs neighbors too
- Use partial register names (`bl`, `cl`) to tell x86 to load only 1 byte
- `bl` = lower 8 bits, `bh` = upper 8 bits of the 16-bit half
- Writing to `bh` vs `bl` gives wildly different values when you read `ebx` — because bit position changes the numeric value
- Match your register size to your data size

---

## What's Next
Next lecture: lists, characters, and strings — how text is stored and accessed in memory.
