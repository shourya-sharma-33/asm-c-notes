# Lecture 15 — Loops & Array Iteration

> First real functional program: sum all values in an array using a loop built from `cmp` + `jne` + `jmp`. Introduces the `inc` instruction and indexed memory access.

---

## Building a Loop from Primitives

x86 has no dedicated loop-with-condition instruction (well, it has `loop`, but it's rarely used). You build loops from:

1. A **counter** register (index into array)
2. A **label** to jump back to
3. `cmp` + conditional jump to exit when done
4. `jmp` back to the top

---

## The Problem: Sum an Array

```nasm
section .data
    list    db    1, 2, 3, 4      ; array of 4 bytes
```

Goal: add all values, store the sum.

---

## Setup

```nasm
section .text
    global _start

_start:
    mov eax, 0          ; eax = current index (starts at 0)
    mov cl, 0           ; cl  = running sum (starts at 0)
```

`eax` = index. `cl` = sum accumulator (byte register, fine since sum of 1+2+3+4=10 fits in 8 bits).

---

## Indexed Memory Access

To get element at index `i` from `list`:

```nasm
mov bl, [list + eax]    ; bl = list[eax]
```

- `list` = base address of the array
- `eax` = byte offset (index × 1 for byte arrays)
- When `eax=0`: loads `list[0]` = 1
- When `eax=1`: loads `list[1]` = 2
- etc.

For arrays of larger types (words, dwords), multiply index by element size: `[list + eax*2]` for word arrays.

---

## `inc` — Increment by 1

```nasm
inc eax         ; eax = eax + 1
```

Equivalent to `add eax, 1` but shorter. Common in loop counters.

---

## Full Loop Program

```nasm
section .data
    list    db    1, 2, 3, 4

section .text
    global _start

_start:
    mov eax, 0              ; index = 0
    mov cl, 0               ; sum = 0

loop:
    mov bl, [list + eax]    ; bl = list[index]
    add cl, bl              ; sum += list[index]
    inc eax                 ; index++
    cmp eax, 4              ; index == 4? (end of array)
    jne loop                ; if not, loop again
    jmp end                 ; if yes, exit

end:
    mov eax, 1
    mov ebx, 1
    int 0x80
```

### Loop flow

```
index=0: bl=1, sum=1,  eax=1 → 1≠4 → loop
index=1: bl=2, sum=3,  eax=2 → 2≠4 → loop
index=2: bl=3, sum=6,  eax=3 → 3≠4 → loop
index=3: bl=4, sum=10, eax=4 → 4=4 → jmp end
```

---

## GDB Session

```bash
nasm -f elf -o loop.o loop.s
ld -m elf_i386 -o loop loop.o
gdb loop
```

```
(gdb) layout asm
(gdb) break _start
(gdb) run
```

**Iteration 1:**
```
(gdb) stepi × 3         ; set eax=0, cl=0, load bl
(gdb) info registers bl
# bl = 0x1    1         ← list[0]

(gdb) stepi             ; add cl, bl
(gdb) info registers cl
# cl = 0x1    1         ← sum so far

(gdb) stepi             ; inc eax
(gdb) info registers eax
# eax = 0x1    1        ← index now 1

(gdb) stepi             ; cmp eax, 4 → 1 ≠ 4
(gdb) stepi             ; jne loop → taken
```

**Iteration 2:**
```
(gdb) stepi             ; mov bl, [list + eax]
(gdb) info registers bl
# bl = 0x2    2         ← list[1]
```

**After all 4 iterations:**
```
(gdb) info registers cl
# cl = 0xa    10        ← 1+2+3+4 = 10

(gdb) info registers eax
# eax = 0x4    4        ← index hit 4, exiting
```

---

## Handling Unknown Array Length

If you don't know the array size at compile time, use a **null terminator** as the exit condition:

```nasm
section .data
    list    db    1, 2, 3, 4, 0    ; 0 = sentinel / end marker

loop:
    mov bl, [list + eax]
    cmp bl, 0               ; hit the null terminator?
    je end                  ; yes → exit
    add cl, bl
    inc eax
    jmp loop
```

Check for the sentinel *before* adding to the sum, otherwise you'll add the terminator value itself.

---

## Loop Pattern — General Template

```nasm
    mov eax, 0              ; counter = 0

loop_label:
    ; --- loop body ---
    ; do work with [array + eax]
    ; ----------------
    inc eax
    cmp eax, LENGTH         ; or: cmp sentinel
    jne loop_label          ; keep going
    ; falls through to code after loop
```

---

## Key Takeaways

- Loops = label + body + `inc` counter + `cmp` + conditional jump back
- `[list + eax]` = indexed array access — `eax` is the byte offset
- `inc reg` = `reg + 1`, idiomatic for loop counters
- For known-length arrays: `cmp index, LENGTH` + `jne`
- For unknown-length arrays: null-terminate and check for sentinel each iteration
- Register sizes matter: use `bl`/`cl` for byte arrays, `ebx`/`ecx` for dword arrays — mismatching sizes causes the packed-byte problem from Lecture 05

---

## What's Next
Functions — `call`, `ret`, and the stack.
