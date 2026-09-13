# Blinded (Pwn, Hard)

**Flag:** ❌ ยังไม่พบ (ในไฟล์ที่แจกมาเป็น placeholder)
**สถานะ:** 🔴 วิเคราะห์ primitive ครบแล้ว ยังไม่ได้เขียน exploit สำเร็จ
**หมวดย่อย:** Pwn — Heap Exploitation (tcache poisoning), Single-byte Write Primitive, glibc 2.35

---

## 📜 โจทย์

ให้ binary + libc มา (Ubuntu 22.04 / glibc 2.35):

```text
challenge/blinded        ← binary
challenge/libc.so.6      ← glibc 2.35
challenge/ld-2.35.so     ← dynamic loader
challenge/server.py      ← harness
```

กติกาของ harness (`server.py`):

```python
PWD_CHARSET  = string.ascii_letters + string.digits
PWD_LENGTH   = 32
ITERATIONS   = 32

def test_payload(payload):
    for i in range(ITERATIONS):
        password = "".join(secrets.choice(PWD_CHARSET) for _ in range(PWD_LENGTH))
        open("password", "w+").write(password)      # สุ่มรหัสผ่านใหม่ทุกครั้ง
        os.chmod("password", 0o644)
        out, err = run(payload, user="ctf")         # ← รัน payload ของเรา 32 ครั้ง
        if out.strip() != password:
            return i, out, err                      # ผิด = บอกว่า fail ที่รอบไหน
    return True                                     # ถูกทั้ง 32 รอบ = ได้ flag
```

👉 ต้องเขียน payload ที่ **อ่านไฟล์ `password` แล้ว print เนื้อหาออกมาให้ตรงเป๊ะ**
และต้องทำสำเร็จ **32 ครั้งติดกัน** (เพราะรหัสเปลี่ยนทุกครั้ง)

---

## 🔍 วิเคราะห์ primitive

### ตัวโปรแกรม (`blinded.c`)

```c
void setup() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    free(malloc(0x800));            // ← ทิ้ง chunk ขนาด 0x800 ไว้ให้ (grooming aid)
}

size_t vuln() {
    size_t size; int i; char c;
    if (scanf("%zu %d %hhd", &size, &i, &c) < 3 || size > 0x3e8 || !(0 <= i < size))
        _exit(EXIT_FAILURE);
    char* ptr = malloc(size);
    ptr[i] = c;                     // ← เขียนได้ 1 byte ต่อ 1 รอบ
    free(ptr);
    return size;
}

int main(int argc, char* argv[]) {
    setup();
    ssize_t rem = (argc < 2) ? LONG_MAX : atoi(argv[1]);   // MAX_MEM = 0x1000
    while (rem > 0) rem -= vuln();
}
```

### สิ่งที่เราทำได้

| ทำได้ | ทำไม่ได้ |
|---|---|
| `malloc(size)` เมื่อ `size ≤ 0x3e8` (1000) | ไม่มีการอ่าน input อื่นนอกจากตัวเลข 3 ตัว |
| เขียน **1 byte** ที่ offset `i` (`0 ≤ i < size`) | เขียนนอกขอบเขตของ chunk ที่ malloc ให้ไม่ได้ |
| `free(ptr)` | ไม่มี buffer overflow / format string |
| จำนวนรอบ = จนกว่า **budget รวม 4096 byte** จะหมด | ไม่มีช่องให้ leak memory ตรง ๆ |

**ข้อสังเกตสำคัญ:** จำนวนรอบ **ไม่จำกัดที่ 32** — จำกัดที่ **ผลรวมของ `size` ≤ 0x1000 (4096)**
ดังนั้นถ้าใช้ `size` เล็ก ๆ (เช่น 1–100) จะได้ **หลายพันรอบ** → มี write primitive หลายพันครั้ง

### ขยายพลังด้วย tcache

เพราะ `malloc(size)` แล้ว `free(ptr)` ทันที → **รอบถัดไป `malloc(size)` จะได้ chunk เดิมกลับมา**
(tcache เป็น LIFO) ทำให้เรา:

1. **เขียน byte ที่ตำแหน่งใดก็ได้ภายใน chunk เดียวกัน หลายรอบ** → สร้าง string/โครงสร้างข้อมูลได้ทีละ byte
2. **poison `tcache->next`** เพื่อให้ `malloc` คืน pointer ที่ **address อะไรก็ได้** ที่เราต้องการ
   ( นี่คือหัวใจของ tcache poisoning )

> glibc 2.35 มี **safe-linking**: `next` ถูก XOR ด้วย `(chunk_addr >> 12)`
> → ต้องรู้ heap address ก่อน ถึงจะ poison ได้ถูกต้อง

### ทางได้ leak: option 2 "Get a glimpse of it"

```python
def get_proc_maps():
    p = spawn()
    with open(f"/proc/{p.pid}/maps", "r") as f:
        print(f.read())            # ← พิมพ์ memory map ทั้งหมดออกมา
    p.kill()
```

👉 ตัว harness เองมี endpoint ให้ **อ่าน `/proc/<pid>/maps`** ซึ่งให้:

- base ของ binary (ถ้า PIE → ได้ base)
- base ของ **heap**
- base ของ **libc**

ครบทั้ง 3 อย่างที่ต้องใช้ (leak ฟรี ไม่ต้องหา bug)

---

## 🎯 แนวทาง exploit ที่เป็นไปได้ (ยังไม่ได้ทดสอบ)

```text
[1] ใช้ option 2 ดึง /proc/pid/maps → heap base, binary base, libc base
[2] สร้าง string คำสั่งใน chunk ให้ครบ (เขียนทีละ byte ด้วย i ที่ต่างกัน)
         ต้องเป็นคำสั่งที่อ่านไฟล์แล้ว print เช่น "cat password"
         — ใช้ malloc/free ซ้ำ ๆ ที่ size เดิม จะได้ chunk เดิมกลับมา
[3] tcache poisoning:
         - เขียน next pointer ของ tcache entry (พร้อม safe-link mangle)
           ให้ชี้ไปที่ free@GOT (หรือ GOT ของฟังก์ชันอื่นที่ถูกเรียกใน loop)
         - malloc(size) ครั้งถัดไป → ได้ pointer = free@GOT
         - เขียน byte ทับ GOT ทีละ byte ให้ชี้ไปที่ system()
[4] ปล่อยให้ loop เรียก free(ptr) โดยที่ ptr = chunk ที่มี string คำสั่ง
         → free@GOT ชี้ไป system แล้ว → system("cat password")
[5] เนื้อหาไฟล์ password ถูก print ออก stdout
         → out.strip() == password → ผ่านทุก 32 รอบ → flag
```

เหตุผลที่ใช้ `free@GOT` → `system`:

- `vuln()` **เรียก `free(ptr)` ทุกรอบ** → เมื่อ GOT ถูกแทนที่ จะกลายเป็น `system(ptr)` ทันที
- `ptr` คือ chunk ที่เราคุมเนื้อหาได้ทั้งหมด → `system("cat password")` ทำงานทันที
- เราไม่ต้องอ่านเนื้อหาไฟล์ในโปรแกรมเอง — ใช้ shell แทน

ความท้าทายที่เหลือ:

| ความท้าทาย | รายละเอียด |
|---|---|
| **Safe-linking** | `next` ต้องถูก mangle ด้วย `(addr >> 12)` — แก้ได้เพราะ leak heap base มาแล้ว |
| **ขนาด GOT address** | `i < size ≤ 0x3e8` → ต้องออกแบบให้ offset ของ byte ที่ต้องเขียนอยู่ในช่วงที่ `i` ไปถึงได้ (อาจต้อง poison ให้ pointer ไปที่ `target - k`) |
| **budget 4096** | การ poison + เขียน GOT (6 byte) + เขียน string กิน byte รวมกัน ต้องวางแผนให้พอดี |
| **ASLR ทุกครั้ง** | รหัสผ่าน/process ใหม่ทุกรอบ → exploit ต้องได้ leak ใหม่ทุกรอบ |
| **32 รอบติดกัน** | ความน่าจะเป็นสูงมาก (ต้อง deterministic ไม่มี randomness ใน exploit) |

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Single-byte write primitive** | เขียนได้ทีละ 1 byte แต่ **หลายครั้ง** — พอจะสร้าง pointer/string ได้ |
| **tcache LIFO reuse** | `malloc/free` ที่ size เดิมซ้ำ ๆ = ได้ chunk เดิม → เขียนได้หลาย byte ในที่เดียว |
| **tcache poisoning** | เขียน `next` ของ tcache entry เพื่อให้ malloc คืน arbitrary pointer → arbitrary write |
| **Safe-linking (≥ 2.32)** | `next` ถูก XOR ด้วย `(chunk_addr >> 12)` — ต้อง leak heap ก่อน |
| **GOT overwrite** | ถ้า binary ไม่ได้ทำ FULL RELRO การเขียน GOT = เปลี่ยนปลายทางการเรียกฟังก์ชัน |
| **`free` → `system`** | classic: ทุก `free(ptr)` กลายเป็น `system(ptr)` ถ้า ptr ชี้ string คำสั่ง |
| **Harness ที่ให้ leak** | `/proc/self/maps` เป็น leak ที่ทรงพลังที่สุดและโจทย์มักเผลอให้ |

## ✅ สรุป

โจทย์นี้เกิดจากการนำ **primitive ที่ดูอ่อนมาก** (single-byte write ใน heap)
มาประกอบกับ **harness ที่ให้ memory map ฟรี** → กลายเป็นวงจร:

```text
leak (maps) → heap grooming → tcache poisoning → arbitrary write
            → GOT overwrite (free → system) → cat password → print
```

สถานะปัจจุบัน: **วิเคราะห์ primitive และวางแผน chain ได้ครบแล้ว**
แต่ยังไม่ได้เขียน exploit ที่ทดสอบผ่านครบ 32 รอบ

> หมายเหตุ: `flag.txt` ที่แจกมาเป็น `HTB{f4k3_fLaG_f0R_t3sT1nG!}` (placeholder)
> flag จริงจะได้จาก **server จริง** ผ่าน `choice == 1` เมื่อ payload ผ่านครบ 32 รอบ:

```python
if choice == 1:
    ...
    if out == True:
        print(FLAG)      # ← flag จริงอยู่ตรงนี้
        exit()
```
