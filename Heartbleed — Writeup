# Heartbleed — Writeup
 
> 賽事：CyLab Academy（picoCTF 2026 風格）
> 類別：Reverse Engineering / Binary（記憶體 over-read + 演算法逆向）
> 給的檔案：`system.out`（ELF 64-bit，**未 strip**，原始碼 `heartbleed.c`）
> 連線：`nc xebec.cylabacademy.net <port>`
 
---
 
## TL;DR（一句話）
 
程式把一個秘密放在緩衝區裡，顯示密碼時卻**用「你宣稱的長度」而不是「實際長度」**來印 → **heartbleed over-read**，洩漏旁邊的秘密。但最後關卡要的不是秘密字串，而是**秘密經過 XOR 0xAA + djb2 雜湊後的「數字」**。讀懂 `make_secret`/`hash` 兩個函式、自己重算那個數字交回去，拿 flag。
 
---
 
## 1. 黑箱偵查
 
### 1-1 先 nc 連上去，看它怎麼互動
 
```
Please set a password for your account:
> AAAA
How many bytes in length is your password?
> 100
You entered: 100
Your successfully stored password:
65 65 65 65 10 0 0 ... 105 85 98 104 56 49 33 106 42 104 110 33 -86 0 ...
Enter your hash to access your account!
```
 
兩個觀察：
- 它把「stored password」**用十進位數字一個 byte 一個 byte 印出來**（`65` = `A`）。
- 第二關要「hash」。
### 1-2 strings 挖線索
 
```bash
strings system.out
```
 
關鍵字串：
```
heartbleed.c        ← 原始碼檔名，直接暴露漏洞種類
make_secret
obf_bytes           ← obfuscated bytes（被混淆的資料）
hash
flag.txt
Could not open flag.txt
Please set a password for your account:
How many bytes in length is your password?
Enter your hash to access your account!
```
 
> **心法：有檔名 `heartbleed.c` = 這題考記憶體 over-read（讀過頭，洩漏旁邊記憶體）。`make_secret`/`obf_bytes`/`hash` = 秘密是算出來的。**
 
---
 
## 2. 觸發 Heartbleed 洩漏
 
### 2-1 概念
 
Over-read ≠ Overflow：
- **Overflow（寫爆）**：塞超過緩衝區 → 蓋掉旁邊記憶體。
- **Over-read（讀過頭）**：叫它讀/印比你實際給的更多 → **把旁邊記憶體吐出來**。
Heartbleed 是後者。做法：**宣稱長度很大、實際密碼很短。**
 
### 2-2 操作
 
```
Please set a password for your account:
> AAAA          ← 密碼給短（4 bytes）
How many bytes in length is your password?
> 100           ← 長度宣稱很大
```
 
它就會印約 90 個 byte（上限），前 4 個是你的 `AAAA`，**後面是記憶體裡剛好放在旁邊的秘密**。
 
### 2-3 讀洩漏
 
```
65 65 65 65    ← "AAAA"（你的密碼，A=65）
10             ← 換行
0 0 0 ...       ← 空白
105 85 98 104 56 49 33 106 42 104 110 33   ← 秘密！
-86            ← 0xAA（後面會發現這是 XOR 金鑰）
```
 
把那串數字轉 ASCII：
 
| 105 | 85 | 98 | 104 | 56 | 49 | 33 | 106 | 42 | 104 | 110 | 33 |
|-----|----|----|-----|----|----|----|-----|----|----|-----|----|
| i | U | b | h | 8 | 1 | ! | j | * | h | n | ! |
 
**secret = `iUbh81!j*hn!`**
 
### 2-4 卡關：直接交秘密失敗
 
在 `Enter your hash` 輸入 `iUbh81!j*hn!`：
 
```
system.out: heartbleed.c:69: main: Assertion `1 == 0' failed.
Aborted (core dumped)
```
 
→ 它要的不是這個字串。黑箱到此撞牆，**該開白箱讀程式碼了**。
 
---
 
## 3. 白箱：用 Ghidra 反組譯
 
### 3-1 安裝與載入（Windows）
 
1. 裝 **JDK 21+**（Adoptium Temurin）。
2. 下載 **Ghidra**（官方 GitHub releases），解壓、跑 `ghidraRun.bat`。
3. New Project → Import File → 選 `system.out` → 雙擊打開 CodeBrowser → 分析時按 Yes / Analyze。
### 3-2 導航心法：從 main 出發，跟著「呼叫」走
 
> **怎麼知道要看哪個函式？** 不是用猜的。從 `main` 讀起 → 看它呼叫誰、拿什麼跟你的輸入比對 → 比對對象就是關鍵函式。
 
左邊 `Symbol Tree → Functions`，因為這支**沒 strip**，函式名都在：`main`、`make_secret`、`hash`。
 
### 3-3 make_secret（第一層：XOR 解混淆）
 
```c
void make_secret(long out) {
    for (i = 0; obf_bytes[i] != '\0'; i++) {
        out[i] = obf_bytes[i] ^ 0xaa;   // 每個 byte XOR 0xAA
    }
    out[12] = 0;
    return hash(out);                    // 解出來的字串再丟去 hash
}
```
 
→ 秘密 = `obf_bytes` XOR `0xAA`。（這解釋了洩漏時看到的 `-86` = `0xAA`，就是金鑰。）
 
### 3-4 hash（第二層：djb2）
 
```c
ulong hash(char *s) {
    ulong h = 0x1505;          // 5381
    while (*s) {
        h = h * 0x21 + *s;     // 0x21 = 33 → h = h*33 + c
        s++;
    }
    return h;                  // 回傳一個數字
}
```
 
→ 這是經典 **djb2** 雜湊，輸出是**數字**，不是字串。這解釋了為什麼交字串會失敗。
 
### 3-5 main（完整邏輯）
 
```c
local_110 = calloc(0x5a, 1);                       // 90 bytes 緩衝區
for (i = 0; i < 0xd; i++)
    local_110[i + 0x3c] = obf_bytes[i] ^ 0xaa;     // 秘密放在偏移 0x3c = 60
 
puts("Please set a password...");
fgets(password, 0x32, stdin);
strcpy(local_110, password);                       // 密碼複製到偏移 0
 
puts("How many bytes...");
param1 = atoi(length_input);                       // param1 = 你宣稱的長度
for (i = 0; i <= param1 && i < 0x5a; i++)          // ← 漏洞：用 param1 不是實際長度！
    printf("%d ", local_110[i]);                   // 一個 byte 一個 byte 印
 
puts("Enter your hash...");
your_num = strtoul(hash_input, &end, 10);          // 你的輸入當「十進位數字」
if (end == hash_input) {                           // 沒有數字
    printf("No digits were found");
    assert(1 == 0);                                // ← crash（line 69）
}
secret_hash = make_secret(buf);                    // 算出正確答案（數字）
if (secret_hash == your_num) {                     // 比對：數字 == 數字
    fopen("flag.txt"); ... printf("%s", flag);     // 對了 → 印 flag
}
```
 
三個關鍵：
1. **漏洞那行**：`for (i = 0; i <= param1 ...)` → 印幾個 byte 看 `param1`（你宣稱的長度），不看密碼實際多長。這就是 heartbleed。
2. **秘密位置**：`local_110[i + 0x3c]` → 偏移 60，對應洩漏時秘密出現的位置。
3. **最後比對**：`strtoul(..., 10)` 把你的輸入當**十進位數字**，跟 `make_secret` 的回傳**數字**比。
---
 
## 4. 資料流追蹤：為什麼要拿 obf_bytes
 
目標 = `make_secret` 的回傳值。往回追它依賴什麼：
 
```
我要的數字
   ↑ 誰算的
make_secret
   ↑ 讀什麼
obf_bytes ^ 0xAA → djb2
```
 
`make_secret` 用到的東西分兩種：
- **常數**（`0xAA`、`5381`、`33`）→ 程式碼裡看得到，直接抄。
- **具名資料 `obf_bytes`** → 值存在 `.rodata`，程式碼看不到 → **必須去挖**。
### 在 Ghidra 挖 obf_bytes
 
雙擊程式碼裡的 `obf_bytes` → 中間 Listing 跳到 `.rodata`：
 
```
00102008  c3   [0]
00102009  ff   [1]
0010200a  c8   [2]
...
00102013  8b   [11]
00102014  00   [12]   ← 結尾 0，迴圈讀到這停
```
 
**obf_bytes = `c3 ff c8 c2 92 9b 8b c0 80 c2 c4 8b`**（前 12 個，第 13 個 `00` 是結尾）。
 
> 補充：這些是 **十六進位（hex）**。辨認訊號：Ghidra 標 `C3h`（h 結尾）、程式碼寫 `0xc3`（0x 開頭）、含 a~f 字母。1 byte = 2 個 hex 數字，所以低階工具都用 hex。`0xC3` = `195`（十進位），同一個值兩種寫法。
 
---
 
## 5. 重算：把演算法照抄一遍
 
### 規劃思路
 
重算腳本不是「發明」，是「翻譯」——程式的計算流程就是腳本的規格書：
 
1. 目標 = make_secret 的輸出。
2. make_secret 的步驟：XOR 0xAA → djb2。照**同樣順序**翻成 Python。
3. 準備輸入：obf_bytes（挖出來的）+ 常數（程式碼裡有）。
4. 每步 print 驗證。
### recompute.py
 
```python
# obf_bytes（從 .rodata 挖出來的 12 個 byte）
obf = [0xc3, 0xff, 0xc8, 0xc2, 0x92, 0x9b, 0x8b, 0xc0, 0x80, 0xc2, 0xc4, 0x8b]
 
# 第一層：抄 make_secret 的 XOR 0xAA
secret = "".join(chr(b ^ 0xAA) for b in obf)
print("secret =", secret)          # iUbh81!j*hn!
 
# 第二層：抄 hash 的 djb2
h = 0x1505                          # 起始值 5381
for ch in secret:
    h = (h * 33 + ord(ch)) & 0xFFFFFFFFFFFFFFFF   # 64-bit 溢位回繞
print("hash   =", h)               # 15237662580160011234
```
 
執行：
```bash
python3 recompute.py
```
 
輸出：
```
secret = iUbh81!j*hn!
hash   = 15237662580160011234
```
 
### 對照表（腳本 = 程式）
 
| recompute.py | Ghidra 程式碼 | 函式 |
|--------------|--------------|------|
| `b ^ 0xAA` | `obf_bytes[i] ^ 0xaa` | make_secret |
| `h = 0x1505` | `h = 0x1505` | hash |
| `h * 33 + ord(ch)` | `h * 0x21 + c` | hash |
| `& 0xFFFF...` | CPU 64-bit 暫存器自動溢位 | hash |
 
---
 
## 6. 拿 Flag
 
重新連線，在 `Enter your hash` 輸入那個**數字**：
 
```
Please set a password for your account:
> AAAA
How many bytes in length is your password?
> 4
Enter your hash to access your account!
> 15237662580160011234
```
 
`strtoul` 解析成數字 → `== make_secret()` → 開 `flag.txt` → 印出 flag 🚩
 
> 註：因為 `obf_bytes` 寫死在檔案裡、演算法固定，這個數字**永遠不變**。讀懂程式後其實**不用 heartbleed 洩漏也能純離線算出答案**；洩漏是另一條獨立驗證路線。
 
---
 
## 7. 兩種解法對照
 
| 路線 | 怎麼拿到 secret 字串 | 之後 |
|------|---------------------|------|
| **A. 黑箱 heartbleed** | over-read 洩漏記憶體 → 轉 ASCII | 再對 secret 做 djb2 |
| **B. 白箱純離線** | 讀 binary 的 obf_bytes → XOR 0xAA | 同上 |
 
兩條都通，最後都要**自己算 djb2 數字**才能過關。
 
---
 
## 8. 帶走的心法
 
| 心法 | 這題怎麼用 |
|------|-----------|
| **Errors/Output are Oracles** | 別猜，直接測 nc 看它吐什麼；`49`/`102` 的數字就是 byte 值 |
| **Trust the claimed length = Heartbleed** | 程式信任「宣稱的長度」而非「實際長度」→ over-read |
| **黑箱撞牆 → 開白箱** | 交字串失敗後，用 Ghidra 讀真正的邏輯 |
| **從 main 跟著呼叫走** | 怎麼知道看哪個函式？看 main 拿什麼跟你的輸入比對 |
| **Data flow 追蹤** | 要重算就追它依賴的輸入；常數現成、具名資料去挖 |
| **逆向 = 讀懂食譜自己煮** | 把演算法照抄成 Python 重算，讓電腦吐答案 |
 
> **核心：讀懂程式的瞬間，答案就已經決定了。你不是在猜，是在「照它做」。**
 
---
 
## 9. Ghidra 快速上手備忘
 
- 安裝：JDK 21+ → 解壓 Ghidra → `ghidraRun.bat`
- 載入：New Project → Import File → 雙擊 → Analyze（預設）
- 三欄：左 Symbol Tree（函式清單）、中 Listing（組語）、右 Decompiler（類 C）
- 讀函式：Symbol Tree → Functions → 雙擊 → 看右邊 Decompiler
- 看資料：雙擊程式碼裡的變數名（如 `obf_bytes`）→ 跳到 Listing 的資料區看 hex
---
 
*授權用途：CTF 學習 / 授權滲透測試 / 自己的機器。*
 
