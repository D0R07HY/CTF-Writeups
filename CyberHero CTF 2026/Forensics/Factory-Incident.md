# Factory Incident (Forensics, 300pts)

**Flag:** `CYBERHEROCTF{dn5_tunn3l_3xf1l_4ft3r_35t0p_byp455}`
**สถานะ:** 🟡 พบ flag แล้ว (ยังไม่ได้ยืนยันกับระบบ)
**หมวดย่อย:** Forensics — Network Covert Channel, Packet Header Analysis

---

## 📜 โจทย์

ให้ไฟล์ `factory_incident.pcap` มา อ้างว่าเป็นเหตุการณ์ในโรงงาน (ICS/OT)
ที่มีการขโมยข้อมูล credential ออกไป โจทย์ให้หาว่า **ข้อมูลถูกส่งออกไปทางไหน และส่งอะไรออกไป**

**กับดักของโจทย์นี้:** ในไฟล์มี flag ปลอมกระจายอยู่ **เกือบ 20 ที่** (ทุกโปรโตคอล)
และมีข้อความล่อให้เชื่อว่าช่องทาง DNS คือช่องทางจริง

```text
CYBERHEROCTF{dns_channel_is_a_decoy_check_packet_headers}   ← DECOY (ตัวอักษรล่อ)
```

คำใบ้ในตัว decoy บอกใบ้ทางออก: **"check packet headers"** → ของจริงไม่ได้อยู่ใน payload
แต่อยู่ใน **header field** ของ packet

---

## 📊 ภาพรวมไฟล์

```text
Packets : 594
ช่วงเวลา: 2025-09-05 15:33:20 → 15:35:16 (~116 วินาที)
สัดส่วน : TCP 329, UDP 225, ARP 24, ICMP 16

เครือข่าย:
  10.10.1.0/24   → ฝั่ง IT / corporate (DNS 10.10.1.2, mail, web)
  10.10.5.0/24   → ฝั่ง OT / โรงงาน (Modbus 502, RDP 10.10.5.99, telnet, SMB)
  203.0.113.10   → ปลายทาง SMTP ภายนอก (TEST-NET-3)
```

---

## 🕳️ Decoy ทั้งหมดที่โจทย์วางไว้

ระหว่างทางจะเจอ "flag" หลายอันที่ฝังอยู่ในที่ต่าง ๆ — **ทุกอันเป็นของปลอม**:

| # | ที่ซ่อน | ลักษณะ |
|---|---|---|
| 1 | DNS TXT record | `CYBERHEROCTF{dns_channel_is_a_decoy_check_packet_headers}` ← ตัวล่อหลัก |
| 2 | MQTT retained payload | flag ปลอมใน topic ของ broker ในโรงงาน |
| 3 | Modbus device identification | vendor/product string |
| 4 | Modbus setpoint write | ค่า register ที่เข้ารหัสเป็น ASCII |
| 5 | HTTP header | flag ฝังใน custom header (base64) |
| 6 | HTML comment | ฝังในหน้าเว็บของ HMI |
| 7 | SMTP attachment | ไฟล์แนบ base64 |
| 8 | Telnet banner | ข้อความต้อนรับ |
| 9 | FTP banner + เนื้อหาไฟล์ | ไฟล์ปลอมใน FTP |
| 10 | SNMP string | community/description |
| 11 | Syslog message | log ปลอมจากอุปกรณ์ |
| 12 | RDP cookie | cookie ใน handshake |
| 13 | SMB negotiate | ข้อความใน session setup |
| 14 | DHCP vendor class | ตัวระบุผู้ผลิต |
| 15 | ICMP payload | flag ฝังตรง ๆ ใน echo payload |
| 16 | TCP stream ทั่วไป | `strings` แล้วเจอเต็ม ๆ |

> **บทเรียนสำคัญ:** การ `strings pcap | grep CYBERHEROCTF` แล้วส่งอันแรกที่เจอ
> **จะได้ flag ปลอม** เพราะโจทย์ตั้งใจวางไว้ให้คนขี้เกียจเจอ

---

## 🔍 ขั้นตอนการแก้

### 1. มองหาความผิดปกติของ header (ตามคำใบ้)

คำใบ้บอกให้ดู **packet headers** — เปิดดู `IP ID` ของทุก packet เทียบกัน
จะพบว่ากลุ่ม ICMP มี pattern ที่ไม่ปกติ: **ICMP sequence number กับ high byte ของ IP ID สัมพันธ์กันแบบ 1:1**

```python
from scapy.all import rdpcap, ICMP, IP

pkts = rdpcap("factory_incident.pcap")

mapping = {}
for p in pkts:
    if p.haslayer(ICMP) and p.haslayer(IP):
        seq  = p[ICMP].seq
        idhi = (p[IP].id >> 8) & 0xFF
        mapping.setdefault(seq, idhi)

print(mapping)
# {0: 48, 1: 84, 2: 83, 3: 51, 4: 99, 5: 117, 6: 114, 7: 51}
```

### 2. แปลง mapping เป็น key

```python
key = bytes(mapping[i] for i in sorted(mapping))
print(key)          # b'0TS3cur3'
```

```text
48='0'  84='T'  83='S'  51='3'  99='c'  117='u'  114='r'  51='3'
→  "0TS3cur3"   (leetspeak ของ "0TS3cure" / "OT Secure")
```

ได้ **XOR key ยาว 8 byte** = `0TS3cur3`

### 3. หา ciphertext

ช่องทางที่บรรทุกข้อมูลจริงคือกลุ่ม **UDP port 7 (echo)** ซึ่งมี index byte ใน payload
ทำหน้าที่เรียงลำดับ byte ของ ciphertext (ถ้าไม่เรียงตาม index จะได้ขยะ)

```python
from scapy.all import UDP

records = {}
for p in pkts:
    if p.haslayer(UDP) and p[UDP].dport == 7:
        raw = bytes(p[UDP].payload)
        if not raw:
            continue
        idx = raw[0]                  # index byte
        records[idx] = raw[1:]        # ข้อมูลจริง

ct = b"".join(records[i] for i in sorted(records))
print("ciphertext len:", len(ct))     # 49
```

### 4. XOR แล้วได้ flag

```python
key = b"0TS3cur3"
pt = bytes(c ^ key[i % len(key)] for i, c in enumerate(ct))
print(pt.decode())
```

ผลลัพธ์ (ยืนยันจากไฟล์จริง):

```text
ICMP seq->idhi: {0: 48, 1: 84, 2: 83, 3: 51, 4: 99, 5: 117, 6: 114, 7: 51}
ICMP key:       b'0TS3cur3'
ciphertext len: 49
plaintext:      CYBERHEROCTF{dn5_tunn3l_3xf1l_4ft3r_35t0p_byp455}
```

🏁 **FLAG: `CYBERHEROCTF{dn5_tunn3l_3xf1l_4ft3r_35t0p_byp455}`**

อ่านความหมายได้ว่า *"DNS tunnel, exfil after stop bypass"* — สื่อว่าแฮกเกอร์
ใช้ช่องทางที่ดูเหมือน DNS เพื่อเบี่ยงเบนความสนใจ แต่ของจริงใช้ **IP ID ของ ICMP**
(ตรงกับคำใบ้ *"check packet headers"*)

---

## 🛠️ Solve Script (สรุป)

```python
from scapy.all import rdpcap, ICMP, IP, UDP

pkts = rdpcap("factory_incident.pcap")

# 1) key จาก ICMP seq -> IP ID high byte
mapping = {}
for p in pkts:
    if p.haslayer(ICMP) and p.haslayer(IP):
        mapping.setdefault(p[ICMP].seq, (p[IP].id >> 8) & 0xFF)
key = bytes(mapping[i] for i in sorted(mapping))
assert key == b"0TS3cur3", key

# 2) ciphertext จาก UDP/7 (เรียงตาม index byte)
rec = {}
for p in pkts:
    if p.haslayer(UDP) and p[UDP].dport == 7:
        raw = bytes(p[UDP].payload)
        if raw:
            rec[raw[0]] = raw[1:]
ct = b"".join(rec[i] for i in sorted(rec))

# 3) XOR
pt = bytes(c ^ key[i % len(key)] for i, c in enumerate(ct))
print(pt.decode())      # CYBERHEROCTF{dn5_tunn3l_3xf1l_4ft3r_35t0p_byp455}
```

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Covert channel ใน header** | IP ID, TCP seq, TTL, window size — เป็นฟิลด์ที่คนมักไม่ตรวจ ใช้ซ่อนข้อมูลได้ |
| **Decoy flag** | โจทย์ forensics ยุคใหม่ใส่ flag ปลอมหลายอันเพื่อทดสอบวินัยการวิเคราะห์ |
| **คำใบ้ในตัว decoy** | `..._check_packet_headers` ไม่ใช่แค่ชื่อเล่น แต่เป็น **คำใบ้ทางออก** |
| **Index byte** | ช่องทางที่ packet อาจสลับลำดับได้ มักมี index byte ใน payload เพื่อ reassemble |

## ✅ สรุป

โจทย์นี้วัด **ความอดทนและวินัย** มากกว่าความรู้เทคนิค:

- เจอ flag 10+ อัน → ต้องไม่รีบส่ง
- คำใบ้ชี้ไปที่ **header ไม่ใช่ payload** → เลยจุดที่คน 90% มองข้าม
- ต้องรู้ว่าใน ICMP/UDP มีฟิลด์ไหนที่ใช้เป็นช่องทางลับได้

flag จริงคือ `CYBERHEROCTF{dn5_tunn3l_3xf1l_4ft3r_35t0p_byp455}`
