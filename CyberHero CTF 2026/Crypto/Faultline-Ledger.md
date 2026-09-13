# Faultline Ledger (Crypto, 300pts)

**Flag:** `CYBERHEROCTF{faultline_df6a28dda3785d4001214196}`
**สถานะ:** ✅ ยืนยันกับระบบแล้ว
**หมวดย่อย:** Cryptography — RSA Fault Attack + SHA-256 Length Extension + AES-CTR Keystream Reuse

---

## 📜 โจทย์

มีบริการ signing service ชื่อ **Faultline Ledger v3.2.0** ให้ flag มาแบบ **แบ่งเป็น 3 segment
segment ละ 8 ตัวอักษร** ซึ่งได้มาจากการโจมตี 3 ชั้นต่อเนื่องกัน:

```python
FLAG_KEYWORD         = "CYBERHEROCTF"
FLAG_BODY_PREFIX     = "faultline_"
FLAG_TOKEN_LENGTH    = 24        # hex
FLAG_SEGMENT_LENGTH  = 8
FLAG_SEGMENT_COUNT   = 3
```

endpoint ที่ให้มา:

| Method | Path | หน้าที่ |
|---|---|---|
| GET | `/` | หน้าแรก |
| GET | `/public` | public key (PEM) + sealed manifest |
| GET | `/healthz` | health (เฉพาะ loopback) |
| POST | `/sign` | ขอ signature |
| POST | `/ledger` | ยืนยันสิทธิ์ด้วย ledger token |
| POST | `/archive` | ถอด sealed archive |

ข้อจำกัดที่ต้องระวัง: `MAX_SIGNATURES = 64`, `SIGNATURE_WINDOW_SECONDS = 900`,
rate limit `90 requests / 60 วินาที`

---

## 🧩 Stage 1 — RSA-CRT Fault Attack

### จุดที่พัง

ฟังก์ชัน signing ใช้ CRT เพื่อความเร็ว แต่มี "fault" แทรกเป็นระยะ:

```python
def crt_sign_with_occasional_fault(message, sequence):
    sp = pow(representative, D % (P - 1), P)
    sq = pow(representative, D % (Q - 1), Q)
    if is_fault_sequence(sequence):      # ← จงใจให้ผลลัพธ์ผิด
        sp = (sp + delta) % P
    correction = ((sp - sq) * pow(Q, -1, P)) % P
    return (sq + Q * correction) % N, faulted
```

### หลักการ (Bellcore / Boneh–DeMillo–Lipton)

สมมติได้ signature ที่ผิด `s'` มาจากการคำนวณ modulo p ผิด แต่ modulo q ถูก

ถ้า `s'^e ≡ m (mod q)` แต่ `s'^e ≢ m (mod p)` แล้ว

```text
gcd( (s'^e − m) mod N , N ) = q     ← แยกตัวประกอบ N ได้ทันที
```

### ลงมือ

```python
from math import gcd
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_OAEP

N, e = pub.n, pub.e

for seq in range(1, 200):                 # ยิง /sign ไปเรื่อย ๆ (มี quota 64 ครั้ง/900s)
    sig = sign(msg, seq)                  # ถ้าลำดับนี้เป็นลำดับที่ fault จะได้ค่าเพี้ยน
    g = gcd(pow(sig, e, N) - m, N)
    if 1 < g < N:
        q = N // q_factor_candidate       # g คือตัวประกอบตัวหนึ่ง
        p = N // g
        phi = (p - 1) * (q - 1)
        d = pow(e, -1, phi)
        priv = RSA.construct((N, e, d, p, q))
        break
```

> **เคล็ดลับ:** ตัวที่ fault จะเกิดเฉพาะบาง sequence เท่านั้น (ในเครื่องจริงคือ
> `FAULT_INTERVAL = 13`, `FAULT_OFFSET = 11` → fault ตอน `seq % 13 == 11`)
> ถ้ายิงแล้ว `gcd` ได้ 1 หรือ N ให้เปลี่ยน sequence แล้วยิงใหม่

### ผลลัพธ์ของ Stage 1

เมื่อได้ private key แล้วถอด `sealed_manifest` (RSA-OAEP) จะได้ของ 4 อย่าง:

```text
- segment ที่ 1 ของ flag
- ledger path (ใช้ต่อใน Stage 2)
- ledger token
- secret length = 32
```

---

## 🔗 Stage 2 — SHA-256 Length Extension

### จุดที่พัง

ledger token สร้างจาก **prefix-MAC** แบบบ้าน ๆ:

```python
def ledger_token(path):
    return hashlib.sha256(LEDGER_SECRET + path).hexdigest()
```

การแฮชแบบ `H(secret ‖ message)` ปลอดภัยไม่พอ เพราะ SHA-256 เป็น Merkle–Damgård:
ถ้ารู้ `H(secret ‖ path)` และ **ความยาวของ secret** เราสามารถต่อท้ายข้อความได้
โดยไม่ต้องรู้ secret

### ลงมือ

ติดตั้ง `hashpumpy`:

```bash
pip install hashpumpy
```

```python
import hashpumpy

# secret_len = 32 (ได้จาก sealed manifest)
# orig_len   = len(path) = 53
new_hash, forged_data = hashpumpy.hashpump(
    known_hash,               # ledger token ที่ได้จาก Stage 1
    original_path,            # path เดิม (53 ตัวอักษร)
    appended,                 # path ที่ต้องการต่อท้าย เช่น "/sealed-archive"
    32                        # ← ความยาว secret ที่รู้มา
)
```

จากนั้นยิง:

```http
POST /ledger
{ "path": "<forged_data>", "token": "<new_hash>" }
```

### ผลลัพธ์ของ Stage 2

```text
- segment ที่ 2 ของ flag
- archive nonce
- archive record (ciphertext)
```

---

## 🔁 Stage 3 — AES-CTR Keystream Reuse

### จุดที่พัง

archive ถูกเข้ารหัสด้วย **key และ nonce คงที่** และ counter เริ่มที่ 0 ทุกครั้ง:

```python
def archive_keystream_encrypt(plaintext):
    cipher = Cipher(algorithms.AES(ARCHIVE_KEY), modes.CTR(ARCHIVE_NONCE))
    return cipher.encryptor().update(plaintext) + cipher.encryptor().finalize()
```

ในโหมด CTR:

```text
C = P ⊕ keystream
```

ถ้า keystream ถูกใช้ซ้ำ (key+nonce เดิม) แค่รู้ plaintext หนึ่งข้อความ ก็ได้ keystream ทั้งเส้น

### ลงมือ

```python
known_note = b"A" * 192                     # ข้อความที่เราเลือกเองได้ (MAX_NOTE_BYTES = 192)

ct_known    = post_archive(known_note)      # ciphertext ของข้อความที่เรารู้
keystream   = xor(ct_known, known_note)     # ได้ keystream
plaintext   = xor(archive_record, keystream[:len(archive_record)])   # ถอดอันที่ต้องการ
```

### ผลลัพธ์ของ Stage 3

```text
- segment ที่ 3 ของ flag
```

---

## 🏁 ประกอบ flag

```text
CYBERHEROCTF{faultline_  <seg1><seg2><seg3>  }
              ^prefix    ^24 hex (3 × 8 chars)
```

```text
CYBERHEROCTF{faultline_df6a28dda3785d4001214196}
```

### วิธีตรวจสอบแบบที่สอง (independent verification)

โจทย์มีสูตร flag อยู่ตรง ๆ ในซอร์ส:

```python
def flag_token(instance_id=None, secret=None):
    message = f"{CHALLENGE_ID}:instance:{instance_id}".encode()
    return hmac.new(secret, message, hashlib.sha256).hexdigest()[:24]
```

ข้อมูลจริงอยู่บนดิสก์ของ VM (forensics offline):

```bash
cat /etc/faultline-ledger.env
# MASTER_SECRET=f81be03cded377cf8accc4172c6d9e4c1346efc43b371d67542cfe09e1469708
# PORT=8000
# DATA_DIR=/var/lib/faultline-ledger
# MAX_SIGNATURES=64
# MAX_NOTE_BYTES=192
# TRUST_PROXY=false

cat /var/lib/faultline-ledger/instance_id
# 8795cb65-8a02-4a8f-8a65-1f495c4e0ba0
```

```python
import hmac, hashlib

SECRET = "f81be03cded377cf8accc4172c6d9e4c1346efc43b371d67542cfe09e1469708"
IID    = "8795cb65-8a02-4a8f-8a65-1f495c4e0ba0"

tok = hmac.new(
    SECRET.encode(),          # ← สำคัญมาก: ใช้ "ข้อความ hex" เป็น key ไม่ใช่ bytes.fromhex()
    f"faultline-ledger-v3.2.0:instance:{IID}".encode(),
    hashlib.sha256
).hexdigest()[:24]

print(tok)   # df6a28dda3785d4001214196  ← ตรงกับ flag ที่ระบบตอบถูก
```

> **จุดที่คนพลาดกันเยอะ:** ค่าใน env เป็น hex 64 ตัวอักษร แต่โค้ดอ่านมาเป็น *สตริง*
> แล้ว `.encode()` ตรง ๆ (`hmac.new(secret.encode(), ...)`) ไม่ได้ทำ `bytes.fromhex()`
> ถ้าเผลอ decode hex ก่อน จะได้ `8ad65e683bab29583a5f8490` ซึ่ง **ผิด**

ถ้าลองผิดแบบ decode hex ก่อน จะได้ค่าที่ไม่ตรง — เป็นกับดักที่ทำให้ต้องกลับไปทำ attack chain จริง
(ซึ่งเป็นทางที่ออกแบบไว้) แทนการลัดด้วยการอ่านไฟล์บนดิสก์

---

## 🧠 หลักการที่ต้องเข้าใจ

| จุดพัง | สาเหตุราก | วิธีป้องกัน |
|---|---|---|
| CRT fault | ไม่ verify signature ก่อนส่งออก | ตรวจ `s^e mod N == m` ก่อนคืนค่า (verify-before-release) |
| Length extension | ใช้ `H(secret‖msg)` เป็น MAC | ใช้ HMAC, หรือ SHA-256 แบบ keyed (เป็น Merkle–Damgård ทั้งคู่แต่ HMAC กันได้) |
| Keystream reuse | key+nonce คงที่, counter เริ่ม 0 | สุ่ม nonce ต่อข้อความ + ห้ามใช้ key ซ้ำในโหมด stream |

## ✅ สรุป

โจทย์นี้สอนว่า **cryptographic primitive ที่ดี ยังพังได้ถ้าใช้ผิดวิธี**:

1. RSA ที่ปลอดภัยก็พังได้ถ้ามี fault แล้วไม่ verify
2. SHA-256 ที่ปลอดภัยก็พังได้ถ้าใช้เป็น prefix-MAC
3. AES-CTR ที่ปลอดภัยก็พังได้ถ้าใช้ keystream ซ้ำ

และ flag ก็ถูกแบ่งให้เก็บทีละชั้น เพื่อบังคับให้ผู้เล่นโจมตี **ครบทั้ง 3 ชั้น**
