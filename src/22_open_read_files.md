# Lecture 22 — Opening & Reading Files via Syscalls

> Covers `sys_open` (syscall 5) and `sys_read` (syscall 3) — how to find what each syscall needs, what a file descriptor is, how to set up a read buffer in `.bss`, and how to inspect the result in GDB.

---

## How to Look Up Any Syscall

Two resources, used together:

**1. x86 Linux syscall table** — tells you the `eax` number and which registers hold each argument.
Search: "Linux x86 32-bit syscall table" or visit `https://syscalls.kernelgrok.com`

**2. `man 2 <syscall>`** — tells you what each argument actually means.
```bash
man 2 open
man 2 read
```

**Workflow:**
1. Find syscall number → put in `eax`
2. Read man page → map arguments to `ebx`, `ecx`, `edx` in order
3. Look up constant values (flags etc.) in the header files mentioned in the man page

---

## `sys_open` — Open a File

### Syscall table entry
| Register | Value |
|---|---|
| `eax` | `5` (sys_open) |
| `ebx` | `char *pathname` — path to file |
| `ecx` | `int flags` — what to do with the file |
| `edx` | `mode_t mode` — permissions (only needed when creating a file) |

### Flags (from `fcntl.h`)

| C constant | Value | Meaning |
|---|---|---|
| `O_RDONLY` | `0` | Open read-only |
| `O_WRONLY` | `1` | Open write-only |
| `O_RDWR` | `2` | Open read/write |
| `O_CREAT` | `64` | Create file if it doesn't exist |

Find these by checking `man 2 open` → it references `fcntl.h` → look up the constants there.

### Return value

After `int 0x80`, `eax` holds the **file descriptor** — an integer the OS assigns to identify this open file. Use it in subsequent syscalls (`read`, `write`, `close`). Typical values: 0=stdin, 1=stdout, 2=stderr, 3+=your files.

### Code

```nasm
section .data
    filename    db    "/path/to/your/file", 0   ; null-terminated path

section .text
    global main

main:
    mov eax, 5              ; syscall: sys_open
    mov ebx, filename       ; arg1: pointer to path string
    mov ecx, 0              ; arg2: O_RDONLY = 0
    int 0x80                ; eax = file descriptor (e.g. 3)
```

---

## `sys_read` — Read from a File

### Syscall table entry
| Register | Value |
|---|---|
| `eax` | `3` (sys_read) |
| `ebx` | `int fd` — file descriptor from `sys_open` |
| `ecx` | `char *buf` — address of buffer to read into |
| `edx` | `size_t count` — max bytes to read |

### Return value

`eax` = number of bytes actually read.
- Returns `0` when end of file is reached (use this to loop through an entire file)
- Returns negative on error

### Buffer setup in `.bss`

```nasm
section .bss
    buffer    resb    1024   ; reserve 1024 bytes — adjust to expected file size
```

Reserve more than the file size to be safe. If you reserve less, you'll truncate the read.

---

## Full Program: Open + Read a File

```nasm
section .data
    filename    db    "/path/to/your/file", 0

section .bss
    buffer      resb    1024

section .text
    global main

main:
    ; --- open the file ---
    mov eax, 5              ; sys_open
    mov ebx, filename       ; path
    mov ecx, 0              ; O_RDONLY
    int 0x80                ; eax = file descriptor

    ; --- read from the file ---
    mov ebx, eax            ; move fd from eax to ebx (sys_read needs fd in ebx)
    mov eax, 3              ; sys_read
    mov ecx, buffer         ; buffer address
    mov edx, 1024           ; max bytes to read
    int 0x80                ; eax = bytes read, buffer filled

    ; --- exit ---
    mov eax, 1
    mov ebx, 0
    int 0x80
```

**Critical step:** After `sys_open`, `eax` holds the fd. Before calling `sys_read`, move that fd to `ebx` first, then overwrite `eax` with `3`. If you set `eax = 3` first, you lose the fd.

---

## GDB Session

```bash
nasm -f elf -o file.o file.s
gcc -no-pie -m32 -o file file.o
gdb file
```

```
(gdb) layout asm
(gdb) break main
(gdb) run
```

**After sys_open fires:**
```
(gdb) stepi × 4             ; load regs, int 0x80
(gdb) info registers eax
# eax = 0x3    3            ← file descriptor assigned by OS
```

**Inspect the path string:**
```
(gdb) x/s $ebx              ; print string at address in ebx
# "/path/to/your/file"      ← confirms correct address loaded
```

**After sys_read fires:**
```
(gdb) stepi × 5
(gdb) info registers eax
# eax = 0x...               ← number of bytes read

(gdb) x/s 0x804c04c         ; or wherever buffer is (use ecx value from before int 0x80)
# "contents of your file..."
```

---

## Reading Until End of File (Pattern)

```nasm
read_loop:
    mov eax, 3
    mov ebx, fd_register    ; file descriptor
    mov ecx, buffer
    mov edx, 1024
    int 0x80                ; eax = bytes read

    cmp eax, 0              ; 0 = EOF
    je done                 ; exit loop at EOF

    ; process buffer here

    jmp read_loop

done:
    ; close file, exit, etc.
```

---

## Syscall Argument Map (Reference)

| Syscall | `eax` | `ebx` | `ecx` | `edx` |
|---|---|---|---|---|
| `sys_exit` | 1 | exit code | — | — |
| `sys_read` | 3 | fd | buffer ptr | byte count |
| `sys_write` | 4 | fd | buffer ptr | byte count |
| `sys_open` | 5 | path ptr | flags | mode |
| `sys_close` | 6 | fd | — | — |

---

## Key Takeaways

- Look up syscall number in the table → put in `eax`
- Read `man 2 <syscall>` to find what each argument means → map to `ebx`, `ecx`, `edx`
- Flag constants (like `O_RDONLY = 0`) are in header files referenced by the man page
- `sys_open` returns a **file descriptor** in `eax` — save it before overwriting `eax`
- Create read buffers in `.bss` with `resb N`
- `sys_read` returns bytes read; returns `0` at EOF — use for loop termination
- Data ends up in the buffer at the address you passed in `ecx`

---

## What's Next
Writing to files — `sys_write` and `sys_close`.
