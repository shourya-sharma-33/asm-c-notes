# Lecture 09 — Subtraction & Negative Numbers

> Covers the `sub` instruction, how negative results are represented in registers (two's complement), which EFLAGS get set, and why negative numbers in assembly work exactly like you'd expect arithmetically.

---

## The `sub` Instruction

```nasm
sub destination, source     ; destination = destination - source
```

Result stored in `destination`. Source unchanged.

### Simple example — positive result

```nasm
_start:
    mov eax, 5
    mov ebx, 3
    sub eax, ebx            ; eax = 5 - 3 = 2
```

### GDB verification

```
(gdb) stepi × 3
(gdb) info registers eax
# eax = 0x2    2            ← correct
```

No surprises. Unlike addition, subtraction can't produce a result *larger* than the register — numbers only get smaller — so no overflow concern in the same way.

---

## Negative Results — Two's Complement

```nasm
_start:
    mov eax, 3
    mov ebx, 5
    sub eax, ebx            ; eax = 3 - 5 = -2
```

### GDB result

```
(gdb) stepi × 3
(gdb) info registers eax
# eax = 0xfffffffe    -2
```

The register contains `0xFFFFFFFE` — a long string of F's followed by E. This is **two's complement** representation of -2.

**Why all the F's?** Subtraction borrows when the top number is smaller. `3 - 5` borrows all the way through 32 bits, producing that pattern. It's not arbitrary — it's mathematically consistent.

**How does GDB know to print `-2` and not just a large positive number?** It checks EFLAGS.

---

## EFLAGS After a Negative Result

```
(gdb) info registers eflags
# eflags = ...    [ CF AF SF IF ]
```

| Flag | Why set |
|---|---|
| `CF` | Carry flag — in subtraction, CF = **borrow flag**. Set because we had to borrow. |
| `SF` | **Sign flag** — set to 1 when result is negative (MSB = 1). This is the key one. |
| `AF` | Auxiliary flag — borrow out of bit 3 |

**`SF = 1` means the result is negative.** That's how you (and GDB) know `0xFFFFFFFE` should be interpreted as `-2` and not `4294967294`.

---

## Two's Complement Arithmetic Just Works

The beauty of two's complement: you don't need special instructions for negative numbers. Regular `add` and `sub` handle them correctly.

### Example: -2 + 2 = 0

```nasm
_start:
    mov eax, 3
    mov ebx, 5
    sub eax, ebx            ; eax = -2  (0xFFFFFFFE)

    mov ebx, 2
    add eax, ebx            ; eax = -2 + 2 = 0
```

### Why it works at the bit level

```
  0xFFFFFFFE   (-2 in two's complement)
+ 0x00000002   (2)
------------
  0x00000000   (0, carry discarded)
```

`E + 2 = 10 hex` → writes `0`, carries `1`. That carry propagates through all the F's, flipping them to 0. Result: `0x00000000`. Exactly right.

### GDB confirmation

```
(gdb) stepi × 5
(gdb) info registers eax
# eax = 0x0    0            ← -2 + 2 = 0, correct
```

Same principle extends: if you added `3` instead of `2`, you'd get `1`. The arithmetic flows exactly as expected.

---

## When Do You Actually Need to Check EFLAGS?

In most cases you don't — the math just works. You need to check EFLAGS when you want to **branch based on the result**:

- Was the result negative? → check `SF`
- Did the subtraction require a borrow? → check `CF`
- Was the result zero? → check `ZF`

These flags are what make conditional jumps (`jl`, `jg`, `jz`, etc.) work — covered when we get to control flow.

---

## `add` vs `sub` EFLAGS Comparison

| Operation | CF set when... | SF set when... | ZF set when... |
|---|---|---|---|
| `add` | Carry out of MSB (overflow) | Result MSB = 1 | Result = 0 |
| `sub` | Borrow needed | Result MSB = 1 (negative) | Result = 0 |

CF plays a dual role — carry for addition, borrow for subtraction.

---

## Key Takeaways

- `sub dst, src` → `dst = dst - src`
- Negative results stored as two's complement — all F's pattern is normal
- `SF = 1` means the result is negative — that's how you detect it
- `CF = 1` after subtraction means a borrow occurred
- Negative numbers work identically to what you'd expect arithmetically — no special handling needed for `add`/`sub`
- The only time you explicitly check `SF` is when you need to branch on whether a result was negative

---

## What's Next
Multiplication and division — these are more complex and have different register conventions.
