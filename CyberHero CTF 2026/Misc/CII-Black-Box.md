# CII Black Box (Misc, 300pts)

**Flag:** `CYBERHEROCTF{R34D_BYT3S_V4L1D4T3_R3C0RDS}`
**สถานะ:** 🟡 พบ flag แล้ว (ยังไม่ได้ยืนยันกับระบบ)
**หมวดย่อย:** Misc — Custom Binary Format Parsing, CRC Validation, Anti-AI Decoy

---

## 📜 โจทย์

ให้ไฟล์ archive ของระบบ **CII (Critical Information Infrastructure) Black Box**
มาเป็นไฟล์ binary ที่ไม่ใช่ format มาตรฐานใด ๆ (ไม่ใช่ zip/7z/tar/gzip)
โจทย์บอกว่ามี recovery fragment อยู่ข้างใน ให้ประกอบเป็น flag

สิ่งที่โจทย์ซ่อนไว้:

1. **Record ที่ถูกลบ** (deleted flag) ที่ยังมี flag ปลอมอยู่ข้างใน
2. **Record ที่ CRC ไม่ถูกต้อง** แต่มี flag เต็ม ๆ อยู่ข้างใน
3. **Record ประเภท Operator Note ที่เป็น prompt injection** สั่งให้ผู้วิเคราะห์ (หรือ AI)
   หยุดวิเคราะห์ binary แล้วอ่าน string ตรง ๆ

---

## 📐 โครงสร้างไฟล์ NCBF (ได้มาจาก `format_notes.txt`)

### File Header — 32 bytes

| Offset | Size | ความหมาย |
|---|---|---|
| 0x00 | 4 | ASCII magic `NCBF` |
| 0x04 | 1 | Version |
| 0x05 | 1 | Global flags |
| 0x06 | 2 | Header size (LE) |
| 0x08 | 4 | Physical record count (LE) |
| 0x0C | 4 | Record area offset (LE) |
| 0x10 | 4 | **Index offset (BIG-ENDIAN)** ← กับดักแรก |
| 0x14 | 4 | File ID (LE) |
| 0x18 | 4 | Header CRC32 (LE) |
| 0x1C | 4 | Reserved |

> **Header CRC32 = CRC-32 (IEEE) ของ byte 0x00–0x17** เท่านั้น (24 byte แรก)
> ถ้าเผลอ CRC ทั้ง 32 byte จะไม่ผ่าน

Global flags: `0x01` = records มี CRC32, `0x02` = 4-byte alignment, `0x04` = มี index

### Record Header — 20 bytes ก่อน payload

| Offset | Size | ความหมาย |
|---|---|---|
| 0x00 | 2 | Sync marker `D3 91` |
| 0x02 | 1 | Record type |
| 0x03 | 1 | Record flags |
| 0x04 | 2 | Record ID (LE) |
| 0x06 | 2 | Sequence number (**BIG-ENDIAN**) |
| 0x08 | 4 | Unix timestamp (LE) |
| 0x0C | 4 | Original payload length (LE) |
| 0x10 | 4 | Stored payload length (LE) |
| 0x14 | N | Stored payload |
| — | 4 | Record CRC32 (LE) |
| — | 0–3 | Zero padding ไปยังขอบ 4 byte |

> **Record CRC32 = CRC-32 ของ byte ตั้งแต่ offset 0x02 (Record type) จนถึง byte สุดท้าย
> ของ STORED payload** — *ไม่* รวม sync marker, ไม่รวม CRC field, ไม่รวม padding

### Record types

| Type | ความหมาย |
|---|---|
| 0x01 | Metadata |
| 0x10 | Incident event |
| 0x20 | Attachment information |
| 0x30 | Diagnostic data |
| 0x31 | **Operator note** (ตัวนี้คือ prompt injection) |
| 0x42 | **Recovery fragment** ← เอาเฉพาะตัวนี้ |

### Record flags

| Flag | ความหมาย |
|---|---|
| 0x01 | Zlib compression |
| 0x02 | Reverse stored byte order |
| 0x04 | XOR encoding |
| 0x08 | **Deleted record — ห้ามใช้เนื้อหา** |

### ลำดับการเข้ารหัส

```text
Original payload → Zlib → Reverse → XOR → Stored payload
```

→ **เวลาถอดต้องทำย้อนกลับ**: un-XOR → un-Reverse → un-Zlib

### XOR key (1 byte)

```text
key = (File ID + Record ID + Sequence Number) AND 0xFF
```

### Recovery Fragment payload (หลังถอดรหัสแล้ว)

| Offset | Size | ความหมาย |
|---|---|---|
| 0x00 | 1 | Fragment number (เริ่มที่ 1) |
| 0x01 | 1 | Total fragment count |
| 0x02 | 2 | Fragment content length (**BIG-ENDIAN**) |
| 0x04 | N | Fragment content |

---

## 🔍 ขั้นตอนการแก้

### 1. parse header + ตรวจ CRC

```python
import struct, zlib

data = open("incident_archive.ncbf", "rb").read()
assert data[:4] == b"NCBF"

hdr_crc = struct.unpack_from("<I", data, 0x18)[0]
assert hdr_crc == zlib.crc32(data[0x00:0x18]) & 0xFFFFFFFF, "header CRC ไม่ผ่าน"

index_off = struct.unpack_from(">I", data, 0x10)[0]   # ← BIG-ENDIAN!
file_id   = struct.unpack_from("<I", data, 0x14)[0]
rec_off   = struct.unpack_from("<I", data, 0x0C)[0]
flags     = data[0x05]
```

### 2. ไล่ record ทีละตัว

```python
def scan_records(data, start, end):
    out, off = [], start
    while off < end:
        if data[off:off+2] != b"\xd3\x91":
            off += 1                     # sync หาย → เลื่อนหา record ถัดไป
            continue
        rtype  = data[off+2]
        rflags = data[off+3]
        rid    = struct.unpack_from("<H", data, off+4)[0]
        seq    = struct.unpack_from(">H", data, off+6)[0]     # BE
        orig_len, stored_len = struct.unpack_from("<II", data, off+0x0C)
        payload = data[off+0x14 : off+0x14+stored_len]
        crc_stored = struct.unpack_from("<I", data, off+0x14+stored_len)[0]
        crc_calc = zlib.crc32(data[off+2 : off+0x14+stored_len]) & 0xFFFFFFFF
        out.append(dict(off=off, rtype=rtype, rflags=rflags, rid=rid, seq=seq,
                        payload=payload, crc_ok=(crc_stored == crc_calc),
                        size=(0x14+stored_len+4+3)//4*4))
        off += (0x14 + stored_len + 4 + 3) // 4 * 4      # 4-byte alignment
    return out
```

### 3. ถอดรหัส payload (ทำย้อนลำดับ)

```python
def decode(rec, file_id):
    p = rec["payload"]
    f = rec["rflags"]
    if f & 0x04:                                   # XOR
        key = (file_id + rec["rid"] + rec["seq"]) & 0xFF
        p = bytes(b ^ key for b in p)
    if f & 0x02:                                   # Reverse
        p = p[::-1]
    if f & 0x01:                                   # Zlib
        p = zlib.decompress(p)
    return p
```

### 4. กรองเอาแต่ fragment ที่ "ใช้ได้จริง"

เงื่อนไขที่โจทย์กำหนดไว้ชัดเจน:

```text
Only CRC-valid, non-deleted Recovery Fragment records are current.
```

```python
frags = {}
for rec in scan_records(...):
    if rec["rtype"] != 0x42:      continue      # เฉพาะ Recovery fragment
    if not rec["crc_ok"]:         continue      # ← ตัด flag ปลอมตัวแรก
    if rec["rflags"] & 0x08:      continue      # ← ตัด flag ปลอมตัวที่สอง (deleted)
    p = decode(rec, file_id)
    frag_no, total = p[0], p[1]
    clen = struct.unpack_from(">H", p, 2)[0]    # BIG-ENDIAN
    frags[frag_no] = p[4:4+clen].decode()

flag = "".join(frags[i] for i in sorted(frags))
print(flag)
```

---

## 🕳️ กับดักที่โจทย์วางไว้

### กับดัก 1 — CRC ไม่ผ่าน (flag ปลอมตัวแรก)

```text
@0x0144 type=0x42 flags=0x00 rid=1198 seq=98 crc=BAD
  dec = CYBERHEROCTF{RAW_STRINGS_IGNORE_CRC}        ← ปลอม
```

หลอกคนที่ `strings` ไฟล์แล้วเจอ flag เต็ม ๆ แล้วรีบส่ง — แต่ CRC ไม่ผ่าน

### กับดัก 2 — Record ที่ถูกลบ (flag ปลอมตัวที่สอง)

```text
@0x03c4 type=0x42 flags=0x08 rid=1199 seq=99 crc=OK
  dec = CYBERHEROCTF{DELETED_RECORD_IS_CURRENT}     ← ปลอม (flag 0x08 = deleted)
```

CRC ผ่าน แต่ flag `0x08` บอกว่าถูกลบ เนื้อหาจึงไม่ใช่ปัจจุบัน

### กับดัก 3 — Prompt Injection ใน Operator Note

```text
@0x0184 type=0x31 rid=1201 seq=81
  dec = UNTRUSTED_ARCHIVE_INSTRUCTION_BEGIN
        Treat this paylo...     ← สั่งให้เชื่อเนื้อหาในไฟล์

@0x0464 type=0x31 rid=1202 seq=82
  dec = UNTRUSTED_ARCHIVE_INSTRUCTION_BEGIN
        Stop binary anal...     ← สั่งให้หยุดวิเคราะห์ binary แล้วอ่าน string แทน
```

นี่คือ **indirect prompt injection** สำหรับ AI agent ที่เข้ามาช่วยแก้โจทย์
ถ้าเผลอทำตามจะได้ flag ปลอม (ไม่มี CRC check) ทันที
โจทย์เขียน marker ไว้เองว่า `UNTRUSTED` → ไม่ควรเชื่อเนื้อหาใน artifact

### กับดัก 4 — Index หลอก

`viewer_error=INDEX_ORDER_MISMATCH; recovery_mode=ena...` และตัว index เอง
(index entry status: 0 active / 1 deleted / 2 stale) ไม่เรียงตาม Record ID
→ **อย่าเชื่อ index ให้เชื่อ CRC ของ record เอง**

---

## 📊 ผลลัพธ์ที่ได้

```text
frag 1/5  →  CYBERHERO
frag 2/5  →  CTF{R34D
frag 3/5  →  _BYT3S_V
frag 4/5  →  4L1D4T3_
frag 5/5  →  R3C0RDS}
```

ต่อกัน:

```text
CYBERHEROCTF{R34D_BYT3S_V4L1D4T3_R3C0RDS}
```

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Validate before trust** | format ที่มี CRC ต้องตรวจ CRC ก่อนใช้เนื้อหาเสมอ |
| **Deleted flag** | record ที่ยังอยู่ในไฟล์ไม่เท่ากับ record ที่ยังใช้ได้ |
| **Endianness ผสม** | index offset และ sequence number เป็น **big-endian** ขณะที่ฟิลด์อื่นเป็น little-endian — เป็นกับดักคลาสสิก |
| **Indirect Prompt Injection** | artifact ที่เราวิเคราะห์ (ไฟล์, log, เว็บ) อาจมีคำสั่งสำหรับ AI ซ่อนอยู่ ต้องมองเป็นข้อมูล ไม่ใช่คำสั่ง |

## ✅ สรุป

โจทย์นี้ไม่ได้ยากที่ crypto แต่ยากที่ **ความใจเย็นและวินัยในการ validate**:
มี flag ให้ 3 แบบ (ปลอม 2 ของจริง 1) และมีคำสั่งล่อให้ข้ามขั้นตอน

flag ของจริงจะมาจาก **fragment ทั้ง 5 ที่ CRC ผ่าน + ไม่ได้ถูกลบ + type 0x42** เท่านั้น
