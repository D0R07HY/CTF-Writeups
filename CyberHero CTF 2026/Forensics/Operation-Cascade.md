# Operation Cascade (Forensics, 300pts)

**Flag (ผู้สมัครหลัก):** `CYBERHEROCTF{Th1S_isA_S3cR@t_M3ssage}`
**สถานะ:** 🟡 พบ flag แล้ว (ยังไม่ได้ยืนยันกับระบบ)
**หมวดย่อย:** Forensics — Linux Memory Analysis + Disk Image Steganography (ASCII-Art QR)

---

## 📜 โจทย์

โจทย์ให้หลักฐาน 2 ชิ้นจากเครื่อง Linux ที่ถูกบุกรุก:

| ไฟล์ | ขนาด | ที่มา |
|---|---|---|
| `memorial_final.raw` | ~8.47 GB | memory dump (เก็บด้วย **AVML** — Microsoft AVML) |
| `disk.img.xz` | ~322 KB (xz) → ~2 GB | disk image (ext4) |

เป้าหมายคือหา **ความลับที่ผู้โจมตีซ่อนไว้ในเครื่อง**

---

## 🔍 ขั้นตอนการแก้

### 1. วิเคราะห์ Memory Dump

หาข้อมูลระบบพื้นฐานก่อน (ใช้ volatility3 / strings):

```bash
vol -f memorial_final.raw banners.Banners
vol -f memorial_final.raw linux.PsList
vol -f memorial_final.raw linux.Bash
```

ข้อมูลที่ได้:

```text
OS      : Ubuntu 24.04.4 LTS
Kernel  : 6.8.0-138-generic
Host    : ubuntu
Acquire : avml → /home/ubuntu/memorial_final.raw   (เก็บจาก /home/ubuntu/c2_server/)
```

ร่องรอยคำสั่งที่น่าสงสัย (จาก bash history / process args):

```text
sudo cp /etc/shadow /var/tmp/.cache/backup.dat
```

→ มีการ **คัดลอก /etc/shadow ไปซ่อนไว้** ที่ `/var/tmp/.cache/backup.dat`
(นี่คือ "ความลับ" ที่ต้องตามหา)

> ใน memory dump เอง **ไม่มี** `CYBERHEROCTF{...}` — ต้องไปหากันในดิสก์

### 2. แกะ Disk Image

```bash
xz -d disk.img.xz
file disk.img
# disk.img: DOS/MBR boot sector ... 
#   partition/offset: ext4 filesystem ที่ offset 0x100000
```

mount ด้วย offset:

```bash
sudo mount -o ro,loop,offset=0x100000 disk.img /mnt/disk
ls /mnt/disk
# /usr/local/lib/  ← จุดน่าสนใจ
cat /mnt/disk/etc/hostname
blkid   # UUID: 5592a90dccae460da7cb0d7a13a8e70a
```

ตรวจ `/var/tmp/.cache/`:

```bash
ls -la /mnt/disk/var/tmp/.cache/
# ว่างเปล่า → backup.dat ถูกลบไปแล้ว (เหลือแค่ร่องรอยใน jbd2 journal)
```

### 3. จุดไพล่ — ไฟล์ libsystemd.so ปลอม

ใน `/usr/local/lib/` มีไฟล์ชื่อ **`libsystemd.so`** ซึ่งตามปกติควรเป็น shared library จริง
แต่ขนาด/ชนิดไฟล์ไม่ถูกต้อง และเมื่อเปิดดูจะเจอ **ภาพ ASCII-ART**

```bash
file /mnt/disk/usr/local/lib/libsystemd.so
head -c 2000 /mnt/disk/usr/local/lib/libsystemd.so
```

ภาพที่ได้คือ **QR Code ที่วาดด้วยตัวอักษร ASCII** (มี drop shadow ด้วย):

```text
ขนาด: 62 คอลัมน์ × 30 บรรทัด
เนื้อหา: QR Code เวอร์ชัน 2 (25 × 25 module)
```

### 4. Reconstruct ASCII-Art เป็น QR ที่สแกนได้

นี่คือขั้นที่ต้องใช้เทคนิคหน่อย เพราะ ASCII-art มีความเพี้ยน:

1. **หา finder pattern** (ตา 3 มุมของ QR) ในภาพ ASCII เพื่อ **calibrate** ตำแหน่ง
2. **sample** ค่ากลางของแต่ละ module ด้วย `cv2.remap` (map จาก grid 25×25 กลับไปที่ภาพ ASCII)
3. **threshold** ด้วยวิธี Otsu (แยกดำ/ขาวอัตโนมัติ)
4. ได้ภาพ QR ขนาด 25×25 ที่สะอาด → decode

```python
import cv2, numpy as np
import zxingcpp

img = cv2.imread("qr_solved.png", cv2.IMREAD_GRAYSCALE)
_, bw = cv2.threshold(img, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)

results = zxingcpp.read_barcodes(bw)
for r in results:
    print(r.format, r.text)
```

ผลลัพธ์:

```text
Th1S_isA_S3cR@t_M3ssage
```

### 5. ประกอบ flag

```text
CYBERHEROCTF{Th1S_isA_S3cR@t_M3ssage}
```

---

## 🧩 เบาะแสที่ยังค้างอยู่ (สำหรับคนที่ทำต่อ)

ระหว่างวิเคราะห์ยังเจอประเด็นที่ยังไม่ได้ข้อยุติ:

1. **`/var/tmp/.cache/backup.dat` ถูกลบ** — เหลือแต่ metadata ใน jbd2 journal ของ ext4
   (ต้อง carve/journal replay ถึงจะได้เนื้อหา)
2. **Orphan inode 12** — ขนาด 520192 byte แต่เนื้อหาเป็นศูนย์ทั้งหมด
   ยกเว้น tail 8 byte ทุก ๆ 4KB — ลักษณะเหมือนไฟล์ที่ถูกเขียนทับ/zap (น่าจะเป็น decoy)
3. **UUID ของ ext4** `5592a90dccae460da7cb0d7a13a8e70a` — ใช้ยืนยันว่า disk image
   เป็นลูกเดียวกันกับที่ mount ใน memory dump

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Memory Forensics** | memory dump ให้ "ร่องรอยการกระทำ" (คำสั่ง, process, network) แต่ไม่จำเป็นต้องมีไฟล์ปลายทาง |
| **Cross-artifact correlation** | ต้องเอา memory (คำสั่ง `cp /etc/shadow`) มาประกอบกับ disk (หาไฟล์ที่ถูกย้าย) |
| **Loop mount ด้วย offset** | disk image ที่เป็น MBR ไม่สามารถ mount ทั้งไฟล์ได้ ต้องระบุ `offset=` ของ partition |
| **ASCII-art → QR** | ต้องมี calibration (finder pattern) + sampling ที่กึ่งกลาง module + threshold ที่เหมาะสม |
| **zxing-cpp** | ไลบรารี decode barcode/QR ที่ทนทานกว่า pyzbar ในภาพคุณภาพต่ำ |

## ✅ สรุป

"Operation Cascade" คือโจทย์ forensics ที่ผสม 3 ทักษะ:

```text
1. Memory analysis    → รู้ว่ามีการขโมย /etc/shadow และเซฟเป็น backup.dat
2. Disk forensics     → mount ext4 (offset 0x100000) และหาไฟล์ผิดปกติ
3. Steganography      → ถอด ASCII-art QR Code กลับเป็นข้อความ
```

flag ที่ได้: `CYBERHEROCTF{Th1S_isA_S3cR@t_M3ssage}`
