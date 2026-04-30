- **Goal:** เปรียบเทียบ passwords.old และ passwords.new 
    
- **Key Command:** ssh , diff , [[Navigation & Exploration|ls]]
    
- **The Logic:** 

			ใช้คำสั่ง `diff passwords.new passwords.old` เพื่อเอารหัส หลังจากได้รหัสแล้วเรา
			ไม่สามารถไปยัง Level 18 ตรงๆได้เราต้องใช้คำสั่ง 
			`ssh bandit18@bandit.labs.overthewire.org -p 2220` เพื่อเข้าไปยัง Level 18
    
- **New Loot (ความรู้ใหม่):** -