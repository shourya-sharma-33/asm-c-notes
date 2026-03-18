# Lecture 18 — Calling C Functions from NASM

> Covers linking NASM object files with GCC to call standard C functions (`printf`, `exit`), how arguments are passed via the stack (push order matters), and the compilation flags needed to make it all work.

---

## Why and When

Instead of using raw `int 0x80` syscalls, you can call C standard library functions directly. This gives you access to `printf`, `malloc`, `exit`, file I/O, etc. — without implementing them yourself.

The tradeoff: you're now depending on libc and GCC for linking instead of raw `ld`.

---

## Prerequisites Setup

### 1. Install 32-bit GCC support (Ubuntu/WSL2)

```bash
sudo apt install gcc-multilib
```

This provides the 32-bit version of libc and GCC support needed for `-m32`.

### 2. Verify NASM is installed

```bash
nasm --version
```

### 3. File structure

```
project/
├── first.s       ← your NASM source
```

---

## Key Differences from Previous Programs

| | Previous (syscall) | This lecture (C functions) |
|---|---|---|
| Entry point | `global _start` / `_start:` | `global main` / `main:` |
| Exit method | `mov eax,1` + `int 0x80` | `call exit` |
| Linker | `ld` | `gcc` |
| External refs | none | `extern printf`, `extern exit` |

GCC requires a `main` function as the entry point — it wraps your code in its own startup/teardown (`crt0`). If you use `_start` with GCC, the C runtime won't initialize properly.

---

## Declaring External C Functions

Tell NASM that these names are defined externally (in libc, linked by GCC):

```nasm
extern printf
extern exit
```

This is like a forward declaration — NASM won't look for the function body, it trusts the linker to resolve it.

---

## The Stack and Argument Passing

C functions receive arguments via the stack, in **reverse order** (right-to-left). This is the **cdecl calling convention** — standard for 32-bit C.

`printf(format, arg1, arg2, ...)` → push in reverse: `arg2`, `arg1`, `format` last.

### `push` instruction

```nasm
push value      ; place value on top of stack, decrement ESP by 4
```

Arguments are popped by the called function in LIFO order — last pushed = first used.

---

## Complete Program: printf + exit

```nasm
extern printf
extern exit

section .data
    message    db    "Hello World", 0           ; null-terminated string
    message2   db    "This is a test", 0
    fmt        db    "%s %s", 10, 0             ; format: two strings + newline + null
    ; 10 = 0x0A = newline character (\n)

section .text
    global main

main:
    ; --- call printf(fmt, message, message2) ---
    ; push arguments RIGHT TO LEFT
    push message2           ; last arg pushed first
    push message            ; second arg
    push fmt                ; format string pushed last (first arg = first off stack)

    call printf

    ; --- call exit(10) ---
    push 10                 ; exit code
    call exit
```

### Argument order explanation

```
printf(fmt, message, message2)
         ↑      ↑        ↑
       arg0   arg1     arg2

Push order (reverse):
  push message2   ← arg2, pushed first (deepest on stack)
  push message    ← arg1
  push fmt        ← arg0, pushed last (top of stack = first popped)

Stack at time of call (top → bottom):
  fmt
  message
  message2
```

When `printf` runs, it reads `fmt` first (top of stack), then `message`, then `message2`. Matches the format string's `%s %s` in order.

---

## Compiling and Linking with GCC

Two steps: assemble with NASM, then link with GCC.

### Step 1: Assemble

```bash
nasm -f elf -o first.o first.s
```

Same as before — produces ELF object file.

### Step 2: Link with GCC

```bash
gcc -no-pie -m32 -o first first.o
```

| Flag | Purpose |
|---|---|
| `-no-pie` | Disable Position Independent Executable — PIE conflicts with our fixed-address NASM code |
| `-m32` | Tell GCC we're targeting 32-bit x86 |
| `-o first` | Output executable name |
| `first.o` | Input object file |

**Why not `ld` anymore?** GCC handles linking libc automatically. Using raw `ld` would require manually specifying every libc path and startup file — much more complex.

### Step 3: Run

```bash
./first
# Hello World This is a test

echo $?
# 10
```

---

## GDB Session

```bash
gdb first
(gdb) layout asm
(gdb) break main
(gdb) run
(gdb) stepi             ; push message2
(gdb) stepi             ; push message
(gdb) stepi             ; push fmt
(gdb) stepi             ; call printf → jumps into libc, returns after print
(gdb) stepi             ; push 10
(gdb) stepi             ; call exit → program terminates
```

You'll see the print output appear in the terminal when `call printf` executes.

---

## Single String Example (simpler)

```nasm
extern printf
extern exit

section .data
    message    db    "Hello World", 0
    fmt        db    "%s", 10, 0

section .text
    global main

main:
    push message
    push fmt
    call printf

    push 1
    call exit
```

```bash
nasm -f elf -o first.o first.s
gcc -no-pie -m32 -o first first.o
./first
# Hello World
echo $?
# 1
```

---

## Key Takeaways

- Use `global main` / `main:` instead of `_start` when linking with GCC
- Declare external C functions with `extern funcname`
- Arguments go on the stack via `push` in **reverse order** (right to left)
- `call funcname` transfers control to the C function
- Compile: `nasm -f elf` then `gcc -no-pie -m32`
- `-no-pie` is required — without it, GCC generates position-independent code that breaks fixed NASM addresses
- `-m32` is required on 64-bit machines to stay in 32-bit mode
- `10` in a string is the newline character (`\n`)

---

## What's Next
Calling your own custom C functions from NASM — writing C code and linking it with your assembly.
