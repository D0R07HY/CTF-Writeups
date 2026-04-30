- **Goal:** เข้าไปเอารหัสใน Bandit14 โดยใช้รหัส sshkey.private
    
- **Key Command:** chmod  ,  ls  / (Window CMD) icacls , inheritance , scp , ssh
    
- **The Logic:**  
	
		วิธีใน Windows:
		สร้างโฟรเดอร์เปล่าขึ้นมาและรัน cmd ในโฟล์เดอร์นั้น พิมพ์คำสั่ง scp -P 2220 
		bandit13@bandit.labs.overthewire.org:sshkey.private . เพื่อก็อปปี้ไฟล์ 
		sshkey.private จากเซิฟเวอร์ของ bandit13@bandit.labs.overthewire.org มาไว้ยัง
		โฟรเดอร์ปัจจุบันที่เราอยู่
		ทำการถอนศิทธิในไฟล์ sshkey.private ทั้งหมดและเพิ่มสิทธิให้ตัวเองโดยใช้คำสั่ง:
		icacls "ตำแหน่งไฟล์ของsshkey" /inheritance:r /grant:r "[ชื่อผู้ใช้]":"(R,W)"
		หลังจากนั้นให้ใช้คำสั่ง:
		ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
		หลังจากนั้นให้พิมพ์คำสั่ง:
		cat /etc/bandit_pass/bandit14 เพื่อเอารหัสผ่าน

		วิธีใน Linux:
		ใช้คำสั่ง chmod 600 sshkey.private เพื่อทำการทอนศิทธิในไฟล์ sshkey.private ทั้งหมดและ
		เพิ่มสิทธิให้ตัวเอง
		หลังจากนั้นให้ใช้คำสั่ง:
		ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
		หลังจากนั้นให้พิมพ์คำสั่ง :
		cat /etc/bandit_pass/bandit14 เพื่อเอารหัสผ่าน
    
- **New Loot (ความรู้ใหม่):** การรันคำสั่งการใช้สิทธิต่าง chmod  บน linux
					 และคำสั่ง `icacls` , `inheritance` บน window