# 通用心法卡：前端限制繞過（Client-Side Bypass）

> 類別：Web · 通用 SOP  
> 目的：下次遇到「UI 擋住你」時，快速定位思路，不要重新發明輪子。  
> 不綁特定題目，適用於任何前端限制場景。

---

## 一句話核心

**前端限制 ≠ 後端限制。前端的話不可信，直接跟後端說。**

---

## 觸發條件（什麼時候該想到這招）

- UI 上某個操作被擋住、變灰、無反應
- 表單有某個欄位被 `disabled`、`hidden`、或值被鎖死
- 某個選項在 dropdown 裡消失了
- 點了按鈕但「什麼都沒發生」
- 題目敘述強調「UI 只允許 X」→ 這幾乎是明示

---

## 決策流程

```
UI 操作被擋
    ↓
打開 DevTools → 找到擋你的 JS 程式碼
    ↓
確認：這個檢查在瀏覽器跑，還是伺服器跑？
    ├─ 瀏覽器 JS → 前端限制 → 可繞過 ✅
    └─ 伺服器回傳錯誤 → 後端限制 → 要另找破口
    ↓
找到 API 端點和請求格式
    ↓
自己送手工請求（主控台 fetch / curl / Burp）
    ↓
觀察伺服器反應 → 調整 payload → 拿 flag
```

---

## 四個「找 API」的快招

| 方法 | 操作 | 適合場景 |
|------|------|----------|
| **Network tab** | F12 → Network → 正常操作一次 → 右鍵請求 → Copy as fetch | 最快，直接拿骨架 |
| **函式 toString()** | 主控台打 `submitXxx.toString()` | 知道函式名時 |
| **搜源碼** | Cmd+F 搜 `fetch` / `/api` / `/check` / `XMLHttpRequest` | 源碼可讀時 |
| **右鍵按鈕** | 右鍵 → 檢查 → 看 `onclick="xxx()"` | 找到函式入口 |

---

## 手工請求範本

```js
// 基本 POST JSON（貼進主控台改改就能用）
fetch('/api/endpoint', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value_ui_would_not_let_you_set' })
}).then(r => r.json()).then(console.log)

// 帶自訂 header
fetch('/api/endpoint', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Custom-Header': 'value'
  },
  body: JSON.stringify({ /* ... */ })
}).then(r => r.json()).then(console.log)
```

> Cookie / Session 在同域請求下會自動帶，不需要額外設定。

---

## 常見「後端沒檢查」的失誤類型

| 前端擋了什麼 | 後端實際上沒驗證 | 攻擊結果 |
|---|---|---|
| 兩個欄位不能相同 | 只驗格式 | 送相同值 |
| 數值不能為負 | 只驗是數字 | 負數操作 → 邏輯反轉 |
| 只允許特定選項 | 只驗欄位存在 | 送任意值 |
| 不能修改 `role` 欄位 | 沒有伺服器端授權 | 提權 |
| 某些按鈕/操作被隱藏 | 只靠前端 CSS/JS 過濾 | 直接呼叫 API |
| 上傳只允許 .jpg | 只看副檔名 | 改 Content-Type 送 .php |

---

## 心法對照

| 心法 | 在這裡的意思 |
|------|-------------|
| **Trust Boundary** | 前端不是信任邊界，伺服器才是。UI 能做的事，API 不一定會擋。 |
| **Drop a Layer** | UI 層卡住 → 往下一層：直接操作 HTTP 請求。 |
| **Anchor** | 從「我知道怎麼用的東西」出發：fetch、curl、Burp，這些都能發請求。 |
| **Errors are Oracles** | 伺服器回傳的 400/422/500 告訴你後端真正驗了什麼。每個錯誤 = 一條線索。 |

---

## 快速查核表（遇題時掃一遍）

- [ ] UI 擋住的操作，是前端 JS 擋，還是伺服器擋？
- [ ] 後端 API 的完整請求格式是什麼？（端點、method、body 結構）
- [ ] 後端實際驗了哪些欄位？有沒有漏掉某個「顯而易見」的檢查？
- [ ] 我要送的「UI 不允許但後端會接受」的 payload 長怎樣？

---

## 通用 SOP：遇到「操作型 / 有前端限制」的題目

1. **先搞懂贏的條件**：這題要我做出什麼結果？
2. **用官方 UI 試著達成**：能做到就做；做不到就記下「卡在哪個限制」。
3. **判斷限制是前端還是後端**：
   - DevTools 讀前端程式碼（源碼 / `函式名.toString()`）
   - 看那個擋你的檢查是寫在**瀏覽器 JS** 裡，還是伺服器回應裡
   - **前端的限制 = 可繞過**
4. **從前端偷 API 用法**：
   - 右鍵按鈕看 `onclick`
   - 搜 `fetch` / `/api` / 端點名
   - 懶人法：Network tab → 正常操作一次 → 右鍵請求 → Copy as fetch
5. **自己送手工請求**：把 body 改成「UI 不讓你做、但後端會接受」的內容
6. **觀察回應，迭代調整**

---

*授權用途：CTF / 授權滲透測試 / 自己的機器。*
