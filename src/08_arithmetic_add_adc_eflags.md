# Lecture 08 — Arithmetic: `add`, EFLAGS, and `adc`

> Covers the `add` instruction, what EFLAGS flags get set and why, what happens when addition overflows a register (carry), and how `adc` (add with carry) recovers the lost bit.

---

## The `add` Instruction

```nasm
add destination, source     ; destination = destination + source
```

Result is stored back in `destination`. Source is unchanged.

### Simple example

```nasm
section .text
    global _start

_start:
    mov eax, 1
    mov ebx, 2
    add eax, ebx            ; eax = eax + ebx = 3

    mov ebx, eax            ; move result to ebx for exit code
    mov eax, 1
    int 0x80
```

### GDB verification

```bash
gdb arith
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi                 ; mov eax, 1
(gdb) stepi                 ; mov ebx, 2
(gdb) stepi                 ; add eax, ebx
(gdb) info registers eax
# eax = 0x3    3            ← correct
```

---

## EFLAGS — What Gets Set After `add`

After every arithmetic operation, the CPU updates EFLAGS automatically. Check it in GDB:

```
(gdb) info registers eflags
# eflags = 0x206    [ PF IF ]
```

The bracketed flags are the ones currently set to 1.

### Common flags after `add 1 + 2 = 3`

| Flag | Name | Set because... |
|---|---|---|
| `PF` | Parity Flag | Result (3) is odd |
| `IF` | Interrupt Flag | Interrupts are enabled — set at program start, not by `add` |

**Parity Flag logic:**
- Result is odd → `PF = 1`
- Result is even → `PF = 0`

Two uses of parity: checking even/odd in logic, or detecting single-bit data transmission errors (a flipped bit changes parity).

---

## Binary Literal Notation

NASM lets you write binary values directly using the `0b` prefix:

```nasm
mov al, 0b11111111      ; al = 0xFF = 255 (eight 1-bits)
mov bl, 0b00000001      ; bl = 1
```

---

## The Overflow Problem — Carry Flag

What happens when addition produces a result too large for the register?

```nasm
_start:
    mov al, 0b11111111      ; al = 255 (max 8-bit value)
    mov bl, 0b00000001      ; bl = 1
    add al, bl              ; 255 + 1 = 256, but al is only 8 bits
```

### What happens mathematically:

```
  1111 1111   (255)
+ 0000 0001   (1)
-----------
1 0000 0000   (256, but the 1 is in the 9th bit position)
```

`al` only holds 8 bits. The ninth bit has nowhere to go inside the register.

### GDB result:

```bash
(gdb) stepi
(gdb) stepi
(gdb) stepi                 ; add al, bl
(gdb) info registers al
# al = 0x0    0             ← looks wrong — result is 0
(gdb) info registers eax
# eax = 0x0   0             ← al is the low byte of eax, also 0
```

The `256` result appears to be `0`. The bit didn't disappear though — it went into **EFLAGS**:

```
(gdb) info registers eflags
# eflags = 0x217    [ CF AF PF IF ZF ]
```

| Flag | Why it's set |
|---|---|
| `CF` | **Carry Flag** — there was a carry out of the top bit (the 9th bit had a 1) |
| `ZF` | **Zero Flag** — the register result is 0 |
| `PF` | Parity Flag — result (0) has even parity |
| `AF` | Auxiliary Flag — carry out of bit 3; used in BCD math, rarely needed |

**The carry flag is where the overflowed bit lives.**

---

## Recovering the Carry — `adc` (Add with Carry)

`adc` = add, but also tacks on whatever is in the carry flag:

```nasm
adc destination, source     ; destination = destination + source + CF
```

### Full example — 8-bit addition capturing carry into AH

```nasm
section .text
    global _start

_start:
    mov al, 0b11111111      ; al = 255
    mov bl, 0b00000001      ; bl = 1
    add al, bl              ; al = 0, CF = 1 (carry)

    adc ah, 0               ; ah = ah + 0 + CF = 0 + 0 + 1 = 1
                            ; the carry is now captured in ah

    mov ebx, eax            ; eax now = ah:al = 0x0100 = 256
    mov eax, 1
    int 0x80
```

### GDB session

```bash
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi                 ; mov al, 255
(gdb) stepi                 ; mov bl, 1
(gdb) stepi                 ; add al, bl  → al=0, CF=1
(gdb) stepi                 ; adc ah, 0   → ah=1

(gdb) info registers ah
# ah = 0x1    1             ← carry captured here

(gdb) info registers eax
# eax = 0x100    256        ← ah:al combined = 256
```

`eax = 256` — which is the correct result of `255 + 1`. The carry that overflowed `al` was preserved in `CF` and then pulled into `ah` via `adc`.

---

## EFLAGS Summary (so far)

| Flag | Abbrev | Set when... |
|---|---|---|
| Carry | `CF` | Result produced a carry/borrow out of the MSB |
| Zero | `ZF` | Result is exactly 0 |
| Sign | `SF` | Result's MSB is 1 (negative in signed interpretation) |
| Parity | `PF` | Result has odd number of 1-bits |
| Overflow | `OF` | Signed arithmetic overflowed |
| Auxiliary | `AF` | Carry out of bit 3 (BCD use, rarely relevant) |
| Interrupt | `IF` | CPU will accept hardware interrupts (usually always on) |

---

## Key Takeaways

- `add dst, src` → `dst = dst + src`, EFLAGS updated automatically
- Binary literals: `0b11111111` = 255
- When a result is too large for the register, the overflow bit goes into **CF (Carry Flag)**
- The register result will appear wrong (truncated) without checking CF
- `adc dst, src` = `add` but also adds CF — use it to capture the carry into the next register
- `ZF` set = result was zero. `CF` set = there was a carry out.

---

## What's Next
More arithmetic instructions — subtraction, multiplication, division — and more EFLAGS usage.
