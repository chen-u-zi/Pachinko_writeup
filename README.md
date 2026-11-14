# picoCTF 2025 — Pachinko Writeup
 
> Web Exploitation · NAND Simulator · 前端繞過 + Von Neumann 記憶體注入
 
Pachinko 是 picoCTF 2025 的一道 Web 題，表面是一個 NAND 邏輯閘模擬器，實際藏了兩層挑戰：
 
| Flag | 難度 | 核心技術 |
|------|------|----------|
| Flag 1 | Medium | 前端 JS 限制繞過，用 `fetch()` 直接送 API 請求 |
| Flag 2 | Hard | Von Neumann 架構 exploit：整數溢位寫入 → opcode patch → 指令注入 |
 
---
 
## 檔案說明
 
| 檔案 | 內容 |
|------|------|
| [`flag1-frontend-bypass.md`](./flag1-frontend-bypass.md) | Flag1 完整解題過程：NAND 原理、前端鎖分析、fetch payload |
| [`flag2-von-neumann-exploit.md`](./flag2-von-neumann-exploit.md) | Flag2 完整解題過程：ISA 分析、寫入原語、Mutation、NAND→ADD patch、flag.bin 注入 |
| [`playbook-frontend-bypass.md`](./playbook-frontend-bypass.md) | 通用心法卡：前端限制繞過的 SOP、決策流程、快速查核表（不綁特定題目） |
 
---
 
## 心法速查
 
這兩道題教會我的六個思維工具：
 
- **Trust Boundary** — 前端不是信任邊界，伺服器才是
- **Drop a Layer** — 當前層解不了，往底層看（UI → HTTP → 記憶體）
- **Anchor** — 從已知能用的東西出發，向外延伸
- **Check ≠ Use** — 驗證時和執行時走不同路徑
- **Errors are Oracles** — 每個錯誤都在告訴你一個假設是錯的
- **Exploit = Chain** — 哪個環節斷了，整個 exploit 就壞了
---
 
*授權用途：CTF 學習 / 授權滲透測試 / 自己的機器。*
