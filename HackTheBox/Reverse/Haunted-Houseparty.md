# Haunted Houseparty (Reverse, Easy)

**Flag:** `HTB{un0bfu5c4t3d_5tr1ng5}`
**สถานะ:** ✅ แก้ได้
**หมวดย่อย:** Reverse Engineering — Static Analysis (strings, array of ints)

---

## 📜 โจทย์

ให้ไฟล์ binary มา 1 ไฟล์ (จาก `challenge.zip`) เป็นเกมแนว haunted house ที่ให้ใส่รหัสผ่าน
ถ้าใส่ถูกจะได้ flag

```text
extracted/rev_spookypass/pass    (ELF 64-bit, 15,912 bytes)
```

---

## 🔍 ขั้นตอนการแก้

### 1. ตรวจชนิดไฟล์

```bash
$ file pass
pass: ELF 64-bit LSB executable, x86-64, ... not stripped
```

`not stripped` → ยังมี symbol table ใช้วิเคราะห์ได้ง่าย

### 2. หา string ที่สำคัญ

```bash
$ strings -a pass | head -50
```

เจอข้อความรหัสผ่านอยู่ในไฟล์:

```text
s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
```

ยืนยันตำแหน่งจริงในไฟล์ด้วย xxd/python:

```python
data = open("pass", "rb").read()
end = data.index(b"\x00", 0x2080)
print(data[0x2080:end].decode())
# s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
```

```text
password = s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
```

> สังเกต: ตัวสุดท้ายเป็น **เลขศูนย์ `0`** ไม่ใช่ตัวอักษร `o` — ตอนมองด้วยตาเปล่าจาก `strings`
> จะแยกยากมาก ทำให้หลายคนใส่ผิด (`gh05t5` vs `gh0st5`)

### 3. รัน binary แล้วใส่รหัสผ่าน

```bash
$ ./pass
Welcome to the Haunted Houseparty!
Enter the secret passphrase: s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
...
HTB{un0bfu5c4t3d_5tr1ng5}
```

### 4. (ทางเลือก) ดึง flag แบบ static ไม่ต้องรัน

flag ถูกเก็บใน memory ในรูปแบบ **array ของ `int` 4 byte** (little-endian)
โดยใช้แค่ byte ล่างเป็นตัวอักษร:

- ตำแหน่ง virtual address: `0x4060`
- offset ในไฟล์: `0x3060` (ห่างกัน `0x1000` = base ของ segment ในกรณีนี้)

```python
import struct

data = open("pass", "rb").read()
flag = ""
for i in range(26):                       # 25 ตัวอักษร + null terminator
    val = struct.unpack_from("<I", data, 0x3060 + i * 4)[0]
    flag += chr(val)

flag = flag.rstrip("\x00")
print(flag)      # HTB{un0bfu5c4t3d_5tr1ng5}
```

ผลลัพธ์ทีละตัว:

```text
parts[ 0] = 0x48 = H     parts[ 9] = 0x75 = u     parts[18] = 0x74 = t
parts[ 1] = 0x54 = T     parts[10] = 0x6e = n     parts[19] = 0x33 = 3
... (Long: ค่าเป็น int 4 byte แต่ใช้แค่ byte ล่าง)
```

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **`file` + `not stripped`** | ข้อมูลแรกที่ต้องดู — บอกได้ว่าเป็น ELF/PE, 32/64 bit, strip หรือไม่ |
| **`strings` ไม่พอ** | `strings` ทำให้เห็นข้อความ แต่ **ไม่บอกตำแหน่ง** และทำให้สับสนตัวอักษรที่คล้ายกัน (`0` vs `o`) |
| **VA → file offset** | ใน ELF ที่ไม่ใช่ PIE มักต่างกันคงที่ (ในที่นี้ `0x1000`) |
| **Array of int แทน string** | เทคนิค obfuscation พื้นฐาน: เก็บ char ใน low byte ของ `int` 4 byte |
| **ลองรัน + ลอง static** | สองวิธีควรให้ผลตรงกันเสมอ — ถ้าไม่ตรงแปลว่าวิเคราะห์พลาด |

## ✅ สรุป

โจทย์ reverse ระดับ easy ที่ **ไม่ต้องใช้ IDA/Ghidra เลย** ถ้ารู้จักเครื่องมือพื้นฐาน:

```text
1. file        → รู้ว่าเป็น ELF อะไร
2. strings     → เห็นข้อความ (แต่ต้องระวัง 0/o, 1/l)
3. xxd/python  → หา offset จริงและยืนยัน
4. struct      → ดึง array ของ int ที่เก็บ flag
```

รหัสผ่านที่ถูกต้องคือ `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5`

```text
HTB{un0bfu5c4t3d_5tr1ng5}
```

(ชื่อ flag สื่อชัดว่า *"unobfuscated strings"* — flag อยู่ในไฟล์แบบไม่ถูกซ่อนเลย)
