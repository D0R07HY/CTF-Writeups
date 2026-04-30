- **Goal:** สร้างช่องทางเชื่อมต่อไปยัง Level 15 โดยใช้รหัสผ่านของ Level 14 โดยมีชื่อIP คือ localhost และ           port 30001 โดยใช้ SSL/TLS
    
- **Key Command:** openssl
    
- **The Logic:** ใช้คำสั่ง `openssl s_client -connect localhost:30001`หลังจากนั้นให้ใส่รหัสของ Level 15
    
- **New Loot (ความรู้ใหม่):** `openssl` คือชุดเครื่องมือครอบจักรวาลที่ใช้จัดการเกี่ยวกับการเข้ารหัสและใบรับรองความปลอดภัยทั้งหมดใน Linux `s_client` ซึ่งทำหน้าที่เป็นเหมือน Netcat ฉบับที่รองรับการเข้ารหัส SSL