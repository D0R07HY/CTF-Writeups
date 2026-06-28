# LOG LOG LOG (Forensics, 100pts)

**Flag:** `DELTA{L0g_An4ly515_D3ft_H4nd}`

## โจทย์

ให้ไฟล์ `access.log` ที่ดูเหมือนเป็น log ธรรมดา แต่จริง ๆ เก็บร่องรอยของ blind SQL injection ไว้

## ขั้นตอน

1. อ่าน request ที่ถูกยิงเข้าเว็บ
2. สังเกตว่า payload มีรูปแบบประมาณ `ascii(substring(flag, pos, 1)) > threshold`
3. ใช้ขนาด response เป็น oracle:
   - `512` = true
   - `480` = false
4. สำหรับแต่ละตำแหน่งของ flag หา threshold สูงสุดที่ยังตอบจริง
5. ค่าจริงของตัวอักษรคือ `threshold + 1`

ตัวอย่าง pattern ที่ใช้ parse:

```python
ascii(substring(flag, pos, 1)) > threshold
```

## หลักการ

Blind SQLi ไม่จำเป็นต้องมีข้อความตอบกลับชัดเจน ขอแค่มี side channel เช่น response length หรือ timing ก็ใช้ดึงข้อมูลออกมาได้

## สรุป

- log file เองอาจเป็นหลักฐานเพียงพอในการกู้ flag
- response size เป็น oracle ที่ทรงพลังมากใน blind SQLi
