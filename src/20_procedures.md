# Lecture 20 — Procedures (Functions) in x86

> Covers defining your own procedures in NASM using labels, `call`, and `ret`. Explains exactly how the CPU knows where to return — the return address on the stack — and why this matters for security.

---

## What is a Procedure?

A procedure = a function in x86. It's a labeled block of code you `call` and that `ret`urns when done. Same concept as a function in any language, just manual.

---

## Defining a Procedure

A procedure is just a label followed by instructions, ending with `ret`:

```nasm
add_two:                    ; procedure label
    add eax, ebx            ; body — operates on registers
    ret                     ; return to caller
```

`ret` pops the return address off the stack and jumps to it. More on how this works below.

---

## Calling a Procedure

```nasm
call add_two        ; push return address onto stack, jump to add_two
```

`call` does two things atomically:
1. Pushes the address of the **next instruction** onto the stack
2. Jumps to the label

---

## Full Example

```nasm
section .text
    global main

add_two:
    add eax, ebx            ; eax = eax + ebx
    ret                     ; return to caller

main:
    mov eax, 3
    mov ebx, 2
    call add_two            ; eax = 5 after this returns

    mov ebx, eax            ; preserve result in ebx for exit code
    mov eax, 1
    int 0x80
```

```bash
nasm -f elf -o proc.o proc.s
gcc -no-pie -m32 -o proc proc.o
./proc
echo $?
# 5                         ← 3 + 2
```

---

## How `ret` Knows Where to Go — The Return Address

This is the key mechanism. When you execute `call add_two`:

```
Stack before call:
  [ ... existing stack ... ]

call add_two:
  1. Push address of next instruction (e.g. 0x8049182) onto stack
  2. Jump to add_two

Stack during add_two:
  [ 0x8049182 ]   ← return address at ESP
  [ ... ]
```

When `ret` executes:
```
  1. Pop value at ESP (0x8049182)
  2. Jump to that address
  3. ESP incremented (stack shrinks)
```

Execution resumes at `0x8049182` — the instruction right after the `call`.

### GDB proof

```bash
gdb proc
(gdb) layout asm
(gdb) break main
(gdb) run
(gdb) stepi × 3             ; mov eax, mov ebx, then land on call
(gdb) info registers esp
# esp = 0xffffd...          ← current stack pointer

(gdb) x/x 0xffffd...        ; examine memory at esp (use actual esp value)
# 0xffffd...: 0x08049182    ← return address pushed by call

(gdb) stepi                 ; execute call — jumps into add_two
(gdb) stepi                 ; add eax, ebx
(gdb) stepi                 ; ret — pops 0x08049182, jumps there
; back at instruction after call
```

---

## The Stack During a `call`/`ret` Cycle

```
Before call:       During procedure:      After ret:
                   ┌──────────────┐
ESP →              │ return addr  │ ← ESP
                   └──────────────┘
                   │ caller stack │       ESP → (back to before call)
```

`call` grows the stack (pushes). `ret` shrinks it (pops). ESP always points to the top.

---

## Security Note: Return Address Hijacking

The return address sitting on the stack is the root of one of the most classic exploits: **stack buffer overflow / ret2libc**.

If an attacker can write past a buffer on the stack, they can overwrite the return address with an address of their choosing. When `ret` fires, instead of returning to your code, it jumps to theirs — typically into libc (`system("/bin/sh")` etc.).

This is why modern systems have:
- **Stack canaries** — detect overwrites before `ret`
- **ASLR** — randomize addresses so attacker can't predict where to jump
- **NX bit** — mark stack non-executable

You'll encounter all of these if you go into exploit development or systems security.

---

## Key Takeaways

- A procedure = label + instructions + `ret`
- `call label` = push return address + jump
- `ret` = pop return address from stack + jump to it
- The return address is the instruction immediately after the `call`
- ESP points to the return address while inside the procedure — don't corrupt it
- This is the exact mechanism that stack overflow exploits target

---

## What's Next
Passing parameters to procedures via the stack — the proper stack frame setup with `ebp`/`esp`.
