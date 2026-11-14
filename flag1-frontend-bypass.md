# picoCTF 2025 — Pachinko Flag1：前端繞過完全解析

> 類別：Web Exploitation（Client-Side Bypass）  
> 難度：Medium  
> Flag：`academy{p4ch1nk0_f146_0n3_e947b9d7}`

---

## 一句話核心

**前端 UI 故意擋你做某個操作，但後端伺服器根本沒檢查 → 繞過前端、自己直接送請求給後端。**

---

## 題目機制

- 畫面是一個 **NAND Simulator**：拖拉節點、連線建電路。
- 8 個節點：**上排 5/6/7/8 = 4 個輸入**（伺服器隨機給值），**下排 1/2/3/4 = 4 個輸出**（要填的答案）。
- 目標：讓每個輸出 = 對應輸入的**反相**（`Flip the outputs!`）。
- 按 Submit → 前端把電路包成 JSON，POST 到 `/check`；對了就回傳 flag1。

### 後端判定

- `/check` 收到 `{circuit:[{input1, input2, output}, ...]}`
- 唯一驗證是 `checkInt`：每個值是 **1~0xFFFF 的整數**即可
- **沒有**檢查 `input1 != input2`
- 電路跑完，若輸出 == 目標（反相）→ `memory[0x1000] == 0x1337` → 回 flag1

---

## 基礎知識：NAND 與反相

| A | B | NAND(A,B) |
|---|---|-----------|
| 0 | 0 | 1 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

**關鍵恆等式：`NAND(x, x) = NOT(x)`**

- NAND(0, 0) = NOT(0 AND 0) = NOT(0) = **1**
- NAND(1, 1) = NOT(1 AND 1) = NOT(1) = **0**

把 NAND 閘的兩個輸入接到**同一個節點**，它就變成 NOT 閘。

---

## 完整決策鏈（從零到 flag 的真實思路順序）

### Step 1：理解目標

讓每個輸出 = 輸入的反相 → 想到 `NAND(x,x) = NOT(x)`

### Step 2：在 UI 嘗試

嘗試把節點 5 同時接到 NAND 閘的 input1 和 input2 → **撞牆**：第二次點擊完全沒反應。

### Step 3：找到前端的鎖

打開 DevTools → 讀 JS 源碼，找到 `handleNodeClick()`：

```js
function handleNodeClick(node) {
  const existingInputs = currentGate.inputs.map(i => i.id);

  // 如果這個 node 已經在 inputs 裡，就不加
  if (!existingInputs.includes(selectedNode.id)
      && existingInputs.length < 2) {
    currentGate.inputs.push(selectedNode);
  }
}
```

`!existingInputs.includes(selectedNode.id)` → 同一個節點不准接兩次。**這是純前端的 JavaScript 檢查。**

### Step 4：判斷——前端限制 vs 後端限制

| | 前端（你的瀏覽器） | 後端（伺服器） |
|---|---|---|
| 跑在哪 | 你的電腦 | 別人的電腦 |
| 能改嗎 | 可以隨意繞過 | 無法直接修改 |
| 有檢查 input1 ≠ input2？ | ✅ 有（handleNodeClick） | ❌ 沒有 |

**結論：後端沒有同樣的檢查 → 可以繞過！**

### Step 5：找出 API 格式

在主控台執行 `submitCircuit.toString()`，看到：

```js
fetch('/check', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ circuit: gates })
})
```

### Step 6：自己送請求

知道了端點是 `POST /check`，body 格式是 `{circuit: [{input1, input2, output}, ...]}`，直接在主控台送出 UI 不讓你組的電路。

---

## Payload（貼進題目頁主控台執行）

```js
fetch('/check', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ circuit: [
    { input1: 5, input2: 5, output: 1 },  // NAND(5,5) = NOT(5) → 輸出節點1
    { input1: 6, input2: 6, output: 2 },  // NAND(6,6) = NOT(6) → 輸出節點2
    { input1: 7, input2: 7, output: 3 },  // NAND(7,7) = NOT(7) → 輸出節點3
    { input1: 8, input2: 8, output: 4 }   // NAND(8,8) = NOT(8) → 輸出節點4
  ]})
}).then(r => r.json()).then(console.log)
// → { flag: "academy{p4ch1nk0_f146_0n3_e947b9d7}", status: "success" }
```

**節點對應**：輸出 1–4 ↔ 輸入 5–8（`output + 4 = input`）

---

## 心法回顧

| 心法 | 這題怎麼用 |
|------|-----------|
| **Trust Boundary** | 前端 JS 不是信任邊界。`handleNodeClick` 擋了，後端 `/check` 沒擋。 |
| **Drop a Layer** | UI 層卡住 → 往下一層：直接操作 HTTP 請求。 |
| **Anchor** | 從 `NAND(x,x) = NOT(x)` 出發，向外構建解法。 |
| **Errors are Oracles** | 後端回傳的錯誤幫你確認 body 格式。 |

---

## 延伸閱讀

- [Flag2 完整解析](./flag2-von-neumann-exploit.md) — 同一個 instance 的困難版，打穿軟體 CPU
- [通用心法卡](./playbook-frontend-bypass.md) — 前端限制繞過的可複製 SOP
