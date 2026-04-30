- **Goal:** ตรวจหาพอร์ตที่เปิดอยู่ตั้งแต่ 31000-32000 ของ localhost จากนั้นนำ sshkey เพื่อไปยังLevel17
    
- **Key Command:** chmod , ncat , nmap / (Window CMD) icacls , inheritance , scp , ssh
    
- **The Logic:** 

				ใช้คำสั่ง nmap -p 31000-32000 localhost เพื่อหาพอร์ตที่เปิดอยู่และพิพ์คำสั่ง 
                ncat --ssl localhost [เลขพอร์ต] และใส่รหัส Level 16 จนกว่าจะมีพอร์ตที่คายรหัส
                -----BEGIN RSA PRIVATE KEY----- 
                ก็อปปี้ข้อความทั้งหมด 
                ไปยัง windows และสร้างโฟล์[ชื่อไฟล์].txt และเอาข้อความที่ก็อปปี้มาวางลงและพิมคำสั่ง
                เปิด cmd และทำการถอนศิทธิในไฟล์ทั้งหมดและเพิ่มสิทธิให้ตัวเองโดยใช้คำสั่ง:
				icacls "ตำแหน่งไฟล์ของ.txt" /inheritance:r /grant:r "[ชื่อผู้ใช้]":(R,W)"
				หลังจากนั้นใช้คำสั่ง:
				ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
				หลังจากนั้นให้พิมพ์คำสั่ง:
				cat /etc/bandit_pass/bandit17 เพื่อเอารหัสผ่าน
    
- **New Loot (ความรู้ใหม่):** -