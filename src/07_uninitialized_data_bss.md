# Lecture 07 — Uninitialized Data (`.bss` Section)

> Covers the `.bss` section for reserving memory without an initial value, how to write data into reserved slots via registers, pointer arithmetic to reach adjacent bytes, and the `dup` directive for bulk initialization in `.data`.

---

## Two Ways to Declare Variables

| Section | Keyword | Meaning |
|---|---|---|
| `.data` | `db`, `dw`, `dd`, `dq` | **Define** — allocate space AND set an initial value |
| `.bss` | `resb`, `resw`, `resd`, `resq` | **Reserve** — allocate space only, value is zero by default |

Use `.bss` when you need a buffer or working space that you'll fill in at runtime.

---

## `.bss` Syntax

```nasm
section .bss
    num    resb    3       ; reserve 3 bytes for 'num', all initialized to 0
```

### Reserve directives

| Directive | Meaning | Size per unit |
|---|---|---|
| `resb N` | Reserve N bytes | 1 byte each |
| `resw N` | Reserve N words | 2 bytes each |
| `resd N` | Reserve N doublewords | 4 bytes each |
| `resq N` | Reserve N quadwords | 8 bytes each |

`resb 3` = 3 separate byte slots, each zeroed, assigned to label `num`.

---

## Writing into Reserved Memory

You **cannot** move an immediate value directly into a memory address:

```nasm
mov [num], 1        ; INVALID — x86 doesn't know the size of the destination
```

x86 needs both sides of a `mov` to have a known, matching size. The fix: stage the value in a sized register first, then move from register to memory.

```nasm
mov bl, 1           ; bl = 1 byte — size is now known
mov [num], bl       ; move 1 byte from bl into num's first slot
```

---

## Accessing Adjacent Bytes — Pointer Arithmetic

`num` is the address of the first reserved byte. To reach subsequent bytes, add an offset:

```nasm
mov [num],     bl   ; byte 0 — first slot
mov [num + 1], bl   ; byte 1 — second slot
mov [num + 2], bl   ; byte 2 — third slot
```

The offset is in **bytes**. If you were using `resw` (2-byte words), offsets would be `+2`, `+4`, etc.

---

## Full Example

```nasm
section .bss
    num    resb    3       ; reserve 3 bytes

section .text
    global _start

_start:
    mov bl, 1
    mov [num],     bl      ; store 1 at byte 0
    mov [num + 1], bl      ; store 1 at byte 1
    mov [num + 2], bl      ; store 1 at byte 2

    mov eax, 1
    int 0x80
```

---

## GDB Session

```bash
nasm -f elf -o unit.o unit.s
ld -m elf_i386 -o unit unit.o
gdb unit
```

```
(gdb) layout asm
(gdb) break _start
(gdb) run
```

**Before any stores — check initial state:**
```
(gdb) x/x 0x804a000
# 0x804a000:    0x00000000      ← all three bytes zeroed (uninitialized default)
```

**After `mov [num], bl`:**
```
(gdb) stepi
(gdb) stepi
(gdb) x/x 0x804a000
# 0x804a000:    0x00000001      ← first byte = 1, rest still 0
```

**After `mov [num+1], bl`:**
```
(gdb) stepi
(gdb) x/x 0x804a000
# 0x804a000:    0x00000101      ← first two bytes = 1
```

**After `mov [num+2], bl`:**
```
(gdb) stepi
(gdb) x/x 0x804a000
# 0x804a000:    0x00010101      ← all three bytes = 1
```

---

## Bulk Initialization with `dup` (in `.data`)

If you want to initialize multiple slots to the same value, use `dup` in `.data`:

```nasm
section .data
    num2    db    3 dup(2)     ; write the value 2, three times
                               ; equivalent to: db 2, 2, 2
```

### GDB confirmation:
```
(gdb) x/x 0x804a000
# 0x804a000:    0x00020202      ← three bytes, each = 2
```

**`dup` vs `.bss`:**

| | `.bss` + `resb` | `.data` + `dup` |
|---|---|---|
| Default value | 0 (always) | Whatever you specify |
| Use case | Runtime buffers, scratch space | Pre-filled lookup tables, default arrays |

---

## Key Takeaways

- `.bss` reserves memory without a value — all bytes start as `0`
- You can't `mov` an immediate directly to memory — stage it in a register first, then store
- Adjacent bytes within a reservation are reached via `[label + N]` offset
- `dup(val)` in `.data` initializes N copies of a value inline
- `.bss` goes in its own section, separate from `.data`

---

## What's Next
Next lectures: actual x86 instructions — arithmetic, logic, jumps. Done with data storage fundamentals.
