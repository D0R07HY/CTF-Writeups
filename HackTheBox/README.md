# 🧪 HackTheBox

**ทีม:** Singularity
**ผู้เล่น:** D0R07HY2
**ช่วงที่เล่น:** กันยายน 2026
**Flag format:** `HTB{...}`

---

## 📊 สรุปผล

| # | Challenge | Category | Difficulty | สถานะ |
|---|---|---|---|---|
| 1 | Acknowledge the Corn | Forensics | Hard | ✅ แก้ได้ |
| 2 | Always Has Been | Crypto | Medium | ✅ แก้ได้ |
| 3 | Haunted Houseparty | Reverse | Easy | ✅ แก้ได้ |
| 4 | Artificial University | Web | — | 🔴 วิเคราะห์ไว้ (ยังไม่สำเร็จ) |
| 5 | Blinded | Pwn | Hard | 🔴 วิเคราะห์ไว้ (ยังไม่สำเร็จ) |

---

## 📂 รายการ Writeup

| # | Challenge | Writeup | Flag |
|---|---|---|---|
| 1 | Acknowledge the Corn | [Forensics/Acknowledge-The-Corn.md](./Forensics/Acknowledge-The-Corn.md) | `HTB{C2s_4r3_n0t_4lw4ys_4_s4f3_c0mmun1c4t10n_ch4nn3l}` |
| 2 | Always Has Been | [Crypto/Always-Has-Been.md](./Crypto/Always-Has-Been.md) | `HTB{w41T...it'5_4ll_L1n34r?????}` |
| 3 | Haunted Houseparty | [Reverse/Haunted-Houseparty.md](./Reverse/Haunted-Houseparty.md) | `HTB{un0bfu5c4t3d_5tr1ng5}` |
| 4 | Artificial University | [Web/Artificial-University.md](./Web/Artificial-University.md) | (ยังไม่พบ) |
| 5 | Blinded | [Pwn/Blinded.md](./Pwn/Blinded.md) | (ยังไม่พบ) |

---

## 🧠 เทคนิคที่ได้จากชุดนี้

| เทคนิค | ใช้ใน |
|---|---|
| Covenant C2 protocol reverse (RSA + AES-CBC + HMAC) | Acknowledge the Corn |
| Process memory forensics (BCRYPT_RSAKEY_BLOB) | Acknowledge the Corn |
| Keylogger log reconstruction | Acknowledge the Corn |
| Affine cipher over GF(2) + Gaussian elimination | Always Has Been |
| Linear cryptanalysis ของ SPN ที่ key = plaintext | Always Has Been |
| Static analysis binary (strings + array of ints) | Haunted Houseparty |
| Flask session/role escalation + admin bot + path traversal | Artificial University |
| gRPC `UpdateService` + `eval` (prototype pollution) | Artificial University |
| Heap tcache manipulation ด้วย single-byte write | Blinded |

---

## 💡 บทเรียนรวม

1. **Protocol reverse engineering** (Covenant) ต้องเข้าใจ handshake 3 stage: setup key → RSA → session key
2. **Cipher ที่ "linear เกินไป"** พังได้ด้วยพีชคณิตเชิงเส้น ไม่ต้อง brute force
3. **Flag ที่ฝังใน binary** ไม่จำเป็นต้อง reverse ยาก ขอแค่สังเกต array ของ int
4. **Bot/SSRF chain** เป็นแพทเทิร์นมาตรฐานของโจทย์ web ยุคใหม่
5. **Heap exploitation** ที่มีแค่ single-byte write ต้องใช้ tcache poisoning เป็นตัวขยายพลัง
