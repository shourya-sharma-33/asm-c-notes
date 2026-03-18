# Lecture 17 — Floating Point Comparisons

> Covers `ucomiss` for comparing floats and the different set of conditional jumps required — `ja`/`jb` instead of `jg`/`jl`.

---

## Why `cmp` Doesn't Work for Floats

`cmp` operates on integer registers (`eax`, `ebx`, etc.). XMM registers need their own compare instruction.

---

## `ucomiss` — Unordered Compare Scalar Single

```nasm
ucomiss xmm0, xmm1      ; compare xmm0 against xmm1, set EFLAGS
```

Works like `cmp` — performs the comparison, updates EFLAGS, discards the result. You then follow with a conditional jump.

`ss` = Scalar Single precision, same as `movss`/`addss`.

---

## Different Jumps for Float Comparisons

After `ucomiss`, you cannot use `jg`/`jl`/`jge`/`jle`. Those are for signed integer comparisons. Use **above/below** variants instead:

| Instruction | Meaning | Float equivalent of |
|---|---|---|
| `ja` | Jump if Above | `>` (greater than) |
| `jae` | Jump if Above or Equal | `>=` |
| `jb` | Jump if Below | `<` (less than) |
| `jbe` | Jump if Below or Equal | `<=` |
| `je` | Jump if Equal | `==` (still works) |

> **Memory aid:** Above = greater, Below = less. These are unsigned/unordered comparisons — they check CF and ZF directly rather than SF/OF, which is why they work correctly with `ucomiss`.

---

## Full Example

```nasm
section .data
    x    dd    3.14
    y    dd    2.1

section .text
    global _start

_start:
    movss xmm0, [x]         ; xmm0 = 3.14
    movss xmm1, [y]         ; xmm1 = 2.1
    ucomiss xmm0, xmm1      ; compare: is xmm0 > xmm1?

    ja greater              ; if xmm0 > xmm1 → jump to greater
    jmp end                 ; else → skip to end

greater:
    mov ecx, 1              ; mark that xmm0 was greater

end:
    mov eax, 1
    mov ebx, 1
    int 0x80
```

Since `3.14 > 2.1`, the `ja` fires and execution jumps to `greater`.

---

## GDB Session

```bash
nasm -f elf -o float_cmp.o float_cmp.s
ld -m elf_i386 -o float_cmp float_cmp.o
gdb float_cmp
```

```
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi             ; movss xmm0, [x]
(gdb) stepi             ; movss xmm1, [y]
(gdb) stepi             ; ucomiss xmm0, xmm1 → EFLAGS updated
(gdb) stepi             ; ja greater → TAKEN (3.14 > 2.1)
; execution jumps to 'greater' label
(gdb) stepi             ; mov ecx, 1
(gdb) info registers ecx
# ecx = 0x1    1        ← greater branch executed
```

---

## Integer vs Float Jump Comparison

| Scenario | Compare | Greater | Less | Equal |
|---|---|---|---|---|
| Integer | `cmp` | `jg` | `jl` | `je` |
| Float | `ucomiss` | `ja` | `jb` | `je` |

`je` is the same in both cases. Only the greater/less variants change.

---

## Key Takeaways

- `ucomiss xmm0, xmm1` = float compare, sets EFLAGS
- Use `ja`/`jae`/`jb`/`jbe` after `ucomiss` — not `jg`/`jl`
- `je` still works for equality (but remember float precision — exact equality is rare)
- The `ss` suffix pattern is consistent: `movss`, `addss`, `ucomiss` — always scalar single precision

---

## What's Next
Functions — `call`, `ret`, and the stack.
