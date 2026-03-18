# Lecture 04 — The `.data` Section & Variables

> Covers declaring variables in `.data`, the different size directives, and the critical distinction between **an address** and **the value at that address** — the classic beginner trap. Full GDB session included.

---

## Declaring Data in `.data`

Every variable declaration needs three things:

```nasm
name    size_directive    initial_value
```

```nasm
section .data
    num    dd    5        ; name=num, size=4 bytes (dd), value=5
```

### Size Directives

| Directive | Meaning | Size | Bits |
|---|---|---|---|
| `db` | Define Byte | 1 byte | 8 |
| `dw` | Define Word | 2 bytes | 16 |
| `dd` | Define Doubleword | 4 bytes | 32 |
| `dq` | Define Quadword | 8 bytes | 64 |
| `dt` | Define Ten bytes | 10 bytes | 80 |

> There are no types in assembly — no int, no float, no char. Everything is just bytes. The size directive only controls **how much space** is reserved. What those bytes *mean* is entirely up to you.

---

## The Address vs Value Trap

This is the most important concept in this lecture.

### Wrong approach (what you'd naively try first):

```nasm
section .data
    num    dd    5

section .text
    global _start

_start:
    mov eax, 1
    mov ebx, num        ; WRONG — this moves the ADDRESS of num into ebx, not 5
    int 0x80
```

Running this and checking `echo $?` gives **0**, not 5. Why?

When you write `num` in NASM without brackets, it resolves to the **memory address** where `num` lives (e.g. `0x804a000`). You're putting the address into `ebx`, not the value `5` stored there.

### GDB proof

```bash
gdb data
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) stepi
(gdb) info registers ebx
# ebx = 0x804a000    (some address — not 5)
```

To inspect what's *at* that address:

```
(gdb) x/x $ebx
# 0x804a000:    0x00000005
```

`x/x $ebx` = "examine memory at the address stored in ebx, print as hex." You can see the value `5` sitting there at that address.

### Correct approach — use square brackets:

```nasm
mov ebx, [num]      ; dereference: go to the address of num, load what's there
```

Square brackets = **dereference** = "don't give me the address, give me what's stored at that address."

```nasm
section .data
    num    dd    5

section .text
    global _start

_start:
    mov eax, 1
    mov ebx, [num]      ; load the VALUE at num's address into ebx
    int 0x80
```

```bash
nasm -f elf -o data.o data.s
ld -m elf_i386 -o data data.o
./data
echo $?
# 5
```

### GDB confirmation

```bash
gdb data
(gdb) layout asm
(gdb) break _start
(gdb) run
```

In the disassembly view you'll notice the instruction now reads differently — no `$` prefix on the address, indicating a memory dereference rather than an immediate value.

```
(gdb) stepi
(gdb) stepi
(gdb) info registers ebx
# ebx = 0x5    5
```

`ebx` is now `5`. Correct.

---

## Full Working Program

```nasm
section .data
    num    dd    5          ; 32-bit variable, value 5

section .text
    global _start

_start:
    mov eax, 1              ; syscall: exit
    mov ebx, [num]          ; exit code = value stored at num = 5
    int 0x80
```

---

## Address vs Value — Mental Model

```
num          →    0x804a000        (the address — where num lives in memory)
[num]        →    5                (the value — what's stored at that address)
```

Same as pointer dereferencing in C:
```c
int num = 5;
int *ptr = &num;   // ptr  = address (like mov ebx, num)
int val = *ptr;    // val  = 5      (like mov ebx, [num])
```

---

## GDB Commands Introduced This Lecture

| Command | What it does |
|---|---|
| `x/x $ebx` | Examine memory at the address in `ebx`, display as hex |
| `x/x 0x804a000` | Examine memory at a literal address |

General form: `x/<count><format> <address>`

| Format | Meaning |
|---|---|
| `x` | Hex |
| `d` | Decimal |
| `c` | Char |
| `s` | String |

Example: `x/4x $ebx` — show 4 hex values starting at the address in `ebx`.

---

## Key Takeaways

- Variable declaration = `name  size_directive  value`
- Size directives: `db`(1) `dw`(2) `dd`(4) `dq`(8)
- No types — just bytes. Size is all that's specified.
- `num` = address. `[num]` = value at that address. **Always use brackets to load a value.**
- `x/x $reg` in GDB lets you peek at what's actually sitting in memory at any address.

---

## What's Next
Next lectures: special data types — strings, characters, arrays. More nuance around sizes and how data interacts with registers.
