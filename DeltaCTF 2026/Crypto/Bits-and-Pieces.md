# Bits and Pieces (Crypto, 100pts)

**Flag:** `DELTA{r5a_p4rt1al_l3ak_i5_d4n9erous}`

## โจทย์

โจทย์ RSA ที่มีการ leak ข้อมูลของ prime ออกมาบางส่วน ทำให้ไม่ต้อง brute-force ทั้ง prime

## ขั้นตอน

1. อ่านค่าที่โจทย์ให้มาและแยกส่วนที่รู้ของ prime
2. brute-force เฉพาะส่วนท้ายที่หายไป
3. ทดสอบว่าค่าที่เดาถูกหาร `n` ลงตัวหรือไม่
4. เมื่อได้ prime ครบ ก็ถอดรหัส RSA ตามปกติ

แนวคิดหลัก:

```python
candidate = known_high_bits | suffix
if n % candidate == 0:
    p = candidate
```

## หลักการ

ถ้าโจทย์ leak บิตของ prime ออกมาบางส่วน พื้นที่ค้นหาจะเล็กลงมากจน brute-force ได้จริง

## สรุป

- partial prime leak ทำให้ RSA อ่อนแอมาก
- ไม่จำเป็นต้องเดาทั้ง prime ถ้ารู้โครงสร้างของมันแล้ว
