# 🚩 CTF Writeups & Wargame Logs

![Focus](https://img.shields.io/badge/Focus-Cybersecurity%20%7C%20Forensics-black?style=for-the-badge&logo=hackthebox)
![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)

พื้นที่สำหรับรวบรวมบันทึกการเจาะระบบ, การแก้โจทย์ Capture The Flag (CTF), และ Wargames

Repository นี้ไม่ได้มีจุดประสงค์เพื่อการแปะ "เฉลย (Flag)" เพียงอย่างเดียว แต่เน้นไปที่การบันทึก **กระบวนการคิด (Methodology)** ทำความเข้าใจพฤติกรรมของระบบ และเจาะลึกการทำงานของคำสั่งต่างๆ อย่างเป็นขั้นเป็นตอน

## 📂 Active Campaigns

### [OverTheWire: Bandit](./OverTheWire%20|%20Bandit)
Wargame ระดับตำนานสำหรับฝึกฝนทักษะ Linux Command Line เบื้องต้นและการทำความเข้าใจสิทธิการเข้าถึงไฟล์ (Permissions)
- **Current Progress:** กำลังลุยทะลวงด่าน (ปัจจุบันถึงช่วง **Level 21** 🚀 และกำลังทยอยอัปเดต Writeup ลง Repo)
- **Focus Area:** Navigation, Text Processing, File Permissions, และ Networking Basics

### [DeltaCTF 2026](./DeltaCTF%202026)
สนาม CTF ที่รวมโจทย์หลายหมวด ทั้ง Web, Pwn, Misc, Crypto, Forensics และ Reverse โดยในคลังนี้จะเน้นบันทึกวิธีคิด ขั้นตอนที่ใช้จริง และสิ่งที่ได้เรียนรู้จากการแตกโจทย์แต่ละข้อ
- **Current Progress:** อัปเดต Writeup แล้วรวม **22 ไฟล์** ครอบคลุมหลายหมวดที่แก้ได้ และยังคงทยอยจัดระเบียบเนื้อหาให้ครบถ้วนและอ่านต่อได้ง่ายขึ้น
- **Focus Area:** Web Exploitation, Binary Exploitation, Cryptanalysis, Log Analysis, File Recovery, และ Reverse Engineering

### [CyberHero CTF 2026](./CyberHero%20CTF%202026)
สนาม CTF สาย Blue Team / Red Team ที่โจทย์เน้นงานวิเคราะห์หลักฐานจริง (VM image, memory dump, pcap) และโซ่ Crypto ที่ต้องโจมตีต่อกันหลายชั้น
- **Current Progress:** อัปเดต Writeup แล้วรวม **8 ไฟล์** — แก้สำเร็จและยืนยัน flag กับระบบแล้ว 2 ข้อ (Overpost, Faultline Ledger)
- **Focus Area:** Offline Disk Forensics, Custom Binary Format Parsing, RSA Fault Attack, SHA-256 Length Extension, AES-CTR Keystream Reuse, Network Covert Channel, ASCII-Art QR Recovery

### [HackTheBox](./HackTheBox)
ชุดโจทย์จาก HackTheBox ที่เน้นงาน Forensics, Crypto และ Reverse Engineering พร้อมบันทึกการวิเคราะห์โจทย์ Web/Pwn ที่ยังแก้ไม่จบไว้เป็นแนวทางต่อยอด
- **Current Progress:** อัปเดต Writeup แล้วรวม **5 ไฟล์** — แก้สำเร็จ 3 ข้อ (Acknowledge the Corn, Always Has Been, Haunted Houseparty)
- **Focus Area:** C2 Protocol Reverse Engineering (Covenant), Windows Memory Forensics, Affine Cryptanalysis บน GF(2), Static Binary Analysis, gRPC Prototype Pollution, Heap Exploitation (tcache poisoning)

### [PicoCTF](./PicoCTF)
สนามฝึก CTF ที่เหมาะกับการเก็บพื้นฐานแบบเป็นขั้นเป็นตอน โดยชุดที่ย้ายเข้ามารอบนี้เน้นโจทย์สาย Cryptography, General Skills และ Web Exploitation จากโน้ตเดิมที่จัดไว้ใน Obsidian
- **Current Progress:** ย้ายและจัดระเบียบ Writeup แล้วรวม **10 ไฟล์ Markdown** พร้อมแยกหมวดและทำลิงก์รูปให้เปิดบน GitHub ได้ตรง ๆ
- **Focus Area:** Classical Ciphers, Encoding Chains, Session Abuse, HTTP Headers, และ SSTI

## 🧠 Writeup Methodology

เพื่อให้ง่ายต่อการทบทวนความรู้และเป็นระบบ บันทึกทุกไฟล์ในคลังนี้จะถูกจัดโครงสร้างอย่างชัดเจน:

*   **🎯 Goal:** กำหนดเป้าหมายหรือเงื่อนไขหลักของโจทย์ในด่านนั้นๆ
*   **⚙️ The Logic:** อธิบายตรรกะ วิธีคิด และขั้นตอนการใช้คำสั่งเพื่อแก้ปัญหา (ทำไมถึงใช้คำสั่งนี้ ทำงานอย่างไร)
*   **💎 New Loot:** สรุปเทคนิค สัญลักษณ์พิเศษ หรือคอนเซปต์ใหม่ที่เก็บเกี่ยวได้จากด่านนี้ เพื่อนำไปประยุกต์ใช้ในอนาคต

## 📁 Repository Structure
```text
CTF-Writeups/
├── CyberHero CTF 2026/
│   ├── Web/
│   ├── Crypto/
│   ├── Misc/
│   ├── Forensics/
│   ├── OSINT/
│   └── README.md
├── HackTheBox/
│   ├── Forensics/
│   ├── Crypto/
│   ├── Reverse/
│   ├── Web/
│   ├── Pwn/
│   └── README.md
├── DeltaCTF 2026/
│   ├── Web/
│   ├── Pwn/
│   ├── Misc/
│   ├── Crypto/
│   ├── Forensics/
│   ├── Reverse/
│   └── README.md
├── PicoCTF/
│   ├── Cryptography/
│   ├── General-Skills/
│   ├── Web-Exploitation/
│   ├── assets/
│   └── README.md
├── OverTheWire | Bandit/
│   ├── Level-0.md
│   ├── Level-1.md
│   └── ...
└── README.md
```
