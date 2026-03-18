# Lecture 16 — Floating Point Numbers

> Covers single-precision floats in x86: the XMM registers, `movss`/`addss` instructions, and the IEEE 754 precision problem — why `2.1` stored in memory isn't exactly `2.1`.

---

## Single vs Double Precision

| Type | Bits | x86 register |
|---|---|---|
| Single precision | 32 bits | XMM (fits in 32-bit x86) |
| Double precision | 64 bits | XMM (needs 64-bit extension) |

We use **single precision** here because it fits natively in 32-bit x86. Double precision would require 64-bit registers or special handling.

---

## Declaring Floats in `.data`

Same `dd` directive as a 32-bit integer — NASM infers float from the value:

```nasm
section .data
    x    dd    3.14
    y    dd    2.1
```

`dd` = 32 bits = single precision float. NASM encodes the value as IEEE 754 at assembly time.

---

## XMM Registers

Floats do **not** go into `eax`/`ebx`/etc. They use dedicated **XMM registers**:

```
xmm0, xmm1, xmm2, ... xmm15
```

16 registers, each 128 bits wide. Can hold:
- **Scalar** — one single float (what we use here)
- **Packed** — multiple floats at once (SIMD, not covered here)

---

## `movss` — Move Scalar Single

```nasm
movss destination, source   ; move one 32-bit float
```

```nasm
movss xmm0, [x]     ; load 3.14 into xmm0
movss xmm1, [y]     ; load 2.1  into xmm1
```

`ss` = **S**calar **S**ingle precision. The brackets dereference the address as usual.

---

## Float Arithmetic Instructions

Same structure as integer instructions — just different mnemonics:

| Instruction | Operation |
|---|---|
| `addss xmm0, xmm1` | `xmm0 = xmm0 + xmm1` |
| `subss xmm0, xmm1` | `xmm0 = xmm0 - xmm1` |
| `mulss xmm0, xmm1` | `xmm0 = xmm0 × xmm1` |
| `divss xmm0, xmm1` | `xmm0 = xmm0 ÷ xmm1` |

Result stored in destination (`xmm0`). Source unchanged.

---

## Full Example Program

```nasm
section .data
    x    dd    3.14
    y    dd    2.1

section .text
    global _start

_start:
    movss xmm0, [x]         ; xmm0 = 3.14
    movss xmm1, [y]         ; xmm1 = 2.1
    addss xmm0, xmm1        ; xmm0 = 3.14 + 2.1 = 5.24 (approx)

    mov eax, 1
    mov ebx, 1
    int 0x80
```

Note: you can still use `eax`/`ebx` alongside XMM registers — they're separate register sets.

---

## GDB — Printing Float Registers

Standard `info registers xmm0` shows raw bytes. To see the float value:

```
(gdb) p $xmm0.v4_float[0]
```

- `$xmm0` = the register
- `.v4_float` = interpret as four packed floats
- `[0]` = the first (lowest) one — which is where `movss` puts the value

### GDB session

```
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi             ; movss xmm0, [x]
(gdb) p $xmm0.v4_float[0]
# $1 = 3.14000010       ← not exactly 3.14

(gdb) stepi             ; movss xmm1, [y]
(gdb) p $xmm1.v4_float[0]
# $2 = 2.09999990       ← not exactly 2.1

(gdb) stepi             ; addss xmm0, xmm1
(gdb) p $xmm0.v4_float[0]
# $3 = 5.23999977       ← not exactly 5.24
```

---

## The IEEE 754 Precision Problem

Floats are stored in **IEEE 754** format — a binary encoding of decimal numbers using powers of two.

**The problem:** most decimal fractions can't be represented exactly in binary. `0.1` in binary is `0.0001100110011...` — an infinite repeating sequence. Stored in 32 bits, it gets truncated.

```
3.14  stored as  3.14000010...
2.1   stored as  2.09999990...
sum   stored as  5.23999977...    (expected 5.24)
```

This is not a bug — it's fundamental to how floating point works in every language. C, Rust, Go — all do this.

### Consequence: never compare floats for exact equality

```c
// Wrong
if (a == b) { ... }

// Correct — check if difference is within acceptable epsilon
if (fabs(a - b) < 0.0001) { ... }
```

Same principle applies in assembly. Checking `xmm0 == xmm1` with exact equality will silently fail for most real-world values.

---

## Mental Model

```
What you write:    3.14
What's stored:     3.14000010...  (closest representable IEEE 754 value)
Error:             ~0.00000010    (about 1 × 10^-7 for 32-bit single)
```

Single precision gives you ~7 significant decimal digits of accuracy. For more precision, use double (`dd` becomes `dq`, `movss` becomes `movsd`, `addss` becomes `addsd`).

---

## Key Takeaways

- Single precision floats = 32 bits = declared with `dd` in `.data`
- Use XMM registers (`xmm0`–`xmm15`), not `eax`/`ebx`
- `movss` loads a float; `addss`/`subss`/`mulss`/`divss` for arithmetic
- `ss` suffix = Scalar Single precision
- Floats are almost never exact — IEEE 754 encoding has inherent rounding
- Never compare floats with exact equality — use an epsilon threshold
- GDB: `p $xmm0.v4_float[0]` to print the float value

---

## What's Next
Functions — `call`, `ret`, the stack, and how arguments are passed.
