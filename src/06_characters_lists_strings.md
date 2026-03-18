# Lecture 06 — Characters, Lists, and Strings

> Covers ASCII encoding, lists of bytes in memory, the null terminator concept, and strings. Also introduces setting GDB to Intel syntax so disassembly matches NASM output.

---

## One-Time GDB Setup: Intel Syntax

By default GDB shows AT&T syntax. Switch it permanently to Intel so it matches your NASM code:

```bash
echo "set disassembly-flavor intel" >> ~/.gdbinit
```

**AT&T vs Intel in GDB disassembly:**

| | AT&T (default) | Intel (after fix) |
|---|---|---|
| Registers | `%eax` | `eax` |
| Operand order | `src, dst` | `dst, src` |
| Immediates | `$1` | `1` |

After this change, GDB output matches your source. No more mental translation.

---

## Characters — Everything is a Number

Assembly has no "char" type. A character is just a byte whose numeric value maps to a letter via an encoding standard. The most common is **ASCII**.

```nasm
section .data
    ch    db    'A'     ; stores the byte 65, not the letter 'A'
```

NASM accepts single-quoted characters as syntactic sugar — it converts them to their ASCII value at assembly time.

### Running it:

```nasm
section .text
    global _start

_start:
    mov bl, [ch]        ; bl = 65
    mov eax, 1
    int 0x80
```

```bash
./data
echo $?
# 65
```

`'A'` → `65`. That's ASCII.

### Key ASCII values to know:

| Char | Decimal | Hex |
|---|---|---|
| `'A'` | 65 | 0x41 |
| `'B'` | 66 | 0x42 |
| `'a'` | 97 | 0x61 |
| `'0'` | 48 | 0x30 |
| newline `\n` | 10 | 0x0a |
| null | 0 | 0x00 |

Full table: look up "ASCII table" — capital letters start at 65, lowercase at 97.

> In GDB, memory is shown in hex. `0x41` = `65` = `'A'`. Get comfortable converting between these three representations.

---

## Lists

You can declare multiple values in one line by comma-separating them:

```nasm
section .data
    list    db    1, 2, 3, 4      ; four consecutive bytes in memory
```

Each value gets its own byte, stored sequentially at adjacent addresses.

### What this looks like in memory (GDB):

```bash
gdb data
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) x/x 0x804a000
# 0x804a000:    0x04030201
```

All four bytes packed into one 32-bit slot: `0x04 0x03 0x02 0x01` (little-endian, so byte at lowest address is rightmost in the hex display).

### The end-of-list problem

Memory doesn't know where your list ends. After your list, the next bytes in memory belong to something else. If you iterate through the list, how do you know when to stop?

**You need a sentinel value — a value that signals "end of list."**

Common choices:
- `0` (null terminator) — if you know zero won't appear in your data
- `-1` — if you know your data is always positive

```nasm
list    db    1, 2, 3, 4, 0       ; 0 = end of list
```

When iterating: keep going until you hit `0`. That's your stop condition.

---

## Strings

A string is just a list of character bytes with a null terminator (`0`) at the end.

```nasm
section .data
    str1    db    'A', 'B', 'A', 0       ; string "ABA" + null terminator
    str2    db    'C', 'D', 'E', 0       ; string "CDE" + null terminator
```

NASM shorthand — you can write string literals directly:

```nasm
    str1    db    "ABA", 0
    str2    db    "CDE", 0
```

Both are identical. The double-quoted form just expands each character to its ASCII byte.

### What this looks like in memory (GDB):

```bash
(gdb) x/3x 0x804a000
# 0x804a000:    0x00414241    0x00454443    0x00000000
#                ^^              ^^
#             ABA + null      CDE + null
```

Breaking it down (little-endian, reading right to left within each slot):
```
0x41 0x42 0x41 0x00  →  'A' 'B' 'A'  null    (str1)
0x43 0x44 0x45 0x00  →  'C' 'D' 'E'  null    (str2)
```

**The `0x00` between the strings is doing critical work.** Without it:
```
... 0x41 0x42 0x41 0x43 0x44 0x45 ...
```
You'd have no way of knowing where `str1` ends and `str2` begins when iterating. The null terminator creates the boundary.

### Why this matters (C connection)

This is exactly how C strings work — `char *str = "hello"` is a null-terminated byte array in memory. Assembly is where that convention comes from.

---

## Full Example Program

```nasm
section .data
    str1    db    "ABA", 0         ; null-terminated string
    str2    db    "CDE", 0         ; second string right after

section .text
    global _start

_start:
    mov ebx, str1       ; load address of str1 (to inspect in GDB)
    mov eax, 1
    int 0x80
```

### GDB session

```bash
gdb data
(gdb) layout asm
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) x/3x $ebx
# Shows memory at str1's address:
# 0x804a000:    0x00414241    0x00454443    ...
#                ABA\0           CDE\0
```

---

## Key Takeaways

- Characters are just numbers — ASCII maps letters to byte values
- `'A'` = `65` = `0x41`. NASM handles the conversion.
- Lists are consecutive bytes in memory — no built-in length tracking
- **Always null-terminate lists and strings** so you know where they end
- Strings = list of ASCII bytes + `0` at the end
- In GDB hex output, `0x41` = `'A'` — practice reading ASCII in hex

---

## What's Next
Next lecture: uninitialized data (`.bss` section) — reserving memory space without giving it an initial value.
