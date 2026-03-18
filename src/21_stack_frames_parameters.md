# Lecture 21 — Stack Frames & Passing Parameters

> The full stack frame setup: why registers alone aren't enough for parameters, how EBP creates a stable reference point, and the exact memory layout when you call a procedure with arguments.

---

## Why Not Just Use Registers?

Registers are limited and shared. If a function uses `eax`/`ebx` for its own work, passing arguments via registers means the caller's data gets overwritten. The stack gives you unlimited, isolated parameter storage.

---

## The Stack at Call Time

Starting state — before the call:

```nasm
push 4              ; arg2
push 1              ; arg1
call add_two        ; pushes return address, jumps to add_two
```

Stack layout after `call` (top = low address, ESP points here):

```
Address offset    Content
ESP      →  [ return address ]
ESP + 4  →  [ 1              ]   ← arg1 (last pushed before call)
ESP + 8  →  [ 4              ]   ← arg2 (first pushed before call)
```

The problem: ESP points at the return address. If you try to access arguments relative to ESP, that offset changes every time you push/pop inside the function. Unstable.

---

## The Stack Frame: EBP as a Fixed Base

The solution — establish a **base pointer** that doesn't move during the function:

```nasm
add_two:
    push ebp            ; save caller's base pointer
    mov ebp, esp        ; EBP now points to current stack top

    ; stack is now stable — EBP won't change for the rest of this function
```

Stack after `push ebp` + `mov ebp, esp`:

```
Address         Content
ESP/EBP  →  [ saved EBP      ]   (EBP + 0)
EBP + 4  →  [ return address ]   (EBP + 4)
EBP + 8  →  [ 1              ]   (EBP + 8)  ← arg1
EBP + 12 →  [ 4              ]   (EBP + 12) ← arg2
```

Now use EBP as a stable anchor. Arguments are always at `[ebp + 8]`, `[ebp + 12]`, etc. regardless of what ESP does inside the function.

**Why +4 increments?** Each 32-bit value takes 4 bytes. Stack slots are 4 bytes wide.

---

## Accessing Parameters via EBP

```nasm
mov eax, [ebp + 8]      ; eax = arg1 = 1
mov ebx, [ebp + 12]     ; ebx = arg2 = 4
add eax, ebx            ; eax = 5
```

EBP offsets:

| `[EBP + N]` | Contains |
|---|---|
| `[EBP + 0]` | Saved EBP (from `push ebp`) |
| `[EBP + 4]` | Return address (from `call`) |
| `[EBP + 8]` | First argument |
| `[EBP + 12]` | Second argument |
| `[EBP + 16]` | Third argument |
| ... | ... |

---

## Teardown Before `ret`

Before returning, restore ESP to where it was (pointing at the return address, not the saved EBP):

```nasm
pop ebp             ; restore caller's EBP, ESP now points at return address
ret                 ; pop return address, jump there
```

If you don't `pop ebp`, `ret` pops the saved EBP value as the return address and jumps to garbage.

---

## Full Program

```nasm
section .text
    global main

add_two:
    push ebp                ; save caller's frame pointer
    mov ebp, esp            ; establish our frame

    mov eax, [ebp + 8]      ; load arg1 (= 1)
    mov ebx, [ebp + 12]     ; load arg2 (= 4)
    add eax, ebx            ; eax = 5

    pop ebp                 ; restore frame pointer
    ret                     ; return to caller, result in eax

main:
    push 4                  ; arg2 (pushed first = deeper on stack)
    push 1                  ; arg1 (pushed second = closer to top)
    call add_two            ; eax = 5 on return

    mov ebx, eax            ; exit code = result
    mov eax, 1
    int 0x80
```

```bash
nasm -f elf -o frame.o frame.s
gcc -no-pie -m32 -o frame frame.o
./frame
echo $?
# 5
```

---

## GDB Session

```bash
gdb frame
(gdb) layout asm
(gdb) break main
(gdb) run
```

After `push 4`, `push 1`, `call add_two`, `push ebp`, `mov ebp, esp`:

```
(gdb) info registers esp
# esp = 0xffffd...          ← note this address

(gdb) x/4x 0xffffd...      ← examine 4 slots from ESP
# 0xffffd000: 0xffffd010    ← saved EBP  (EBP + 0)
# 0xffffd004: 0x08049186    ← return address (EBP + 4)
# 0xffffd008: 0x00000001    ← arg1 = 1   (EBP + 8)
# 0xffffd00c: 0x00000004    ← arg2 = 4   (EBP + 12)
```

After `mov eax, [ebp + 8]`:
```
(gdb) info registers eax
# eax = 0x1    1            ← arg1 loaded correctly
```

After `mov ebx, [ebp + 12]`:
```
(gdb) info registers ebx
# ebx = 0x4    4            ← arg2 loaded correctly
```

After `add eax, ebx`:
```
(gdb) info registers eax
# eax = 0x5    5            ← 1 + 4 = 5
```

After `pop ebp`, `ret`:
```
; execution returns to main, eax still = 5
```

---

## Stack Diagram — Full Lifecycle

```
BEFORE call:            INSIDE function:         AFTER ret:

                        EBP/ESP →[ saved EBP  ]
                                 [ ret addr   ]
ESP →[ ret addr  ]               [ arg1 = 1   ]
     [ arg1 = 1  ]      EBP+8 →[ arg1 = 1   ]    (stack cleaned up)
     [ arg2 = 4  ]      EBP+12→[ arg2 = 4   ]    ESP back to before push 4
```

---

## Why EBP Matters for Nested Calls

If `add_two` called another function, that function would do its own `push ebp` / `mov ebp, esp`. The saved EBP chain creates a **linked list of stack frames** — each frame knows where the previous one started. Debuggers use this chain to show you a call stack.

---

## Key Takeaways

- Push args before `call` — they sit on the stack when the function starts
- `push ebp` / `mov ebp, esp` = standard function prologue — creates a stable frame
- `[ebp + 8]` = arg1, `[ebp + 12]` = arg2, `[ebp + 4]` = return address
- `pop ebp` / `ret` = standard function epilogue — tears down the frame and returns
- ESP moves around inside functions; EBP stays fixed — always use EBP to reference args and locals
- Return value goes in `eax` (cdecl convention)

---

## What's Next
More advanced procedure topics — local variables on the stack, preserving registers across calls.
