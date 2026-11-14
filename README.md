# CTF Writeups ＆ 作戰手冊 🏴
 
> 個人 CTF 解題紀錄與攻擊心法，主攻 **Web Exploitation**（偶有 RE/Pwn）。
> 目標賽事：AIS3 EOF。
 
這不是單純的 writeup 倉庫，而是**會成長的作戰手冊**：心法卡（上層思維，不變）＋ writeup（每解一題補一篇）＝ 自己的作戰手冊。
 
---
 
## 🗂️ 題目 Writeup
 
| 題目 | 類別 | 檔案 | 核心技術 |
|------|------|------|----------|
| Pachinko Flag1 | **Web** | [`flag1-frontend-bypass.md`](./flag1-frontend-bypass.md) | 前端 JS 限制繞過，`fetch()` 直連後端 |
| Pachinko Flag2 | RE/Pwn | [`flag2-von-neumann-exploit.md`](./flag2-von-neumann-exploit.md) | Von Neumann 記憶體注入 / opcode patch |
| No FA | **Web** | [`no-fa.md`](./no-fa.md) | 弱密碼破解 + OTP 存於不加密 session cookie |
| Credential Stuffing | **Web** | [`credential-stuffing.md`](./credential-stuffing.md) | 外洩帳密自動灌入（含 nc 服務自動化模板） |
| Heartbleed | RE | [`heartbleed.md`](./heartbleed.md) | 記憶體 over-read + Ghidra 逆向 + djb2 重算 |
 
---
 
## 🧭 通用心法卡
 
| 檔案 | 內容 |
|------|------|
| [`web-playbook.md`](./web-playbook.md) | **Web 通用心法總卡**：Source→Sink、攻擊面地圖、Trust Boundary、Check≠Use、Errors are Oracles、Exploit=Chain |
| [`playbook-frontend-bypass.md`](./playbook-frontend-bypass.md) | 前端限制繞過 SOP：UI 擋住你時，如何繞過前端直接打後端 |
 
---
 
## 🔑 六個核心心法
 
| 心法 | 一句話 |
|------|--------|
| **Source → Sink** | 追蹤使用者可控資料流到哪個危險終點——注入類題目的萬能鑰匙 |
| **攻擊面地圖** | 看不到的東西打不到；拿到源碼，題目解一半 |
| **Trust Boundary** | 前端 / cookie 不是信任邊界，伺服器才是 |
| **Check ≠ Use** | 驗證的解析器和使用的解析器，看到的常常不一樣 |
| **Errors are Oracles** | 回應的差異就是情報，每個錯誤告訴你一個假設錯了 |
| **Exploit = Chain** | 高分題是串出來的；單招打不穿就找墊腳石 |
 
---
 
## 📈 Web 各洞進度
 
解到就打勾，並在 `web-playbook.md` 補範例：
 
- [x] 前端限制繞過（Pachinko Flag1）
- [x] Session / Cookie 敏感資料外洩（No FA）
- [x] 認證 / 自動化攻擊（Credential Stuffing）
- [ ] SQL Injection
- [ ] SSTI（模板注入）
- [ ] SSRF
- [ ] JWT / 認證繞過
- [ ] 檔案上傳 / LFI / 路徑穿越
- [ ] 反序列化
- [ ] XSS（偷 admin bot cookie）
- [ ] Prototype Pollution
---
 
## 🎯 練習資源
 
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — Web 必刷，免費且系統化
- [picoCTF](https://picoctf.org) — 入門題庫
- [PentesterLab](https://pentesterlab.com) — 實戰導向
- [root-me.org](https://www.root-me.org) — 大量 Web 題
---
 
*授權用途：CTF 學習 / 授權滲透測試 / 自己的機器。*
 
