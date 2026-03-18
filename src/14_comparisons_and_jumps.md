# Lecture 14 — Comparisons & Conditional Jumps

> Covers `cmp`, how it sets EFLAGS via subtraction, the full set of conditional jump instructions, labels, and the critical rule about always having an else-path to avoid falling through.

---

## `cmp` — Compare

```nasm
cmp operand1, operand2
```

Internally performs `operand1 - operand2`, **discards the result**, but keeps the EFLAGS update. You never see the subtraction result — you only use the flags it sets.

### What the flags tell you

| Subtraction result | Meaning | Flags set |
|---|---|---|
| Positive | operand1 > operand2 | — |
| Zero | operand1 == operand2 | ZF = 1 |
| Negative | operand1 < operand2 | SF = 1 |

> For signed comparisons with negative numbers, the CPU also checks OF (overflow flag) alongside SF. You don't need to handle this manually — the conditional jump instructions account for it automatically.

---

## Labels

Labels are named addresses in your code. Any instruction can jump to them.

```nasm
_start:             ; entry point label (you know this one)
    ...

lesser:             ; user-defined label
    mov ecx, 1

end:                ; another label
    mov eax, 1
    int 0x80
```

Labels are just markers — they don't execute anything by themselves. Code falls through them unless you jump.

---

## Conditional Jumps

Jump instructions check EFLAGS and either jump to a label or continue to the next instruction.

### Full jump reference

| Instruction | Full name | Jumps when... |
|---|---|---|
| `je` | Jump if Equal | `operand1 == operand2` (ZF = 1) |
| `jne` | Jump if Not Equal | `operand1 != operand2` (ZF = 0) |
| `jl` | Jump if Less | `operand1 < operand2` (signed) |
| `jle` | Jump if Less or Equal | `operand1 <= operand2` (signed) |
| `jg` | Jump if Greater | `operand1 > operand2` (signed) |
| `jge` | Jump if Greater or Equal | `operand1 >= operand2` (signed) |
| `jz` | Jump if Zero | result == 0 (ZF = 1) |
| `jnz` | Jump if Not Zero | result != 0 (ZF = 0) |
| `jmp` | Unconditional Jump | Always |

> `je` and `jz` are equivalent — both check ZF. Same flags, different mnemonics.

---

## Basic Structure: If / Else

```nasm
    cmp eax, ebx
    jl lesser           ; if eax < ebx → jump to lesser

    ; else path (eax >= ebx)
    mov ecx, 0
    jmp end             ; MUST jump to end — otherwise falls into 'lesser'

lesser:
    mov ecx, 1

end:
    mov eax, 1
    int 0x80
```

### Critical rule: always handle the non-jump path

If `jl` doesn't fire, execution continues to the next line — which would be the `lesser:` block. You **must** add an unconditional `jmp end` in the else path to skip over the other branch.

Forgetting this = both branches execute = bugs.

---

## Full Example Programs

### Case 1: `eax > ebx` — jump NOT taken

```nasm
section .text
    global _start

_start:
    mov eax, 3
    mov ebx, 2
    cmp eax, ebx
    jl lesser           ; 3 < 2 is false → skip
    jmp end             ; fall to end

lesser:
    mov ecx, 1          ; skipped

end:
    mov eax, 1
    int 0x80
```

### GDB walkthrough

```
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi             ; mov eax, 3
(gdb) stepi             ; mov ebx, 2
(gdb) stepi             ; cmp eax, ebx  → EFLAGS updated
(gdb) stepi             ; jl lesser     → NOT taken (3 is not < 2)
(gdb) stepi             ; jmp end       → taken
; execution jumps to 'end', skips 'lesser' block entirely
```

### Case 2: `eax < ebx` — jump taken

```nasm
_start:
    mov eax, 1          ; changed to 1
    mov ebx, 2
    cmp eax, ebx
    jl lesser           ; 1 < 2 is true → jump taken
    jmp end

lesser:
    mov ecx, 1          ; THIS executes now

end:
    mov eax, 1
    int 0x80
```

### GDB walkthrough

```
(gdb) stepi × 3         ; load regs, cmp
(gdb) stepi             ; jl lesser → TAKEN (1 < 2)
; execution jumps directly to 'lesser:' label
(gdb) stepi             ; mov ecx, 1
(gdb) info registers ecx
# ecx = 0x1    1        ← lesser block executed
```

---

## Execution Flow Diagram

```
    cmp eax, ebx
         │
    ┌────┴────┐
    │  jl ?   │
    └────┬────┘
    false│         true
         │              └──→ lesser:
    jmp end                  mov ecx, 1
         │                       │
         └──────────┐            │
                  end:◄──────────┘
                  int 0x80
```

---

## Key Takeaways

- `cmp` = subtraction that only updates EFLAGS, never stores the result
- Conditional jumps read EFLAGS set by the most recent `cmp` (or any arithmetic instruction)
- `je`/`jz` are identical — both check ZF
- Labels are just named code addresses — execution falls through them unless you jump
- **Always add `jmp` in the non-taken path** to skip over other branches — falling through is a common bug
- `jmp label` is unconditional — always taken

---

## What's Next
Loops — using `cmp` + jumps + a counter register to repeat instructions.
