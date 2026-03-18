# Supplement 03A — Syscalls, `int 0x80`, and Bare Metal

---

## What is `int 0x80`?

`int` = software interrupt. It tells the CPU: **pause execution, jump to the handler registered for interrupt number N.**

`0x80` = 128 in decimal. Linux registers its kernel entry point at interrupt 128. So `int 0x80` means: **jump into the Linux kernel and ask it to do something.**

The CPU looks up interrupt 128 in the **Interrupt Descriptor Table (IDT)** — a table the OS sets up at boot that maps interrupt numbers to handler addresses. Linux puts its syscall dispatcher at 0x80.

---

## The Syscall Table

When you fire `int 0x80`, the kernel looks at `eax` to decide *what* to do. Every OS service has a number. This is the **syscall table**.

### 32-bit Linux x86 Calling Convention

```
eax = syscall number       (which service do you want?)
ebx = argument 1
ecx = argument 2
edx = argument 3
esi = argument 4
edi = argument 5
```

Put your args in the right registers, set `eax`, fire `int 0x80`. Kernel does the work. Return value comes back in `eax`.

### Common Syscalls (32-bit Linux)

| `eax` | Syscall | `ebx` | `ecx` | `edx` |
|---|---|---|---|---|
| 1 | `exit` | exit code | — | — |
| 2 | `fork` | — | — | — |
| 3 | `read` | file descriptor | buffer address | byte count |
| 4 | `write` | file descriptor | buffer address | byte count |
| 5 | `open` | filename address | flags | mode |
| 6 | `close` | file descriptor | — | — |

Full table: search **"Linux x86 32-bit syscall table"** or visit `https://syscalls.kernelgrok.com`

---

## Examples

### Exit with code 0 (success)
```nasm
mov eax, 1          ; syscall: exit
mov ebx, 0          ; exit code 0 = success
int 0x80
```

### Exit with code 1 (failure/error)
```nasm
mov eax, 1          ; syscall: exit
mov ebx, 1          ; exit code 1 = general error
int 0x80
```

### Write "Hello" to stdout
```nasm
section .data
    msg db "Hello", 0x0a    ; string + newline

section .text
    global _start

_start:
    mov eax, 4          ; syscall: write
    mov ebx, 1          ; fd 1 = stdout
    mov ecx, msg        ; address of string
    mov edx, 6          ; number of bytes to write
    int 0x80

    mov eax, 1          ; syscall: exit
    mov ebx, 0
    int 0x80
```

**File descriptors on Linux:**
| fd | Meaning |
|---|---|
| 0 | stdin |
| 1 | stdout |
| 2 | stderr |

### Read from stdin into a buffer
```nasm
section .bss
    buf resb 64         ; reserve 64 bytes

section .text
    global _start

_start:
    mov eax, 3          ; syscall: read
    mov ebx, 0          ; fd 0 = stdin
    mov ecx, buf        ; where to store the data
    mov edx, 64         ; max bytes to read
    int 0x80
    ; after this: eax = number of bytes actually read
```

---

## How to Look Up Any Syscall

1. Find the syscall number in the table
2. Check the Linux man page: `man 2 <syscallname>` — e.g. `man 2 write`
3. Map the C function signature to registers:
   - return value → `eax` (after the call)
   - 1st arg → `ebx`
   - 2nd arg → `ecx`
   - 3rd arg → `edx`

Example — `man 2 write` gives:
```c
ssize_t write(int fd, const void *buf, size_t count);
```
So: `ebx=fd`, `ecx=buf`, `edx=count`. That's it.

---

## On Bare Metal — No Syscalls Exist

On bare metal there is no OS, so:
- The IDT is **empty** (or whatever firmware set up)
- `int 0x80` has **no handler** — firing it causes a CPU exception and crashes
- There is no syscall table
- There is no stdout, no exit, no file descriptors

### How you do I/O on bare metal

**Option 1: Port-mapped I/O (`in`/`out` instructions)**

Hardware devices (serial port, keyboard controller, etc.) are accessible via **I/O ports** — a separate 16-bit address space the CPU exposes via `in`/`out`.

```nasm
; Write byte to serial port COM1 (port 0x3F8)
mov al, 'A'         ; byte to send
mov dx, 0x3F8       ; COM1 data port
out dx, al          ; send it
```

```nasm
; Read a byte from a port
mov dx, 0x60        ; keyboard data port
in al, dx           ; al = byte read from port
```

**Option 2: Memory-mapped I/O**

Some hardware (VGA, framebuffer) is mapped directly into the memory address space. You write to those addresses like regular memory.

```nasm
; Write directly to VGA text buffer (address 0xB8000 in real/protected mode)
mov eax, 0xB8000
mov byte [eax], 'A'     ; character
mov byte [eax+1], 0x07  ; attribute: white on black
```

**To "exit" on bare metal:**
```nasm
; Halt the CPU — stops execution, waits for interrupt
hlt

; Or infinite loop — spin forever
hang:
    jmp hang
```

There's no "exit code" — you're talking to hardware, not an OS.

---

## Summary: Linux vs Bare Metal

| | Linux (on top of OS) | Bare Metal |
|---|---|---|
| I/O | `int 0x80` + syscall number | `in`/`out` ports or memory-mapped I/O |
| Exit | `eax=1, int 0x80` | `hlt` or infinite loop |
| Syscall table | Yes — Linux provides it | Doesn't exist |
| IDT | Set up by Linux | You set it up yourself |
| stdout/stdin | fd 1, fd 0 via `write`/`read` | Doesn't exist — talk to UART/serial directly |
| Reference | `man 2 <syscall>` + syscall table | Hardware datasheet for your specific chip/board |

---

## Key Mental Model

> `int 0x80` is just a **knock on the kernel's door**. The kernel decides what to do based on `eax`. If there's no kernel, there's no one to answer the door.

On Linux: knock → kernel answers → does the work for you.
On bare metal: knock → silence → CPU exception → crash.

You replace the kernel's services with your own code when on bare metal.
