# Pwned01 (Pwn, 100pts)

**Flag:** `DELTA{formatstringvulnerability}`

## โจทย์

Binary ช่องโหว่ Format String — อ่าน/เขียนหน่วยความจำตามอำเภอใจ

## ขั้นตอน

### 1. วิเคราะห์

```bash
file vuln     # ELF 64-bit, x86-64
checksec vuln # No PIE, No canary, NX enabled
```

### 2. Decompile

```c
void vuln() {
  char buf[64];
  read(0, buf, 200);
  printf(buf);     // ⚠️ FORMAT STRING!
}
```

### 3. ทดสอบ

```bash
echo "AAAA.%p.%p.%p.%p" | ./vuln
```

`%p` leak pointer จาก stack

## หลักการ

```c
printf("%s", buf);  // SAFE
printf(buf);         // VULNERABLE
```

Format specifiers:
- `%p` — leak pointer
- `%x` — leak hex
- `%s` — leak string
- `%n` — **เขียน** จำนวน byte ไปยัง address

## สรุป

- `printf(user_input)` → format string vulnerability
- ใช้ `%p`, `%x` อ่าน stack, `%n` เขียนค่า
- ป้องกัน: `printf("%s", buf)` เสมอ
