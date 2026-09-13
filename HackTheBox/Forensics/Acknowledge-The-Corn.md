# Acknowledge the Corn (Forensics, Hard)

**Flag:** `HTB{C2s_4r3_n0t_4lw4ys_4_s4f3_c0mmun1c4t10n_ch4nn3l}`
**สถานะ:** ✅ แก้ได้
**หมวดย่อย:** Forensics — Network Forensics, Memory Forensics, C2 Protocol Reverse Engineering

---

## 📜 โจทย์

ให้ไฟล์ 2 อย่างจากเครื่องเหยื่อที่ถูก APT โจมตี:

| ไฟล์ | ขนาด | ความหมาย |
|---|---|---|
| `capture.pcap` | ~2.9 MB | network capture |
| `powershell.dmp` | ~302 MB | memory dump ของ process `powershell` |

โจทย์ถามว่า **แฮกเกอร์เจาะเข้าเครือข่ายได้หรือไม่ และขโมยอะไรออกไป**

---

## 🔍 ขั้นตอนการแก้

### 1. ดู network capture ก่อน

```bash
tshark -r capture.pcap -q -z conv,tcp
```

พบว่ามี 2 IP คุยกัน:

```text
192.168.1.8   → เครื่องเหยื่อ (HTBBANK-WS01)
192.168.1.11  → C2 server (IIS)
```

เส้นทางที่น่าสนใจ:

```text
GET  /byp.ps1            → PowerShell script (AMSI bypass)
GET  /dwn.ps1            → downloader (โหลด .NET stager)
POST /en-us/test.html    → Covenant C2 traffic (เข้ารหัส)
POST /en-us/docs.html
POST /en-us/index.html
```

### 2. แกะ PowerShell stager

`dwn.ps1` เป็น dropper รูปแบบ:

```powershell
sv o (New-Object IO.MemoryStream);
sv d (New-Object IO.Compression.DeflateStream([IO.MemoryStream][Convert]::FromBase64String('...'), Decompress));
...
[Reflection.Assembly]::Load(...)
```

เอา base64 string ไป decode แล้ว decompress แบบ **raw deflate** (`wbits=-15`):

```python
import base64, zlib
data = base64.b64decode(b64)
dll  = zlib.decompress(data, -15)
open("stage.bin", "wb").write(dll)
assert dll[:2] == b"MZ"
```

เปิดด้วย dnSpy → เจอ namespace/class **`GruntStager`**, method `ExecuteStager`
→ ยืนยันว่าเป็น **Covenant C2** (open-source offensive framework)

### 3. เข้าใจ crypto ของ Covenant

จาก `GruntStager.cs` ของ Covenant, handshake ทำงานแบบนี้:

```text
Stage 0 (grunt → server):
  - มี shared secret (SetupKeyBytes) ฝังมาใน stager
  - สุ่ม GUID 10 hex + สร้าง RSA-2048 keypair
  - ส่ง RSA public key (XML) โดย AES-CBC ด้วย SetupKeyBytes + HMAC-SHA256

Stage 0 response (server → grunt):
  - server สุ่ม Session key
  - RSA-encrypt session key ด้วย public key ของ grunt
  - แล้ว AES-encrypt ซ้ำด้วย SetupKeyBytes → ส่งกลับ
  - ฝั่ง grunt: AES-decrypt (setup) → RSA-decrypt (private) → ได้ Session key

หลังจากนั้น:
  - tasking (Type 1) และ results (Type 0/2) เข้ารหัสด้วย Session key
  - HMAC key = AES key ตัวเดียวกัน
```

รูปแบบข้อความ:

```json
{"GUID":"<10 หรือ 20 hex>","Type":<0|1|2>,"Meta":"<task id>",
 "IV":"<b64>","EncryptedMessage":"<b64>","HMAC":"<b64>"}
```

ใน pcap body ของ POST เป็น `i=<id>&data=<base64 ของ JSON>&session=<id>` → ต้อง base64-decode ก่อน

### 4. ดึง key จาก memory dump

- **Setup AES key** อยู่ใน stager ที่ decompile ได้ (ค่า `{{REPLACE_GRUNT_SHARED_SECRET_PASSWORD}}`):

```text
e+MPqFZXA52Kx1xuTPTK6M/HtJkjq/0dfBJUsSJfzQw=
```

- **RSA private key** อยู่ใน `powershell.dmp` ในรูปแบบ **BCRYPT_RSAKEY_BLOB** (magic `RSA2`)
  ที่ file offset `0x45def20`:

```text
RSA2 | BitLength=2048 | cbPublicExp=3 | cbModulus=256 | cbPrime1=128 | cbPrime2=128
PublicExponent (3) | Modulus (256) | Prime1 (128) | Prime2 (128) | ...
```

หาได้ด้วยการค้น byte `52 53 41 32` ("RSA2") ใน dump แล้ว parse ตามโครงสร้าง
(ยืนยันด้วย `n == p * q`)

### 5. ถอด session key

```text
AES-CBC-decrypt(setup_key, IV, EncryptedMessage)
    → RSA-OAEP-decrypt(private_key)
        → Session key = k6Xh4oOEESrjNtykri43F7NzjND8tPGCTsMXTSiZWyBI=
```

(ตรวจ HMAC-SHA256 ของแต่ละข้อความก่อน decrypt เสมอ — ถ้า key ผิด HMAC จะไม่ผ่าน)

### 6. ถอดข้อความทั้งหมด → เจอหลักฐาน

```json
{"integrity":3,"process":"powershell","userDomainName":"HTBBANK-WS01",
 "userName":"bank_administrator","hostname":"HTBBANK-WS01",
 "operatingSystem":"Microsoft Windows NT 10.0.19042.0"}
{"status":"3","output":"HTBBANK-WS01\\bank_administrator"}     ← ผล whoami
{"status":"2","output":"Starting keylogger for 0 seconds.\r\n"}
{"status":"2","output":"...admlshiftkeyLSHIFTKEY...HTB{C2s..." } ← keylogger output
```

👉 **ตอบคำถามโจทย์: เจาะสำเร็จ** — แฮกเกอร์ได้ shell ในนาม
`HTBBANK-WS01\bank_administrator` และรัน keylogger ขโมย keystroke ออกไป

### 7. Reconstruct flag จาก keylogger log

keylogger จะมีชื่อปุ่มพิเศษแทรกอยู่ในข้อความ (`lshiftkey` / `LSHIFTKEY`)

```text
admlshiftkeyLSHIFTKEYLSHIFTKEYLSHIFTKEYLSHIFTKEYLSHIFTKEYLSHIFTKEY_banklshiftkeyHTB{C2s...
```

ให้เอาทุก fragment ต่อกันตามเวลา แล้วลบ `lshiftkey` (case-insensitive) ออก:

```python
import re
clean = re.sub(r'lshiftkey', '', raw, flags=re.I)
print(re.search(r'HTB\{[^}]+\}', clean).group(0))
```

ผลลัพธ์:

```text
adm_bankHTB{C2s_4r3_n0t_4lw4ys_4_s4f3_c0mmun1c4t10n_ch4nn3l}
```

---

## 🛠️ Solve Script (สรุปหลักการ)

```python
import base64, json, re, struct, socket, hashlib, hmac, dpkt
from Crypto.Cipher import AES, PKCS1_OAEP
from Crypto.PublicKey import RSA
from Crypto.Util.Padding import unpad

PCAP, DUMP = "capture.pcap", "powershell.dmp"
SETUP_KEY = base64.b64decode("e+MPqFZXA52Kx1xuTPTK6M/HtJkjq/0dfBJUsSJfzQw=")
MSG_RE = re.compile(rb'\{"GUID":"[0-9a-f]{10,20}","Type":\d+,"Meta":"[^"]*",'
                    rb'"IV":"[^"]*","EncryptedMessage":"[^"]*","HMAC":"[^"]*"\}')

def hmac_ok(key, ct, h):
    return hmac.compare_digest(hmac.new(key, ct, hashlib.sha256).digest(), base64.b64decode(h))

def aes_dec(key, iv, data):
    try:
        return unpad(AES.new(key, AES.MODE_CBC, base64.b64decode(iv))
                     .decrypt(base64.b64decode(data)), 16)
    except Exception:
        return None

def rsa_keys(dump_bytes):
    """หา BCRYPT_RSAKEY_BLOB ทั้งหมดใน memory dump"""
    for m in re.finditer(rb'RSA2', dump_bytes):
        o = m.start()
        bitlen, cb_e, cb_n, cb_p1, cb_p2 = struct.unpack('<IIIII', dump_bytes[o+4:o+24])
        if bitlen not in (1024, 2048, 4096) or cb_p1 != cb_p2:
            continue
        cur = o + 24
        e  = int.from_bytes(dump_bytes[cur:cur+cb_e], 'big');  cur += cb_e
        n  = int.from_bytes(dump_bytes[cur:cur+cb_n], 'big');  cur += cb_n
        p  = int.from_bytes(dump_bytes[cur:cur+cb_p1], 'big'); cur += cb_p1
        q  = int.from_bytes(dump_bytes[cur:cur+cb_p2], 'big'); cur += cb_p2
        if n == p * q:                     # ยืนยันว่า parse ถูก
            yield RSA.construct((n, e, pow(e, -1, (p-1)*(q-1)), p, q))

# ... (ส่วน reassemble TCP stream และ decrypt ทำตามขั้นตอนข้อ 3-6)
```

> สคริปต์เต็มอยู่ในโฟลเดอร์โจทย์ (`solve.py`) — รันแล้ว print flag ได้ทันที

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Covenant handshake** | 3 ชั้น: shared secret → RSA public key → session key (ถ้าเข้าใจก็ถอดได้ทั้งหมด) |
| **HMAC ก่อน decrypt** | ลำดับที่ถูกคือ verify HMAC → decrypt (ประหยัดเวลาและยืนยัน key) |
| **BCRYPT_RSAKEY_BLOB** | โครงสร้าง key ของ Windows ใน memory (`RSA2` + little-endian lengths + big-endian values) |
| **Keylogger artifacts** | ปุ่ม modifier (`lshiftkey`) โผล่มาใน log เป็นข้อความ — เป็นทั้งสัญญาณรบกวนและเบาะแส |
| **TCP stream reassembly** | C2 traffic ถูกแบ่งหลาย packet ต้องต่อ stream ตาม (src,sport,dst,dport) ตามเวลาก่อน parse |

## ✅ สรุป

โจทย์นี้จำลองการโจมตีด้วย **Covenant C2** จริง และให้ทั้ง pcap + memory dump
เพื่อบังคับให้ผู้เล่นทำ **protocol reverse engineering + memory forensics ครบวงจร**

คำตอบคือ **"เจาะได้"** และ flag มาจาก keylogger log ที่ถูกส่งกลับไปยัง C2 server:

```text
HTB{C2s_4r3_n0t_4lw4ys_4_s4f3_c0mmun1c4t10n_ch4nn3l}
```

(อ่านว่า *"C2s are not always a safe communication channel"* — สื่อว่าแฮกเกอร์
ใช้ C2 ที่ไม่ปลอดภัยพอ ทำให้เราถอดการสื่อสารได้)
