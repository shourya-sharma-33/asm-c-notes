# Lecture 03 — First NASM Program

> **First code lecture.** Covers NASM syntax basics, program structure, the `mov` instruction, system interrupts to exit, and a full GDB debug session walking through register state step by step.

---

## NASM vs AT&T Syntax

Two main x86 assembly syntaxes exist:

| Syntax | Used by | Notes |
|---|---|---|
| **NASM (Intel-based)** | This course | Cleaner, more readable |
| **AT&T** | Default on Linux (`gcc -S`, `objdump`) | More complex notation |

Core concepts are identical — syntax differences are superficial (operand order flips, different sigils, etc.). Learn NASM first, AT&T is easy to pick up after.

---

## File Setup

```bash
nano first.s
```

File extensions for assembly: `.s`, `.asm`. The course uses `.s`.

---

## Program Structure

Every NASM program has sections. The two fundamental ones:

```nasm
section .data       ; initialized variables (stored in memory at load time)
section .text       ; actual executable code
```

### Entry Point

The linker needs to know where execution starts. Declare it like this:

```nasm
global _start       ; export _start so the OS/linker can find it

_start:             ; label — everything below executes when _start is called
```

- `_start` is a **label** — a named address in the code
- `global _start` makes it visible outside the object file (to the linker)
- Without `global _start`, the linker won't know where your program begins and will error

---

## The `mov` Instruction

```nasm
mov destination, source
```

Moves data from `source` into `destination`. Source is the **right** operand, destination is the **left** operand.

```nasm
mov eax, 1      ; put the value 1 into register eax
mov ebx, 1      ; put the value 1 into register ebx
```

> **Direction:** right → left. Value on the right goes into the register on the left.

---

## Exiting a Program (System Interrupt)

To exit cleanly you need to tell the OS to run the `exit` syscall. This is done via a **software interrupt**:

```nasm
int 0x80        ; trigger interrupt 80h — hands control to the Linux kernel
```

When `int 0x80` fires, the kernel looks at `eax` to decide what syscall to run:

| `EAX` value | Syscall |
|---|---|
| `1` | `exit` |
| `4` | `write` |
| ... | ... |

For `exit`, `ebx` holds the **exit status code** returned to the shell.

---

## Complete First Program

```nasm
section .data

section .text
    global _start

_start:
    mov eax, 1      ; syscall number: 1 = exit
    mov ebx, 1      ; exit status code = 1
    int 0x80        ; call kernel
```

### What this does:
1. Moves `1` into `eax` → selects the `exit` syscall
2. Moves `1` into `ebx` → sets exit code to `1`
3. `int 0x80` → kernel takes over, exits the program with code 1

---

## Assembling & Linking

Two-step process: **assemble** (source → object file), then **link** (object file → executable).

### Step 1: Assemble
```bash
nasm -f elf -o first.o first.s
```

| Flag | Meaning |
|---|---|
| `-f elf` | Output format: ELF (Linux binary format) |
| `-o first.o` | Output object file name |
| `first.s` | Input source file |

### Step 2: Link
```bash
ld -m elf_i386 -o first first.o
```

| Flag | Meaning |
|---|---|
| `-m elf_i386` | Target 32-bit x86 — needed even on 64-bit machines to stay in 32-bit mode |
| `-o first` | Output executable name |
| `first.o` | Input object file |

### Run it
```bash
./first
```

### Check exit code
```bash
echo $?
# Output: 1
```

`$?` in bash always holds the exit code of the last command. Here it's `1` because we set `ebx = 1`.

---

## Files produced

| File | What it is |
|---|---|
| `first.s` | Your source code |
| `first.o` | Object file (intermediate, not executable) |
| `first` | Final executable |

---

## GDB Debug Session

```bash
gdb first
```

### Enter assembly layout mode
```
(gdb) layout asm
```
Shows the disassembly view — you can see each instruction and which one is currently executing.

### Set a breakpoint at `_start`
```
(gdb) break _start
```
Program will pause as soon as it hits the `_start` label.

### Run the program
```
(gdb) run
```
Execution starts and immediately halts at `_start`.

### Step one instruction forward
```
(gdb) stepi
```
Executes exactly one instruction. `stepi` = step instruction (as opposed to `step` which steps by source line).

After the first `stepi`:
- Executed: `mov eax, 1`

### Inspect register state
```
(gdb) info registers eax
# eax = 0x1    1
```
Confirms `eax` is now `1`.

```
(gdb) info registers ebx
# ebx = 0x0    0
```
`ebx` is still `0` — we haven't executed that instruction yet.

### Step again
```
(gdb) stepi
```
Now executed: `mov ebx, 1`

```
(gdb) info registers ebx
# ebx = 0x1    1
```
`ebx` is now `1`. Confirmed.

---

## GDB Quick Reference (so far)

| Command | What it does |
|---|---|
| `layout asm` | Show assembly disassembly view |
| `break _start` | Set breakpoint at label |
| `run` | Start execution |
| `stepi` | Execute one instruction |
| `info registers eax` | Print value of a specific register |
| `info registers` | Print all registers |

---

## Key Takeaways

- `mov dst, src` — data flows right to left
- `int 0x80` + `eax=1` + `ebx=<code>` is the standard exit pattern for 32-bit Linux
- Assemble with `nasm -f elf`, link with `ld -m elf_i386`
- GDB's `stepi` + `info registers` lets you watch register values change instruction by instruction

---

## What's Next
Next lecture: declaring variables in the `.data` section.
