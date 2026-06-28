# Point Broken (Crypto, 100pts)

**Flag:** `DELTA{ph_attack_smooth_prime}`

## โจทย์

โจทย์ discrete logarithm บนกลุ่มที่ลำดับแตกตัวประกอบได้ง่าย

## ขั้นตอน

1. หาลำดับของกลุ่มหรือรู้โครงสร้างของมันจากโจทย์
2. แยกตัวประกอบลำดับกลุ่ม
3. ใช้ Pohlig-Hellman แก้ discrete log ทีละตัวประกอบ
4. รวมคำตอบกลับด้วย CRT
5. แปลงผลลัพธ์กลับเป็นข้อความหรือ flag

## หลักการ

Pohlig-Hellman มีประสิทธิภาพมากเมื่อ order ของกลุ่มเป็น smooth number เพราะสามารถแตกปัญหาใหญ่เป็นปัญหาเล็กหลายก้อนได้

## สรุป

- เจอ discrete log ไม่ได้แปลว่าต้อง brute-force
- ถ้า group order smooth ให้คิดถึง Pohlig-Hellman ก่อน
