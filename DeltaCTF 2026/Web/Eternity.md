# Eternity (Web, 100pts)

**Flag:** `DELTA{1ts_just_l00k1ng_1n_th3_c0mm3nts}`

## โจทย์

Flag ซ่อนอยู่ในหน้าเว็บ — ดู source code แล้วเจอ comment

## ขั้นตอน

1. เปิด URL
2. คลิกขวา → View Page Source (`Ctrl+U`)
3. พบ flag ใน HTML comment

```html
<!-- DELTA{1ts_just_l00k1ng_1n_th3_c0mm3nts} -->
```

## หลักการ

HTML comment (`<!-- ... -->`) ใช้เขียนหมายเหตุในโค้ด ไม่แสดงผลบนหน้าเว็บแต่มองเห็นใน source code นักพัฒนาลืมลบ comment ก่อน deploy

## สรุป

- เช็ค HTML source code ทุกครั้งก่อนลองวิธีอื่น
- `Ctrl+U` หรือคลิกขวา → View Page Source
