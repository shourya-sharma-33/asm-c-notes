# Lecture 12 — Logical Operators: `and`, `or`, `not`, `xor`

> Covers the four bitwise instructions, the critical gotcha with `not` (it flips the entire register), and how to use a **bitmask** with `and` to isolate only the bits you care about.

---

## Setup — Working in Binary

Binary literals make bitwise ops easy to reason about:

```nasm
mov eax, 0b1010     ; eax = 1010 in binary = 10 decimal
mov ebx, 0b1001     ; ebx = 1001 in binary = 9 decimal
```

---

## `and` — Bitwise AND

```nasm
and destination, source     ; destination = destination AND source
```

Output bit = 1 only when **both** input bits are 1.

```
  1010
& 1001
------
  1000    = 8
```

```nasm
mov eax, 0b1010
mov ebx, 0b1001
and eax, ebx        ; eax = 0b1000 = 8
```

```
(gdb) info registers eax
# eax = 0x8    8
```

---

## `or` — Bitwise OR

```nasm
or destination, source      ; destination = destination OR source
```

Output bit = 1 when **either** input bit is 1.

```
  1010
| 1001
------
  1011    = 11
```

```nasm
mov eax, 0b1010
mov ebx, 0b1001
or eax, ebx         ; eax = 0b1011 = 11
```

```
(gdb) info registers eax
# eax = 0xb    11
```

---

## `not` — Bitwise NOT (The Gotcha)

```nasm
not destination     ; flip every bit in destination
```

**One operand only.** Flips all bits in the register — not just the ones you care about.

```nasm
mov eax, 0b1010     ; eax = 0x0000000A
not eax             ; eax = 0xFFFFFFF5  ← all 32 bits flipped
```

### GDB result

```
(gdb) info registers eax
# eax = 0xfffffff5    -11    ← not just 0101, entire register flipped
```

All the upper bits that were `0` became `1`. The actual lower-nibble result (`0101` = 5) is buried in there, but it's surrounded by garbage bits.

### Why this is a problem

If you only cared about 4 bits, you wanted `0101` = 5. Instead you got `0xFFFFFFF5`. The upper 28 bits all flipped to 1 even though they weren't part of your calculation.

---

## Fix: Bitmask with `and`

After `not`, use `and` with a **mask** to clear all the bits you don't want.

A mask is a value where:
- `1` in a position = **keep** that bit
- `0` in a position = **clear** that bit

To keep only the lowest 4 bits:

```nasm
mov eax, 0b1010     ; eax = 1010
not eax             ; eax = 0xFFFFFFF5 (all bits flipped)
and eax, 0xF        ; 0xF = 0b00001111 — keep only bottom 4 bits
                    ; eax = 0x00000005 = 5
```

### GDB session

```
(gdb) stepi                 ; not eax
(gdb) info registers eax
# eax = 0xfffffff5          ← messy

(gdb) stepi                 ; and eax, 0xF
(gdb) info registers eax
# eax = 0x5    5            ← clean, only the 4 bits we wanted
```

### Why `0xF` works

`0xF` = `0b00001111` — ones in the lowest 4 positions, zeros everywhere else. ANDing anything with `0` forces it to `0`. ANDing anything with `1` preserves it.

> You don't need to write `0x0000000F` — NASM fills in the upper zeros. Just `0xF` is enough.

### Mask reference

| Mask | Bits kept |
|---|---|
| `0xF` | Low 4 bits |
| `0xFF` | Low 8 bits (1 byte) |
| `0xFFFF` | Low 16 bits (1 word) |
| `0xFFFFFFFF` | All 32 bits |

---

## `xor` — Exclusive OR

```nasm
xor destination, source     ; destination = destination XOR source
```

Output bit = 1 when **exactly one** input bit is 1 (not both, not neither).

```
  1010
^ 1100
------
  0110    = 6
```

```nasm
mov eax, 0b1010
mov ebx, 0b1100
xor eax, ebx        ; eax = 0b0110 = 6
```

```
(gdb) info registers eax
# eax = 0x6    6
```

### XOR truth table

| A | B | A XOR B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

### Common XOR trick — zeroing a register

```nasm
xor eax, eax        ; eax = 0, always
```

Any value XOR'd with itself = 0. This is the idiomatic way to zero a register in assembly — faster than `mov eax, 0`.

---

## Complete Example

```nasm
section .text
    global _start

_start:
    ; AND
    mov eax, 0b1010
    mov ebx, 0b1001
    and eax, ebx            ; eax = 0b1000 = 8

    ; OR
    mov eax, 0b1010
    mov ebx, 0b1001
    or eax, ebx             ; eax = 0b1011 = 11

    ; NOT with mask
    mov eax, 0b1010
    not eax                 ; eax = 0xFFFFFFF5
    and eax, 0xF            ; eax = 0x5 (keep only low 4 bits)

    ; XOR
    mov eax, 0b1010
    mov ebx, 0b1100
    xor eax, ebx            ; eax = 0b0110 = 6

    mov eax, 1
    int 0x80
```

---

## Operator Summary

| Instruction | Rule | Example |
|---|---|---|
| `and dst, src` | 1 only if both are 1 | `1010 & 1001 = 1000` |
| `or dst, src` | 1 if either is 1 | `1010 \| 1001 = 1011` |
| `not dst` | Flip all bits (entire register) | `1010 → 0xFFFFFFF5` |
| `xor dst, src` | 1 if exactly one is 1 | `1010 ^ 1100 = 0110` |

---

## Key Takeaways

- `not` flips **every bit in the register** — always follow with an `and` mask to isolate the bits you want
- A mask uses `1`s to mark bits to keep and `0`s to clear
- `0xF` keeps the low 4 bits, `0xFF` keeps the low 8, etc.
- `xor eax, eax` is the standard idiom for zeroing a register
- All four instructions take the same `dst, src` form — except `not` which is single-operand

---

## What's Next
Control flow — `cmp`, conditional jumps (`je`, `jne`, `jl`, etc.), and loops. EFLAGS finally do real work.
