# The Broken Key (Crypto, 100pts)

**Flag:** `DELTA{gcd_is_your_friend_1337}`

## โจทย์

ให้ค่า RSA สองชุดมา แต่ทั้งสองชุดใช้ prime ร่วมกันโดยไม่ตั้งใจ

## ขั้นตอน

1. อ่านค่า `n1`, `n2`, `c1`, `c2`, `e`
2. หา prime ที่ใช้ร่วมกันด้วย `gcd(n1, n2)`
3. แยก `n1` และ `n2` ออกเป็นตัวประกอบ
4. คำนวณ private exponent ของทั้งสองชุด
5. ถอด `c1` และ `c2` แล้วนำ plaintext มาต่อกัน

ตัวอย่าง:

```python
from math import gcd

p = gcd(n1, n2)
q1 = n1 // p
q2 = n2 // p
```

## หลักการ

RSA จะพังทันทีถ้า modulus คนละตัวแชร์ prime เดียวกัน เพราะ `gcd` จะดึง prime ร่วมนั้นออกมาได้ตรง ๆ

## สรุป

- ถ้าเห็น RSA หลายชุด ให้ลอง `gcd` ระหว่าง modulus ก่อนเสมอ
- shared prime = ได้ private key กลับมาง่ายมาก
