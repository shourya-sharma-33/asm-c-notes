# Lecture 19 — Calling Custom C Functions from NASM

> Extends lecture 18: instead of calling libc functions, you write your own C function and call it from assembly. Covers `extern` on the C side, return values landing in `eax`, and linking multiple source files with GCC.

---

## The C File (`test.c`)

```c
#include <stdio.h>

extern int test(int a, int b);      // export declaration (accessible from outside)

int test(int a, int b) {
    printf("here\n");               // print to confirm we're inside the function
    return a + b;                   // return value goes back to the caller
}
```

Key points:
- `extern int test(...)` makes `test` visible to the linker — your NASM file can call it
- The return value of any C function comes back in `eax` (cdecl convention)
- No `main` needed in the C file — main lives in the NASM file

---

## The NASM File (`first.s`)

```nasm
extern test             ; our custom C function
extern exit             ; from libc

section .text
    global main

main:
    ; call test(2, 1) — push args right to left
    push 1              ; arg b = 1 (pushed first = deeper on stack)
    push 2              ; arg a = 2 (pushed second = closer to top)
    call test           ; return value lands in eax automatically

    ; use return value as exit code
    push eax            ; push eax (which holds a+b = 3) as exit's argument
    call exit
```

### Return value convention

After any `call` returns, `eax` holds the return value. You don't do anything special — it's just there.

```nasm
call test           ; eax = return value of test() = a + b = 3
push eax            ; pass that value as the exit code
call exit           ; exit(3)
```

---

## Compiling — Linking Multiple Files

Now you have two source files: `first.s` (NASM) and `test.c` (C). Both go to GCC:

### Step 1: Assemble the NASM file

```bash
nasm -f elf -o first.o first.s
```

### Step 2: Compile + link everything with GCC

```bash
gcc -no-pie -m32 -o first first.o test.c
```

| Argument | Purpose |
|---|---|
| `-no-pie` | Disable PIE — required for fixed-address NASM code |
| `-m32` | 32-bit target |
| `-o first` | Output binary name |
| `first.o` | Assembled NASM object file |
| `test.c` | C source file — GCC compiles and links it in |

GCC compiles `test.c` and links it together with `first.o` into one binary. The linker resolves the `extern test` reference in your NASM code to the `test` function in `test.c`.

### Step 3: Run

```bash
./first
# here              ← printed by printf inside test()

echo $?
# 3                 ← exit code = return value of test(2, 1) = 2 + 1
```

---

## What's Happening at Link Time

```
first.s  →  nasm  →  first.o  ─┐
test.c   →  gcc   →  test.o   ─┼─→  gcc linker  →  first (executable)
libc     ──────────────────────┘
```

The linker sees `extern test` in `first.o`, finds `test` defined in `test.o`, and connects them. Same process for `extern exit` → libc.

---

## Argument Order Reminder

```
test(a=2, b=1)

Push order (reverse):
  push 1      ← b (arg1, last arg, pushed first)
  push 2      ← a (arg0, first arg, pushed last)

Stack at call time (top → bottom):
  2  ← a (first off stack = first argument)
  1  ← b
```

---

## Return Value Flow

```
C function returns a + b = 3
          ↓
    stored in eax automatically
          ↓
push eax          ; push 3 onto stack
call exit         ; exit(3)
          ↓
echo $?   →  3
```

Any C function you call will put its return value in `eax`. Check `eax` immediately after `call` returns — the next instruction could overwrite it.

---

## Key Takeaways

- Mark C functions with `extern` in the C file to export them to the linker
- Declare them with `extern funcname` in your NASM file
- Push arguments right-to-left (same cdecl convention as always)
- Return value is always in `eax` after `call` returns
- Link with `gcc -no-pie -m32 -o output first.o yourfile.c` — pass both the object file and C source
- GCC handles compiling the C file and linking everything together

---

## What's Next
Writing your own NASM functions — `call`, `ret`, stack frames, and `ebp`/`esp`.
