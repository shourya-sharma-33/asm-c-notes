# Lecture 23 — File Seeking with `lseek`

> Covers `sys_lseek` (syscall 19) — moving the file position pointer to read from specific locations in a file. Demonstrated with fixed-width records.

---

## What is `lseek`?

When you open a file, the OS maintains a **position pointer** — where the next read starts. By default it starts at byte 0 and advances after each read.

`lseek` lets you move that pointer to any byte offset before reading. Primary use case: **fixed-width records** — when you know every record is N bytes, you can jump directly to record K without reading all the preceding ones.

---

## Fixed-Width Record Example

File contents (`test.txt`) — each line is 9 visible characters + 1 newline = **10 bytes per record**:

```
000 111\n
222 333\n
444 555\n
...
```

To read record 3 (zero-indexed: record 2): seek `2 × 10 = 20` bytes from the start, then read 10 bytes.

---

## `sys_lseek` — Syscall 19

### Syscall table entry

| Register | Value |
|---|---|
| `eax` | `19` (sys_lseek) |
| `ebx` | `int fd` — file descriptor |
| `ecx` | `off_t offset` — number of bytes to move |
| `edx` | `int whence` — where to seek FROM |

### `whence` values (from `unistd.h`)

| C constant | Value | Meaning |
|---|---|---|
| `SEEK_SET` | `0` | Offset from start of file |
| `SEEK_CUR` | `1` | Offset from current position |
| `SEEK_END` | `2` | Offset from end of file (use negative offset to move backward) |

### Return value

`eax` = the new absolute byte position in the file (i.e. the offset from the start). Use this to confirm the seek worked.

---

## Full Program: Open → Seek → Read

```nasm
section .data
    filename    db    "/path/to/test.txt", 0

section .bss
    buffer      resb    10          ; one record = 10 bytes

section .text
    global main

main:
    ; --- sys_open ---
    mov eax, 5                      ; sys_open
    mov ebx, filename
    mov ecx, 0                      ; O_RDONLY
    int 0x80                        ; eax = file descriptor

    ; --- sys_lseek: jump to byte 20 (record 3) ---
    mov ebx, eax                    ; fd → ebx (save before overwriting eax)
    mov eax, 19                     ; sys_lseek
    mov ecx, 20                     ; offset = 20 bytes
    mov edx, 0                      ; SEEK_SET = from start of file
    int 0x80                        ; eax = 20 (new file position)

    ; --- sys_read: read 10 bytes from current position ---
    ; ebx still holds fd
    mov eax, 3                      ; sys_read
    mov ecx, buffer                 ; destination buffer
    mov edx, 10                     ; read 10 bytes (one record)
    int 0x80                        ; eax = bytes read, buffer filled

    ; --- exit ---
    mov eax, 1
    mov ebx, 0
    int 0x80
```

**After this runs, `buffer` contains the third record from the file.**

---

## Compile & Run

```bash
nasm -f elf -o seek.o seek.s
gcc -no-pie -m32 -o seek seek.o
./seek
```

---

## GDB Session

```bash
gdb seek
(gdb) layout asm
(gdb) break main
(gdb) run
```

**After sys_open:**
```
(gdb) info registers eax
# eax = 0x3    3            ← file descriptor
```

**After sys_lseek:**
```
(gdb) info registers eax
# eax = 0x14   20           ← 0x14 = 20 decimal = new file position
```

Confirms the seek worked — file pointer is now at byte 20.

**After sys_read — inspect buffer:**
```
(gdb) x/s 0x804c048         ; use the actual address of 'buffer' (shown in ecx before int 0x80)
# "444 555\n"               ← third record, read correctly
```

---

## How to Calculate the Seek Offset

```
record_number (0-indexed) × bytes_per_record = offset

Record 0: 0  × 10 = 0   bytes (start of file)
Record 1: 1  × 10 = 10  bytes
Record 2: 2  × 10 = 20  bytes  ← third record
Record 3: 3  × 10 = 30  bytes
```

**Newline counts as a byte.** A "9 character" line is actually 10 bytes (9 chars + `\n`). Always account for it.

---

## `whence` Use Cases

| Scenario | `whence` | `offset` |
|---|---|---|
| Jump to record N | `SEEK_SET` (0) | `N × record_size` |
| Skip forward N bytes from current position | `SEEK_CUR` (1) | `N` |
| Go to last N bytes of file | `SEEK_END` (2) | `-N` |
| Go back to start of file | `SEEK_SET` (0) | `0` |

---

## Syscall Reference (File Operations)

| Syscall | `eax` | `ebx` | `ecx` | `edx` | Returns |
|---|---|---|---|---|---|
| `sys_open` | 5 | path ptr | flags | mode | fd |
| `sys_read` | 3 | fd | buffer ptr | byte count | bytes read |
| `sys_write` | 4 | fd | buffer ptr | byte count | bytes written |
| `sys_lseek` | 19 | fd | offset | whence | new position |
| `sys_close` | 6 | fd | — | — | 0 on success |

---

## Key Takeaways

- `lseek` moves the file position pointer without reading any data
- Three seek modes: `SEEK_SET`(0) from start, `SEEK_CUR`(1) from current, `SEEK_END`(2) from end
- Return value = new absolute position in file (use to verify)
- Fixed-width records: offset = `record_index × record_size`
- **Always count the newline character** — `\n` is a byte (value 10)
- Save the fd to `ebx` before overwriting `eax` with the next syscall number

---

## What's Next
Writing to files — `sys_write` and `sys_close`.
