- **Goal:** เลื่อนAlpha Bet ของแต่ละตัวอักษรในข้อความไป 13 ตำแหน่ง
    
- **Key Command:** [[Navigation & Exploration|ls]]  , [[Viewing & Searching|cat]] , tr , [[Piping & Redirection|pipe]] ( | )
    
- **The Logic:** ใช้คำสั่ง **[[Viewing & Searching|cat]]** เพื่ออ่านไฟล์ขึ้นจอและ **[[Piping & Redirection|pipe]] ( | )** เพื่อส่งคำสั่งไปให้ **tr "A-Za-z" "N-ZA-Mn-za-m"** เพื่อเลื่อนตำแหน่งของ Alpha bet ไป 13 ตำแหน่ง 
    
- **New Loot (ความรู้ใหม่):** คำสั่ง **tr** เพื่อแปร , แทนที่ , หรือลบตัวอักขระ