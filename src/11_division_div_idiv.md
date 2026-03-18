# Lecture 11 — Division: `div` and `idiv`

> Covers integer division in x86 — how the operands work, where the quotient and remainder land, and the signed variant `idiv`.

---

## Integer Division

x86 division is **integer division** — it always produces:
- A **quotient** (whole number result)
- A **remainder**

Example: `11 ÷ 2 = 5 remainder 1` because `5 × 2 = 10`, closest to 11 without exceeding it.

---

## `div` — Unsigned Division

### Syntax

```nasm
div source      ; one operand — the divisor
                ; EAX is the implicit dividend
```

**Register roles:**

| Register | Role | Before | After |
|---|---|---|---|
| `eax` | Dividend (input) / Quotient (output) | value being divided | result of division |
| `edx` | Remainder (output) | must be 0 | remainder |
| `source` | Divisor | value dividing by | unchanged |

> **Important:** `edx` must be zeroed before division, otherwise the CPU treats `edx:eax` as a 64-bit dividend and you get unexpected results or a divide error.

### Example

```nasm
section .text
    global _start

_start:
    mov eax, 11         ; dividend
    mov edx, 0          ; zero edx — required before div
    mov ecx, 2          ; divisor
    div ecx             ; eax = 11 ÷ 2 = 5, edx = remainder = 1

    mov eax, 1
    int 0x80
```

### GDB session

```
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi × 4         ; step through all four instructions

(gdb) info registers eax
# eax = 0x5    5        ← quotient: 11 ÷ 2 = 5

(gdb) info registers edx
# edx = 0x1    1        ← remainder: 11 mod 2 = 1
```

---

## Result Placement Summary

| Result | Register |
|---|---|
| Quotient | `eax` |
| Remainder | `edx` |

Same pattern as multiplication's auto-expansion — the A register is always central, with `edx` as the overflow/extra register.

---

## `idiv` — Signed Division

Same as `div` but interprets values as signed two's complement.

```nasm
mov eax, 0xFF       ; unsigned: 255 | signed: -1
cdq                 ; sign-extend eax into edx:eax (use instead of mov edx, 0)
mov ecx, 2
idiv ecx            ; -1 ÷ 2 = 0 remainder -1 (signed integer division)
```

> For signed division, use `cdq` (Convert Double to Quad) instead of `mov edx, 0` to correctly sign-extend `eax` into `edx`. Using `mov edx, 0` would break signed division for negative dividends.

---

## `div` vs `idiv`

| | `div` | `idiv` |
|---|---|---|
| Interpretation | Unsigned | Signed (two's complement) |
| `0xFF` in `al` | 255 | -1 |
| Clear `edx` with | `mov edx, 0` | `cdq` |

---

## Arithmetic Instructions — Complete Summary

| Instruction | Operation | Notes |
|---|---|---|
| `add dst, src` | `dst = dst + src` | Use `adc` to capture carry |
| `sub dst, src` | `dst = dst - src` | CF = borrow, SF = negative result |
| `mul src` | `eax = eax × src` (unsigned) | Result auto-expands |
| `imul src` | `eax = eax × src` (signed) | Same expansion rules |
| `div src` | `eax = eax ÷ src`, `edx = remainder` (unsigned) | Zero `edx` first |
| `idiv src` | `eax = eax ÷ src`, `edx = remainder` (signed) | Use `cdq` first |

---

## Key Takeaways

- `div`/`idiv` take **one operand** — `eax` is the implicit dividend
- Quotient → `eax`, Remainder → `edx`
- Always zero `edx` before `div` (or use `cdq` before `idiv`)
- `div` = unsigned, `idiv` = signed — same bit pattern, different interpretation
- x86 has no modulo instruction — the remainder from `div`/`idiv` is your modulo

---

## What's Next
Control flow — jumps, comparisons, and loops. This is where EFLAGS start doing real work.
