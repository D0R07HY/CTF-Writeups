# Pieces of the Past (Misc, 300pts)

**Flag (ผู้สมัครหลัก):** `CYBERHEROCTF{m1sc_x0r_p4st3_n3v3r_f0rg3tsy}`
**สถานะ:** 🟡 พบ flag แล้ว (ยังไม่ได้ยืนยันกับระบบ)
**หมวดย่อย:** Misc — A1Z26, XOR, OSINT (Wayback Machine)

---

## 📜 โจทย์

โจทย์ให้เบาะแสเป็นชุดตัวเลข/ข้อความสั้น ๆ แล้วให้ไล่ตาม "ชิ้นส่วนในอดีต"
โดยมี 3 ขั้นตอนที่ต้องต่อกัน

---

## 🔍 ขั้นตอนการแก้

### ขั้นที่ 1 — A1Z26 แล้ว XOR

เบาะแสแรก:

```text
16 1 19 20 5
```

A1Z26 (A=1, B=2, …):

```text
16 = P, 1 = A, 19 = S, 20 = T, 5 = E  →  PASTE
```

คำว่า `PASTE` คือ **key XOR** สำหรับ ciphertext ที่ให้มา:

```python
import binascii
ct = bytes.fromhex("<ciphertext จากโจทย์>")
key = b"PASTE"
pt = bytes(c ^ key[i % len(key)] for i, c in enumerate(ct))
print(pt.decode())
```

ได้ผลลัพธ์:

```text
CYBERHEROCTF{m1sc_x0r_                 ← ชิ้นส่วนที่ 1 ของ flag
artifact = nWU5Lg0R                    ← รหัส paste
next     = pastebin
```

### ขั้นที่ 2 — ตามไปที่ Pastebin (และ Wayback Machine)

ลองเปิด `https://pastebin.com/nWU5Lg0R` ตรง ๆ → **ได้เนื้อหาปลอม (decoy)**
เนื้อหาจริงถูกลบ/แก้ไปแล้ว จึงต้องดู **สำเนาในอดีต**:

```text
https://web.archive.org/web/20260831033019id_/https://pastebin.com/nWU5Lg0R
```

> สังเกตคำต่อท้าย `id_` ใน URL ของ Wayback — ใช้เพื่อขอ **ไฟล์ต้นฉบับดิบ** ไม่ให้ archive
> เติมแบนเนอร์/แก้ HTML ของเรา

เนื้อหาที่ได้จากสำเนา:

```text
PROJECT AFTERIMAGE
Build 0.9.3

Recovery Fragment: p4st3_n3v3r_            ← ชิ้นส่วนที่ 2 ของ flag

ARCHIVE DATA: 27762622613d3e38             ← ciphertext ก้อนสุดท้าย

Same operation. Different secret.
The secret is hiding in plain sight.
```

### ขั้นที่ 3 — XOR อีกครั้ง แต่เปลี่ยน secret

คำใบ้บอกว่า "operation เดิม แต่ secret ต่างกัน" → ใช้ XOR เหมือนเดิม
แต่ key คือสิ่งที่ "ซ่อนอยู่ในที่ที่มองเห็น" = **ชื่อโปรเจกต์ `AFTERIMAGE`**

```python
ct  = bytes.fromhex("27762622613d3e38")
key = b"AFTERIMAGE"
pt  = bytes(c ^ key[i % len(key)] for i, c in enumerate(ct))
print(pt.decode())
# f0rg3tsy                                ← ชิ้นส่วนที่ 3 ของ flag
```

(ตรวจทีละไบต์จะเห็นว่า XOR ตรงตัวทุกตัวอักษร เช่น `0x27 ^ ord('A') = 0x66 = 'f'`)

### ประกอบ flag

```text
CYBERHEROCTF{ m1sc_x0r_  p4st3_n3v3r_  f0rg3tsy }
              ↑ขั้น 1      ↑ขั้น 2        ↑ขั้น 3
```

```text
CYBERHEROCTF{m1sc_x0r_p4st3_n3v3r_f0rg3tsy}
```

---

## 🤔 หมายเหตุเรื่องความกำกวมของชิ้นส่วนสุดท้าย

ผลลัพธ์จาก XOR ได้ **`f0rg3tsy`** ตรงตัว (deterministic) แต่ถ้าอ่านเชิงความหมาย
มันน่าจะตั้งใจสื่อคำว่า *"never forgets"* → `f0rg3ts`

ดังนั้นมีผู้สมัคร 2 แบบ:

| # | Candidate | ที่มา |
|---|---|---|
| A | `CYBERHEROCTF{m1sc_x0r_p4st3_n3v3r_f0rg3ts}` | อ่านเชิงความหมาย ("never forgets") |
| B | `CYBERHEROCTF{m1sc_x0r_p4st3_n3v3r_f0rg3tsy}` | ผลลัพธ์ XOR ตรงตัว (น่าจะเป็นค่าที่ตั้งใจ) |

ให้ลอง **B ก่อน** แล้วถ้าไม่ผ่านจึงลอง A

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **A1Z26** | การแปลงตัวเลข 1–26 เป็นตัวอักษร อังกฤษ เป็น cipher คลาสสิกที่โจทย์ misc ชอบใช้ |
| **Repeating-key XOR** | ถ้ารู้ key ยาวเท่าใดก็ถอดได้ทันที (ไม่ใช่ crypto ที่ปลอดภัยจริง) |
| **Wayback Machine** | เนื้อหาบน pastebin ที่ถูกลบ/แก้ ยังดูได้จาก archive — และต้องใช้ `id_` เพื่อเอาของดิบ |
| **"Secret hiding in plain sight"** | คำใบ้แบบนี้มักหมายถึง ชื่อเรื่อง/ชื่อไฟล์/ชื่อโปรเจกต์ ที่โชว์อยู่ตรงหน้าแล้ว |

## ✅ สรุป

โจทย์นี้เป็นลูกโซ่ OSINT + classic crypto 3 ทอด:

```text
A1Z26 → XOR(PASTE) → pastebin → Wayback Machine → XOR(AFTERIMAGE) → flag
```

ทอดที่ยากที่สุดไม่ใช่ crypto แต่คือ **การรู้จักมองหา snapshot ในอดีต**
เมื่อหน้าเว็บจริงถูกทำให้เป็น decoy ไปแล้ว
