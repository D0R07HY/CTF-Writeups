- **Goal:** หาไฟล์ที่มีเงื่อนไขตามนี้ :

								owned by user bandit7
								owned by group bandit6
								33 bytes in size
	
- **Key Command:**   ls   , find ,  cat 
    
- **The Logic:** ใช้คำสั่ง **find / -user[] -group[] -size[] 2>/dev/null** เพื่อหาไฟล์ตามเงื่อนไขที่โจทย์บอก

- **New Loot (ความรู้ใหม่):** 

				/ เพื่อให้ค้นหาตั้งแต่รากฐานของระบบจนถึงไฟล์ย่อยๆ หรือ ค้นหาไฟล์ทั้งหมด
				2>/dev/null เป็นเทคนิคในการ "ทิ้งข้อความแจ้งข้อผิดพลาด"ของคำสั่งในระบบ Linux
    