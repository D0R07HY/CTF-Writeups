# Always Has Been (Crypto, Medium)

**Flag:** `HTB{w41T...it'5_4ll_L1n34r?????}`
**สถานะ:** ✅ แก้ได้
**หมวดย่อย:** Cryptography — Linear/Affine Cryptanalysis (GF(2)), Gaussian Elimination

---

## 📜 โจทย์

ให้ไฟล์ 2 อย่าง:

| ไฟล์ | เนื้อหา |
|---|---|
| `securehash.py` | ซอร์สโค้ด hash function ที่เขียนเอง |
| `output.txt` | ค่า hash ของ flag (hex 64 ตัว) |

```text
output.txt = 61b5649e894a15a053276c0dc828ee64ec2336f809e2dd7d2912c61c8ef02c26
```

ต้องย้อนกลับจาก hash ไปหา flag (flag ยาว 32 byte)

---

## 🔍 วิเคราะห์โค้ด

### โครงสร้างที่สำคัญ

```python
BLOCK_SIZE = 32

class SecureCipher:
    def __init__(self, key):
        self.sbox = [KEY_SBOX[i] ^ key[0] for i in range(256)]   # ← S-box ขึ้นกับ key!
        self.key = key

    def substitute(self, data):   return bytes([self.sbox[b] for b in data])
    def permute(self, data):      ...    # bit permutation 256 bits ตาม PBOX

    def encrypt(self, data):
        for _ in range(100):   # "No linear or differential cryptanalysis here!"
            block = self.substitute(block)
            block = self.permute(block)
            block = xor(block, self.key)
        return block

class SecureHash:
    def __init__(self, data=None):
        self.state = b"\x00" * 32
        if data: self.update(data)

    def update(self, data):
        data = self.pad(data)              # flag ยาว 32 = BLOCK_SIZE พอดี → ไม่มี padding
        for block in blocks(data):
            c = SecureCipher(block)        # ← key = block (key = plaintext!)
            self.state = xor(self.state, c.encrypt(block))
```

### สังเกต 3 จุดที่ทำให้พัง

1. **flag ยาว 32 byte = 1 block พอดี** → `pad()` คืนค่าเดิม (ไม่เติมอะไร)
   และ `state` เริ่มจาก 0 ทั้งหมด

   ```text
   output = 0 ⊕ E_x(x)  =  f(x)        โดยที่ x = flag
   ```

2. **key = plaintext = x** → ทั้งกระบวนการกลายเป็นฟังก์ชันของ x ตัวเดียว

3. **S-box เป็น affine** (นี่คือจุดตาย):

   ```python
   KEY_SBOX[v] == M(v) ^ 170        # 170 = 0xAA
   ```

   โดยที่ `M` เป็น **ฟังก์ชันเชิงเส้นบน GF(2)** (พิสูจน์ได้: `M(a ^ b) == M(a) ^ M(b)` ทุกคู่)

   ```text
   M(0x00)=0x00  M(0x01)=0xf3  M(0x02)=0xfb  M(0x04)=0xeb  M(0x08)=0xcb
   M(0x10)=0x8b  M(0x20)=0x0b  M(0x40)=0x16  M(0x80)=0x2c
   ```

   → `substitute` เป็น affine, `permute` เป็น bit-permutation (เชิงเส้น),
     `xor` กับ key ก็เชิงเส้น → **ทั้ง 100 รอบเป็น affine**

> คอมเมนต์ในโค้ดเขียนว่า *"No linear or differential cryptanalysis here!"*
> — เป็น **กับดัก** ตรงกันข้ามกับความจริงทั้งหมด

### สรุปทางคณิตศาสตร์

เพราะทุก operation เป็น affine และ `key = plaintext = x`

```text
f(x) = A·x ⊕ b        (over GF(2), x คือ 256 bit)
```

ถ้าหา `A` (เมทริกซ์ 256×256 บน GF(2)) และ `b` ได้ ก็แก้สมการ `A·x = target ⊕ b` ด้วย
**Gaussian elimination** ได้เลย

---

## 🛠️ วิธีแก้

### 1. หา constant term `b`

```python
b = f(b"\x00" * 32)      # ป้อน input ที่เป็นศูนย์ทั้งหมด
```

### 2. หาเมทริกซ์ `A` ทีละคอลัมน์

ใช้ **basis vectors** ของ GF(2)^256 (คือ input ที่มีบิตเดียวเป็น 1):

```python
def f(x: bytes) -> bytes:
    return SecureHash(x).digest()

b = f(bytes(32))

A_cols = []
for i in range(256):                  # 256 บิต
    e = bytearray(32)
    e[i // 8] = 1 << (7 - (i % 8))
    col = xor(f(bytes(e)), b)         # f(e_i) ⊕ f(0) = คอลัมน์ที่ i ของ A
    A_cols.append(col)
```

> ต้นทุน: 257 ครั้ง แต่ละครั้งรัน 100 รอบ × 32 byte — เร็วมาก (< 1 วินาที)

### 3. ตรวจสอบว่า affine จริง

ถ้า `f` เป็น affine จริง ต้องได้:

```python
assert f(xor(a, b)) == xor(xor(f(a), f(b)), f(bytes(32)))   # สุ่ม a, b มาทดสอบ
```

ถ้าผ่าน → ใช้วิธีนี้ได้แน่นอน

### 4. แก้ระบบสมการ

ประกอบเมทริกซ์ (256 สมการ × 256 ตัวแปร) แล้วทำ Gaussian elimination บน GF(2):

```python
# rows[i] = สมการที่ i : [A_row_i | target_bit_i]
target = bytes.fromhex(open("output.txt").read())
rhs = xor(target, b)

# สร้าง augmented matrix 256 x 257
M = [[bit(A_cols[c], r) for c in range(256)] + [bit(rhs, r)] for r in range(256)]

# Gaussian elimination over GF(2) → ได้ x (หรือ nullspace ถ้า rank < 256)
x = gauss_solve_gf2(M)
print(x)          # b"HTB{w41T...it'5_4ll_L1n34r?????}"
```

### 5. ผลลัพธ์จริง: rank = 255/256

เมื่อรันสคริปต์จริงได้ผลดังนี้:

```text
[+] KEY_SBOX is affine: KEY_SBOX[v] = M(v) ^ 0xaa
[+] f(x) = E_x(x) is affine over GF(2)
[+] rank of system: 255/256
[+] candidate flag bytes: b"HTB{w41T...it'5_4ll_L1n34r?????}"
[+] verify f(x) == target: True
[+] flag: HTB{w41T...it'5_4ll_L1n34r?????}
[+] free columns: [249]
    kernel bit 249: alt = b'\x82"\xbc\xb6...'   f(alt)==target: True
```

เมทริกซ์มี rank **255 จาก 256** → มี **1 free variable** (คอลัมน์ 249)
จึงมีคำตอบ 2 ตัวที่ให้ hash ตรงกัน:

1. `HTB{w41T...it'5_4ll_L1n34r?????}` ← ASCII อ่านออก (คำตอบที่ต้องการ)
2. อีกตัวที่มี byte `\x82\x22\xbc...` (non-printable ทั้งก้อน) ← second preimage

**วิธีเลือกคำตอบ:** กรองเอาเฉพาะตัวที่ทุก byte อยู่ในช่วง printable ASCII
และตรงรูปแบบ `HTB{...}` — เหลือตัวเดียว

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Affine over GF(2)** | ฟังก์ชันที่เขียนเป็น `A·x ⊕ b` ได้ — ถ้าทุก operation (S-box, permutation, XOR) เป็น affine ทั้งหมด |
| **S-box ที่สร้างจาก XOR กับ key** | `S[i] = KEY_SBOX[i] ^ key[0]` ทำให้ S-box ไม่ผสมข้อมูลแบบไม่เชิงเส้น (non-linear) เลย |
| **Known-plaintext basis recovery** | ใช้ `f(0)` + `f(e_i)` เพื่อสร้างเมทริกซ์ของฟังก์ชัน affine ภายใน 257 queries |
| **Gaussian elimination บน GF(2)** | ใช้แก้ระบบสมการบิต — เป็นเครื่องมือพื้นฐานที่สุดของ crypto CTF |
| **กับดักในคอมเมนต์** | คอมเมนต์ในซอร์สบอกว่าปลอดภัยจาก linear cryptanalysis — แต่กลับกันเลย |

## ✅ สรุป

โจทย์นี้แสดงให้เห็นว่า **การไม่ผสม non-linearity (confusion) ที่เพียงพอ ทำให้ cipher ทั้งตัว
กลายเป็นสมการเชิงเส้น** ที่แก้ได้ด้วยพีชคณิต ไม่ต้อง brute force

- S-box ต้องเป็น non-linear (เช่น AES S-box ที่สร้างจาก inverse ใน GF(2^8) + affine)
- การที่ key = plaintext ทำให้กำลังของ cipher ลดลงอย่างรุนแรง (กลายเป็น fixed function)
- 100 รอบไม่ช่วยอะไร ถ้าทั้งหมดเป็น affine

```text
HTB{w41T...it'5_4ll_L1n34r?????}
```

(*"wait… it's all linear?"* — เป็น meme ที่ล้อว่าทุกอย่าง linear จริง ๆ)
