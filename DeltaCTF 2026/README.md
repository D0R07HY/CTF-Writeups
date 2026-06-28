# DeltaCTF 2026 - Writeups

**ทีม:** Singularity  
**ผู้เล่น:** D0R07HY2  
**แพลตฟอร์ม:** http://ctf.deltactf.site (IP: 13.238.150.105, EC2 ap-southeast-2)

---

## Web Challenges (5)

| # | Challenge | Points | ไฟล์ |
|---|-----------|--------|------|
| 1 | [Eternity](Web/Eternity.md) | 100 | HTML comment |
| 2 | [Librarian Secret](Web/Librarian-Secret.md) | 100 | SQL Injection (UNION SELECT) |
| 3 | [Pingpong](Web/Pingpong.md) | 100 | Command Injection |
| 4 | [Magic Ruby](Web/Magic-Ruby.md) | 100 | Ruby YAML Deserialization RCE |
| 5 | [Subzero Station](Web/Subzero-Station.md) | 100 | JWT forgery + Prototype Pollution |

## Pwn Challenges (3)

| # | Challenge | Points | ไฟล์ |
|---|-----------|--------|------|
| 6 | [Pwned01](Pwn/Pwned01.md) | 100 | Format String Vulnerability |
| 7 | [The Echoes](Pwn/The-Echoes.md) | 176 | Buffer Overflow -> ret2win |
| 8 | [Flag Facility](Pwn/Flag-Facility.md) | 356 | Buffer Overflow -> ROP Chain |

## Misc Challenges (3)

| # | Challenge | Points | ไฟล์ |
|---|-----------|--------|------|
| 9 | [Welcome](Misc/Welcome.md) | - | Intro / visible flag |
| 10 | [Find The Flag](Misc/Find-The-Flag.md) | - | Search hidden text |
| 11 | [Robots](Misc/Robots.md) | - | robots.txt / hidden path |

## Crypto Challenges (4)

| # | Challenge | Points | ไฟล์ |
|---|-----------|--------|------|
| 12 | [The Broken Key](Crypto/The-Broken-Key.md) | 100 | RSA shared prime via GCD |
| 13 | [Bits and Pieces](Crypto/Bits-and-Pieces.md) | 100 | Partial prime leak |
| 14 | [Point Broken](Crypto/Point-Broken.md) | 100 | Pohlig-Hellman |
| 15 | [wavy](Crypto/Wavy.md) | 100 | Chebyshev keystream XOR |

## Forensics Challenges (3)

| # | Challenge | Points | ไฟล์ |
|---|-----------|--------|------|
| 16 | [unfinished-file](Forensics/Unfinished-File.md) | 356 | Recover data from incomplete file |
| 17 | [LOG LOG LOG](Forensics/LOG-LOG-LOG.md) | 100 | Blind SQLi from access log |
| 18 | [Broken Image](Forensics/Broken-Image.md) | 436 | Repair JPEG header |

## Reverse Challenges (4)

| # | Challenge | Points | ไฟล์ |
|---|-----------|--------|------|
| 19 | [average flag checker](Reverse/Average-Flag-Checker.md) | 400 | Reverse transform tables |
| 20 | [Checking Flags](Reverse/Checking-Flags.md) | 400 | TEA-style decryption |
| 21 | [Lisan Switchboard](Reverse/Lisan-Switchboard.md) | 400 | Custom VM / substitution tables |
| 22 | [Ancient Artifact](Reverse/Ancient-Artifact.md) | 400 | Symbol encoding + ROT13 + Base64 |

---

## Summary

- **DeltaCTF Writeups in Repo:** 22
- **Added in this update:** Misc, Crypto, Forensics, Reverse
- **Tools:** Python, pwntools, Ghidra, objdump, custom decoders, log parsing
