# Lecture 13 — Shifts: `shr`, `shl`, `sar`, `sal`

> Covers logical and arithmetic bit shifts, what happens to the shifted-out bit, and the key practical insight: shifts are fast multiply/divide by powers of 2.

---

## What Shifts Do

A shift moves all bits in a register left or right by N positions.

```
Value:  0 0 1 0   (2)
SHR 1:  0 0 0 1   (1)   ← everything moves right one spot
SHL 1:  0 1 0 0   (4)   ← everything moves left one spot
```

The bit that falls off the end goes into **CF (Carry Flag)**.

---

## `shr` — Shift Right (Logical)

```nasm
shr destination, count      ; shift bits right by count positions
```

```nasm
mov eax, 2          ; eax = 0b0010 = 2
shr eax, 1          ; eax = 0b0001 = 1, CF = 0 (the shifted-out bit)
```

### GDB session

```
(gdb) stepi × 2
(gdb) info registers eax
# eax = 0x1    1            ← 2 >> 1 = 1
```

**Effect: divides by 2** (integer division — remainder goes to CF).

### Chaining shifts

```nasm
mov eax, 12         ; 12
shr eax, 1          ; 12 >> 1 = 6
shr eax, 1          ; 6  >> 1 = 3
```

Each right shift = divide by 2. Two shifts = divide by 4.

General rule: `shr eax, N` = `eax ÷ 2^N`

---

## `shl` — Shift Left (Logical)

```nasm
shl destination, count      ; shift bits left by count positions
```

```nasm
mov eax, 2          ; eax = 0b0010 = 2
shl eax, 1          ; eax = 0b0100 = 4, CF = 0
```

**Effect: multiplies by 2.**

General rule: `shl eax, N` = `eax × 2^N`

### Examples

| Start | Instruction | Result |
|---|---|---|
| 2 | `shl eax, 1` | 4 |
| 3 | `shl eax, 2` | 12 |
| 1 | `shl eax, 4` | 16 |
| 10 | `shr eax, 1` | 5 |
| 12 | `shr eax, 2` | 3 |

---

## The Carry Flag and Shifted-Out Bits

The bit that gets pushed off the edge always lands in CF:

```nasm
mov eax, 0b0011     ; eax = 3
shr eax, 1          ; eax = 0b0001 = 1, CF = 1  (the 1-bit that fell off)
```

This is how you can check for odd numbers before dividing:
- If CF = 1 after `shr eax, 1` → original value was odd (remainder = 1)
- If CF = 0 → original value was even

---

## Arithmetic Shifts — `sar` and `sal`

Logical shifts (`shr`/`shl`) fill vacated bits with **0**.
Arithmetic shifts preserve the **sign bit**.

| Instruction | Full name | Fills vacated bits with |
|---|---|---|
| `shr` | Shift Right Logical | `0` |
| `shl` | Shift Left Logical | `0` |
| `sar` | Shift Right Arithmetic | **sign bit** (MSB) |
| `sal` | Shift Left Arithmetic | `0` (same as `shl`) |

### Why `sar` matters for signed values

```nasm
mov eax, -8         ; eax = 0xFFFFFFF8 (negative, MSB = 1)
sar eax, 1          ; eax = 0xFFFFFFFC = -4  (correct: -8 ÷ 2 = -4)
```

With `shr`:
```nasm
mov eax, -8         ; eax = 0xFFFFFFF8
shr eax, 1          ; eax = 0x7FFFFFFC = +2147483644  (wrong — sign bit zeroed out)
```

**Rule: use `sar`/`sal` for signed values, `shr`/`shl` for unsigned.**

---

## Performance Note

Shifts are faster than `mul`/`div` for powers of 2. Prefer:

```nasm
shl eax, 1      ; over: imul eax, 2
shr eax, 1      ; over: idiv eax... (though div needs edx setup too)
```

Only works for exact powers of 2. For anything else, use `mul`/`div`.

---

## Complete Example

```nasm
section .text
    global _start

_start:
    mov eax, 10
    shr eax, 1          ; eax = 5  (10 ÷ 2)

    mov eax, 3
    shl eax, 2          ; eax = 12 (3 × 4 = 3 × 2^2)

    mov eax, -8
    sar eax, 1          ; eax = -4 (-8 ÷ 2, sign preserved)

    mov eax, 1
    int 0x80
```

---

## Key Takeaways

- `shr` = logical right shift = divide by 2 (unsigned)
- `shl` = logical left shift = multiply by 2
- `shr/shl N` = divide/multiply by `2^N`
- Shifted-out bit always goes to **CF**
- `sar` = signed right shift — preserves sign bit, use for signed division by 2
- `shl` and `sal` behave identically
- Shifts are faster than `mul`/`div` — use them whenever dividing/multiplying by a power of 2

---

## What's Next
Control flow — `cmp`, conditional jumps, and loops.
