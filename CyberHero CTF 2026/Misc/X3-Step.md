# X3 Step (Misc, 100pts)

**Flag (ผู้สมัครหลัก):** `CYBERHEROCTF{MkaeX345fsimple}`
**สถานะ:** 🟡 พบ flag แล้ว (ยังไม่ได้ยืนยันกับระบบ)
**หมวดย่อย:** Misc — Encoding Chain (Hex → Reverse → ROT13)

---

## 📜 โจทย์

ให้ไฟล์ชื่อ `X3_step` มา เป็นไฟล์ข้อความขนาด 58 byte ที่มีแต่ตัวอักษร hex
ชื่อโจทย์บอกใบ้ว่ามี **3 ขั้น** ("X3 step")

---

## 🔍 ขั้นตอนการแก้

### 0. ดูไฟล์

```bash
$ file X3_step
X3_step: ASCII text, with no line terminators

$ cat X3_step
7d7279637a7666733534334b726e785a7b5347504245525545524f4c50
```

58 ตัวอักษร hex คู่ = 29 byte → decode ได้เลย

### 1. ขั้นที่ 1 — Hex decode

```python
h = "7d7279637a7666733534334b726e785a7b5347504245525545524f4c50"
b = bytes.fromhex(h)
print(b.decode('latin1'))
# }ryczvfs543KrnxZ{SGPBERUEROLP
```

ยังอ่านไม่ออก แต่สังเกตว่ามัน **ลงท้ายด้วย `}` และมี `{` อยู่ตรงกลาง**
→ น่าจะเป็น flag ที่ถูก **กลับด้าน (reverse)**

### 2. ขั้นที่ 2 — Reverse

```python
r = b[::-1]
print(r.decode('latin1'))
# PLOREUREBPGS{ZxnrK345sfvzcyr}
```

### 3. ขั้นที่ 3 — ROT13

```python
import codecs
print(codecs.encode(r.decode('latin1'), 'rot13'))
# CYBERHEROCTF{MkaeX345fsimple}
```

ได้ flag:

```text
CYBERHEROCTF{MkaeX345fsimple}
```

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Hex encoding** | 2 ตัวอักษร = 1 byte, สังเกตได้จาก `[0-9a-f]` ล้วนและความยาวเป็นเลขคู่ |
| **Reverse** | สังเกตจากเครื่องหมาย `{` `}` ที่อยู่ผิดตำแหน่ง (ปิดก่อน เปิดทีหลัง) |
| **ROT13** | Caesar shift 13 — เป็น self-inverse (เข้ารหัส/ถอดรหัสด้วยฟังก์ชันเดียวกัน) |
| **สังเกต `CYBERHERO`** | คำว่า `SGPBERUEROLP` คือ `CYBERHEROCTF` หลัง ROT13 → ช่วยยืนยันว่าถูกทาง |

> **เทคนิคสังเกต:** ขั้น reverse กับ ROT13 **สลับลำดับกันได้** เพราะ ROT13 ทำงานทีละตัวอักษร
> (char-wise) ส่วน reverse ทำแค่การสลับตำแหน่ง ดังนั้นไม่ว่าจะ "reverse แล้ว ROT13" หรือ
> "ROT13 แล้ว reverse" ก็ได้ผลเหมือนกัน — แต่ **hex decode ต้องทำก่อนเสมอ**

---

## 🛠️ Solve Script (ไฟล์เดียวจบ)

```python
import codecs

h = "7d7279637a7666733534334b726e785a7b5347504245525545524f4c50"

step1 = bytes.fromhex(h).decode('latin1')            # hex decode
print("step1 (hex):    ", step1)

step2 = step1[::-1]                                  # reverse
print("step2 (reverse):", step2)

step3 = codecs.encode(step2, 'rot13')                # rot13
print("step3 (rot13):  ", step3)

assert step3 == "CYBERHEROCTF{MkaeX345fsimple}"
print("\n[+] FLAG:", step3)
```

---

## ✅ สรุป

โจทย์นี้เป็น **encoding chain** พื้นฐาน 3 ชั้น — ไม่อาศัย crypto ที่แข็งแรงเลย
หัวใจคือการ **สังเกตโครงสร้าง** ของข้อความที่ได้ในแต่ละขั้น:

1. hex ล้วน ๆ ยาวเป็นเลขคู่ → `bytes.fromhex`
2. เจอ `{` `}` กลับด้าน → `[::-1]`
3. ข้อความยังเป็นตัวอักษรปนกันแบบ Caesar → ลอง ROT13 (และเห็น `SGPB` ≈ `CYBE` ชัด ๆ)

> **ข้อสังเกตเล็ก ๆ:** ผลลัพธ์ได้ `Mkae` (ไม่ใช่ `Make`) ซึ่งลักษณะคล้ายกับโจทย์
> Pieces of the Past ที่ได้ `f0rg3tsy` — คือเป็น "typo/leetspeak ที่ตั้งใจ" ของผู้เขียนโจทย์
> ให้ใช้ค่าที่ถอดได้ตรงตัวก่อนเสมอ
