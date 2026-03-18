# Lecture 24 — Creating & Writing Files

> Covers creating a new file with `sys_open` using `O_CREAT | O_WRONLY`, setting file permissions in octal, and writing content with `sys_write` (syscall 4).

---

## Creating a File with `sys_open`

Same syscall as opening (`eax = 5`), but with different flags in `ecx`.

### Flags needed

| C constant | Octal | Meaning |
|---|---|---|
| `O_WRONLY` | `01` | Open for writing |
| `O_CREAT` | `0100` | Create file if it doesn't exist |

Combine flags with bitwise OR:

```
O_WRONLY | O_CREAT  =  01 OR 0100  =  0101  (octal)
```

In NASM, octal is written with an `o` suffix:

```nasm
mov ecx, 101o           ; 0101 octal = O_WRONLY | O_CREAT
```

> **Why octal?** These flags are defined in `fcntl.h` / `unistd.h` using octal constants. The `o` suffix in NASM tells the assembler to interpret the number as octal, not decimal.

---

## File Permissions (mode) — `edx`

When creating a file, `edx` sets the permissions. Permissions use octal flags too.

| C constant | Octal | Meaning |
|---|---|---|
| `S_IRUSR` | `400` | User can read |
| `S_IWUSR` | `200` | User can write |
| `S_IXUSR` | `100` | User can execute |

OR them together by adding (since no two bits overlap):

```
400 + 200 + 100 = 700  (octal)  →  rwx for owner
```

```nasm
mov edx, 700o           ; owner: read + write + execute
```

### Common permission combos

| Octal | Permissions |
|---|---|
| `700o` | Owner rwx |
| `644o` | Owner rw, group/other r |
| `600o` | Owner rw only |
| `755o` | Owner rwx, group/other rx |

---

## `sys_write` — Write to a File

| Register | Value |
|---|---|
| `eax` | `4` (sys_write) |
| `ebx` | fd — file descriptor |
| `ecx` | pointer to data to write |
| `edx` | number of bytes to write |

Return value: `eax` = bytes written.

Works for any fd: `1` = stdout, `2` = stderr, or any fd from `sys_open`.

---

## Declaring the String to Write

```nasm
section .data
    to_write    db    "Hello World!", 0Ah, 0Dh
    ; 0Ah = 0x0A = newline (\n)
    ; 0Dh = 0x0D = carriage return (\r)
    ; Total: 12 chars + 2 = 14 bytes
```

Count bytes carefully — the byte count in `edx` must match exactly.

| Character | Bytes |
|---|---|
| `"Hello World!"` | 12 |
| `0Ah` (newline) | 1 |
| `0Dh` (carriage return) | 1 |
| **Total** | **14** |

> Use `$` and `$$` NASM expressions to avoid manual counting:
> ```nasm
> to_write    db    "Hello World!", 0Ah, 0Dh
> to_write_len equ  $ - to_write      ; computed at assembly time
> ```
> Then use `to_write_len` in `edx` instead of hardcoding `14`.

---

## Full Program

```nasm
section .data
    filename    db    "/home/user/documents/test.txt", 0
    to_write    db    "Hello World!", 0Ah, 0Dh
    to_write_len equ  $ - to_write     ; = 14

section .text
    global main

main:
    ; --- sys_open: create + open for writing ---
    mov eax, 5              ; sys_open
    mov ebx, filename       ; path
    mov ecx, 101o           ; O_WRONLY | O_CREAT (octal)
    mov edx, 700o           ; permissions: rwx for owner
    int 0x80                ; eax = file descriptor

    ; --- sys_write ---
    mov ebx, eax            ; fd → ebx (before overwriting eax)
    mov eax, 4              ; sys_write
    mov ecx, to_write       ; pointer to string
    mov edx, to_write_len   ; byte count
    int 0x80                ; eax = bytes written

    ; --- exit ---
    mov eax, 1
    mov ebx, 0
    int 0x80
```

---

## Compile & Run

```bash
nasm -f elf -o write.o write.s
gcc -no-pie -m32 -o write write.o
./write
cat /home/user/documents/test.txt
# Hello World!
```

Check permissions:
```bash
ls -la /home/user/documents/test.txt
# -rwxr--r-- ... test.txt   ← 700 gives owner rwx
```

---

## GDB Session

```bash
gdb write
(gdb) layout asm
(gdb) break main
(gdb) run
```

**After sys_open:**
```
(gdb) info registers eax
# eax = 0x3    3            ← fd assigned to the new file
```

**After sys_write:**
```
(gdb) info registers eax
# eax = 0xe    14           ← 14 bytes written (0xe hex = 14 decimal)
```

If `eax` after `sys_write` is negative, the write failed (check fd and permissions).

---

## Common Bug: Forgetting `int 0x80`

The instructor hit this in the video — the file was created but empty because the `int 0x80` after the `sys_write` setup was missing. The syscall never fired, nothing was written.

Always verify all four registers are set (`eax`, `ebx`, `ecx`, `edx`) AND `int 0x80` is present.

---

## Octal vs Hex vs Decimal in NASM

| Notation | NASM syntax | Example |
|---|---|---|
| Decimal | plain number | `mov eax, 15` |
| Hex | `0x` prefix or `h` suffix | `mov eax, 0xF` or `mov eax, 0Fh` |
| Octal | `o` suffix | `mov ecx, 101o` |
| Binary | `0b` prefix | `mov al, 0b1010` |

---

## Key Takeaways

- Creating a file = `sys_open` with `O_CREAT | O_WRONLY` flags ORed together
- Flags and permissions are **octal** — use `o` suffix in NASM
- OR flags by adding their octal values (no bit overlap): `400 + 200 + 100 = 700`
- `sys_write` (eax=4): `ebx`=fd, `ecx`=data ptr, `edx`=byte count
- Save fd from `sys_open`'s `eax` to `ebx` BEFORE overwriting `eax` with `4`
- Byte count in `edx` must include all bytes — newline (`0Ah`) and carriage return (`0Dh`) count
- Use `$ - label` to let NASM compute string length automatically

---

## Complete Syscall File I/O Reference

| Syscall | `eax` | `ebx` | `ecx` | `edx` | Returns |
|---|---|---|---|---|---|
| `sys_open` | 5 | path ptr | flags | mode (octal) | fd |
| `sys_read` | 3 | fd | buffer ptr | byte count | bytes read |
| `sys_write` | 4 | fd | data ptr | byte count | bytes written |
| `sys_lseek` | 19 | fd | offset | whence | new position |
| `sys_close` | 6 | fd | — | — | 0 on success |

---

## What's Next
More complex programs combining all these concepts — reading from one file, processing, writing to another.
