# Lecture 10 — Multiplication: `mul` and `imul`

> Covers the `mul` (unsigned) and `imul` (signed) instructions, why only one operand is provided, how the destination register auto-expands to fit large results, and the critical difference between signed and unsigned interpretation.

---

## Two Multiplication Instructions

| Instruction | Treats values as | Use when |
|---|---|---|
| `mul src` | Unsigned | Values are always positive |
| `imul src` | Signed (two's complement) | Values may be negative |

---

## `mul` — Unsigned Multiply

### Syntax

```nasm
mul source      ; only ONE operand — the non-A register
```

**The A register is always the implicit second operand.** `mul` automatically multiplies whatever is in `AL`/`AX`/`EAX` by `source`, and stores the result back in the A register.

This is because `EAX` is the **accumulator** — x86 designates it as the default destination for multiplication.

### Simple example

```nasm
_start:
    mov al, 2
    mov bl, 3
    mul bl              ; al = al × bl = 2 × 3 = 6, result in al
```

### GDB session

```
(gdb) stepi × 3
(gdb) info registers al
# al = 0x6    6          ← correct
```

---

## What Happens When the Result Overflows

Unlike `add` (which loses the overflow bit to CF), `mul` **automatically expands the destination** to fit the result.

```nasm
_start:
    mov al, 0xFF        ; al = 255 (max 8-bit unsigned)
    mov bl, 2
    mul bl              ; 255 × 2 = 510 — doesn't fit in 8 bits
```

### GDB session

```
(gdb) stepi × 3
(gdb) info registers al
# al = 0xfe    -2        ← lower 8 bits only, looks wrong

(gdb) info registers ah
# ah = 0x1    1          ← upper 8 bits — the overflow went here

(gdb) info registers ax
# ax = 0x1fe    510      ← full result: ah:al combined = 510
```

**The result expanded from `al` (8-bit) to `ax` (16-bit) automatically.**

### Auto-expansion rules for `mul`

| Operand size | Implicit source | Result stored in |
|---|---|---|
| 8-bit (`bl`) | `al` | `ax` (16-bit) |
| 16-bit (`bx`) | `ax` | `dx:ax` (32-bit) |
| 32-bit (`ebx`) | `eax` | `edx:eax` (64-bit) |

When the result is too big for the source register, the high half spills into the next register up (`ah`, `dx`, `edx`).

> This is the key advantage over `add`: you never lose bits. The destination grows to fit.

---

## `imul` — Signed Multiply

Same syntax, but interprets the operand as a **signed** two's complement value.

```nasm
_start:
    mov al, 0xFF        ; unsigned: 255 | signed: -1
    mov bl, 2
    imul bl             ; interprets al as -1, result = -1 × 2 = -2
```

### GDB session

```
(gdb) info registers al       ; before multiply
# al = 0xff    -1             ← GDB shows -1 because SF is considered

(gdb) stepi                   ; imul bl
(gdb) info registers al
# al = 0xfe    -2             ← -1 × 2 = -2, correct
```

### Comparison: `mul` vs `imul` on `0xFF`

| Instruction | Interprets `0xFF` as | Result of `× 2` |
|---|---|---|
| `mul bl` | `255` (unsigned) | `510` (stored in `ax`) |
| `imul bl` | `-1` (signed) | `-2` (stored in `al`) |

**Same bits, completely different result** — the instruction determines the interpretation.

---

## Full Example Program

```nasm
section .text
    global _start

_start:
    mov al, 0xFF        ; 255 unsigned / -1 signed
    mov bl, 2

    ; Unsigned:
    ; mul bl            ; ax = 510

    ; Signed:
    imul bl             ; al = -2

    mov eax, 1
    int 0x80
```

---

## Key Takeaways

- `mul`/`imul` take **one operand** — the A register is the implicit second operand
- Result always goes into the A register (expands if needed: `al` → `ax` → `dx:ax` → `edx:eax`)
- `mul` = unsigned. `0xFF` = 255.
- `imul` = signed. `0xFF` = -1.
- Unlike `add`, multiplication **never loses bits** — the destination auto-expands
- Use `ax` (not just `al`) to see the full result when multiplying 8-bit values

---

## What's Next
Division — similar structure to multiplication (uses the A register implicitly, has signed/unsigned variants), but with a key difference: the remainder.
