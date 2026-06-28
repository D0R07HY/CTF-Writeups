# wavy (Crypto, 100pts)

**Flag:** `tjctf{ch3bysh3v_p0lyn0m1!l_676767}`

## โจทย์

โจทย์ใช้ Chebyshev polynomial สร้าง keystream แล้ว XOR กับข้อความ

## ขั้นตอน

1. อ่าน logic การสร้างค่า sequence จากโจทย์
2. สร้าง keystream แบบเดียวกับฝั่งเข้ารหัส
3. XOR keystream กลับกับ ciphertext
4. ได้ plaintext ที่มี flag

ตัวอย่างแนวคิด:

```python
plaintext = bytes(c ^ k for c, k in zip(ciphertext, keystream))
```

## หลักการ

โจทย์ stream/XOR ส่วนใหญ่พังทันทีเมื่อสร้าง keystream ได้ตรงกับฝั่งเข้ารหัส

## สรุป

- แกนสำคัญคือสร้าง keystream ให้ตรง
- เมื่อ keystream ถูกต้อง การถอดรหัสด้วย XOR จะตรงไปตรงมา
