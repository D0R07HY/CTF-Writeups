# 🎖️ CyberHero CTF 2026

**ทีม:** Singularity
**ผู้เล่น:** D0R07HY2
**ช่วงการแข่ง:** กันยายน 2026

---

## 📊 สรุปผล

| # | Challenge | Category | Points | สถานะ |
|---|---|---|---|---|
| 1 | Overpost | Web | 300 | ✅ ยืนยันกับระบบแล้ว |
| 2 | Faultline Ledger | Crypto | 300 | ✅ ยืนยันกับระบบแล้ว |
| 3 | CII Black Box | Misc | 300 | 🟡 พบ flag |
| 4 | Pieces of the Past | Misc | 300 | 🟡 พบ flag |
| 5 | X3 Step | Misc | 100 | 🟡 พบ flag |
| 6 | Factory Incident | Forensics | 300 | 🟡 พบ flag |
| 7 | Operation Cascade | Forensics | 300 | 🟡 พบ flag |
| 8 | Capture The Hill | OSINT | 300 | 🔴 ยังไม่สำเร็จ |

> **สัญลักษณ์สถานะ**
> ✅ = ส่ง flag เข้าระบบแล้วถูกต้อง
> 🟡 = แกะได้ flag แล้ว แต่ยังไม่ได้ยืนยันกับระบบ
> 🔴 = ยังไม่สำเร็จ / ต้องวิเคราะห์ต่อ

---

## 📂 รายการ Writeup

### 🌐 Web

| # | Challenge | Points | Writeup |
|---|---|---|---|
| 1 | Overpost | 300 | [Overpost](./Web/Overpost.md) |

### 🔐 Crypto

| # | Challenge | Points | Writeup |
|---|---|---|---|
| 2 | Faultline Ledger | 300 | [Faultline Ledger](./Crypto/Faultline-Ledger.md) |

### 🧩 Misc

| # | Challenge | Points | Writeup |
|---|---|---|---|
| 3 | CII Black Box | 300 | [CII Black Box](./Misc/CII-Black-Box.md) |
| 4 | Pieces of the Past | 300 | [Pieces of the Past](./Misc/Pieces-of-the-Past.md) |
| 5 | X3 Step | 100 | [X3 Step](./Misc/X3-Step.md) |

### 🔬 Forensics

| # | Challenge | Points | Writeup |
|---|---|---|---|
| 6 | Factory Incident | 300 | [Factory Incident](./Forensics/Factory-Incident.md) |
| 7 | Operation Cascade | 300 | [Operation Cascade](./Forensics/Operation-Cascade.md) |

### 🕵️ OSINT

| # | Challenge | Points | Writeup |
|---|---|---|---|
| 8 | Capture The Hill | 300 | [Capture The Hill](./OSINT/Capture-The-Hill.md) |

---

## 🧠 แนวคิดหลักที่ใช้ในชุดโจทย์นี้

| เทคนิค | ใช้ในโจทย์ |
|---|---|
| RSA CRT Fault Attack (Bellcore / Lenstra) | Faultline Ledger |
| SHA-256 Length Extension | Faultline Ledger |
| AES-CTR Keystream Reuse | Faultline Ledger |
| Custom Binary Format Parsing + CRC Validation | CII Black Box |
| Prompt Injection บน artifact (decoy) | CII Black Box |
| A1Z26 / XOR / Wayback Machine | Pieces of the Past |
| Multi-step XOR + ROT13 | X3 Step |
| Covert Channel ใน IP ID high byte | Factory Incident |
| ASCII-Art QR Reconstruction | Operation Cascade |
| Offline Disk Forensics (LVM + qemu-nbd) | Overpost |
| SUID Binary Abuse (bash `-p`) | Overpost |

---

## 💡 บทเรียนที่ได้

1. **อ่าน format ให้ละเอียดก่อน parse** — CII Black Box ซ่อนทั้ง record ที่ถูกลบ (flag ปลอม) และ record ประเภทคำสั่ง (prompt injection) ไว้ ถ้าไม่ validate CRC และ type จะได้ flag ผิดทันที
2. **Decoy คือของจริงที่ซ่อนอยู่ในที่ที่คนขี้เกียจไม่ไปดู** — Factory Incident มี flag ปลอมกระจายอยู่เกือบ 20 ที่ แต่ของจริงอยู่ใน header ของ packet ที่คนมักมองข้าม (IP ID)
3. **อย่าเชื่อสิ่งที่โจทย์บอก** — Overpost บอกว่าบล็อกไฟล์อันตรายแล้ว แต่ลืม `.pyw` ซึ่งอยู่ในลิสต์ที่รันได้
4. **ทุกอย่างที่รันได้ = รันได้** — Faultline Ledger ไม่ได้พังแค่จุดเดียว แต่พัง 3 ชั้น (fault injection, length extension, keystream reuse) ให้ประกอบกันเป็น flag
