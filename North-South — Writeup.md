# North-South — Writeup（Geo-based Routing 繞過）
 
> 賽事：CyLab Academy（picoCTF 2026 風格）
> 類別：**Web Exploitation**（GeoIP 路由繞過 / 改變來源地理位置）
> Flag：`academy{g30_b453d_r0u71n9_5e58725f}`
 
---
 
## TL;DR
 
Nginx 用 **geoip2** 模組看「你的 IP」判斷國家：只有國碼 = **IS（冰島）** 才被路由到藏 flag 的伺服器。解法：**讓請求真的從冰島 IP 連出去**（用有冰島節點、且不限 port 的瀏覽器 VPN）。核心技能不是「用 VPN」，是**讀懂 config 推出「要從冰島連」+ 一步步排除行不通的方法**。
 
---
 
## 🧭 決策鏈：發現什麼 → 所以下一步（這題最值得學的）
 
每一步都是「上一步的結果」把你推向下一步。這就是「怎麼知道下一步」的答案。
 
| # | 我做了什麼 | 發現什麼（情報） | 所以下一步 |
|---|-----------|-----------------|-----------|
| 1 | 讀題目：geo-based routing | 地理限制 → 跟「你在哪」有關 | 找源碼看它怎麼判斷地理 |
| 2 | 讀 nginx config | `geoip2` 看 IP → 國碼 `IS` 才進 flag 伺服器 | 讓我的 IP 看起來是冰島 |
| 3 | `IS` 是哪國？ | **Iceland 冰島** | 需要冰島 IP |
| 4 | 試 `X-Forwarded-For: 冰島IP` | 回 `No flag in this region`（失敗） | header 沒用 → 想為什麼 |
| 5 | 回頭看 config | geoip2 **沒設 real_ip** → 看的是**真實連線 IP**，不是 header | 必須「真的」換連線 IP |
| 6 | 試 Tor 冰島出口 | `ipinfo.io` 回 `IS`（偽裝成功！） | 用這條線路打目標 |
| 7 | Tor 打目標 | `ipinfo`(443) 通、目標(高 port) 擋 | 推論：Tor 出口政策擋非標準 port |
| 8 | 認清 Tor 限制 | Tor 改得了國家，但擋怪 port | 換「不限 port 的冰島工具」 |
| 9 | 用瀏覽器 VPN（冰島） | 不限 port + 冰島 IP | 瀏覽器開目標 → **flag** 🚩 |
 
> **核心：不是一開始就知道用 VPN。是每一步的失敗/成功,把你一路逼到「不限 port 的冰島 VPN」這個答案。**
 
---
 
## 1. 讀 Nginx Config：贏的條件全寫在裡面
 
```nginx
load_module .../ngx_http_geoip2_module.so;
http {
    geoip2 /etc/nginx/GeoLite2-Country.mmdb {       # 用 GeoIP 庫，從 IP 查國家
        $geoip2_data_country_code default=ZZ country iso_code;
    }
    upstream north { server 127.0.0.1:8000; }        # 無聊伺服器
    upstream south { server 127.0.0.1:9000; }        # 藏 flag 的伺服器
    server {
        listen 80;
        location / {
            if ($geoip2_data_country_code = IS) {    # 國碼 = IS 才進 south
                proxy_pass http://south;              # south = flag
            }
            proxy_pass http://north;                  # 其他人 → north
        }
    }
}
```
 
讀出：進 flag 伺服器要 `國碼 = IS`；`IS` = **冰島**；國碼來自 `geoip2` 查**你的 IP**。→ **贏的條件：讓伺服器以為你的 IP 來自冰島。**
 
---
 
## 2. 嘗試①：偽造 header（失敗，但有情報）
 
```sh
curl -H "X-Forwarded-For: 82.221.128.1" http://xebec.cylabacademy.net:PORT/
```
 
**curl 參數詳解：**
- `-H "名: 值"` → 加一個**自訂 HTTP header**。這裡偽造 `X-Forwarded-For`（代理常用來傳「真實客戶端 IP」的 header），填一個冰島 IP（`82.221.x.x` = 冰島 Thor DC）。
**結果**：`No flag in this region!` → 失敗。
**情報**：geoip2 **預設看真實連線 IP(`$remote_addr`)**，且 config 沒 `real_ip_header` → **header 偽造無效**。→ 必須真的換連線 IP。
 
---
 
## 3. 嘗試②：Tor 冰島出口（偽裝成功，但卡 port）
 
```sh
# 寫 torrc：SOCKS port 9050、出口只用冰島、強制
printf 'SocksPort 9050\nExitNodes {is}\nStrictNodes 1\n' > /etc/tor/torrc
tor -f /etc/tor/torrc &
 
# 驗證出口在冰島
curl --socks5-hostname 127.0.0.1:9050 https://ipinfo.io/country
# → IS（偽裝成冰島成功！）
 
# 打目標
curl --socks5-hostname 127.0.0.1:9050 http://xebec.cylabacademy.net:PORT/
```
 
**curl 參數詳解：**
- `--socks5-hostname 127.0.0.1:9050` → 讓 curl **走本機 9050 的 SOCKS5 代理**（就是 Tor）。
  - 為什麼用 `-hostname` 版（不是 `--socks5`）？→ 讓**代理去解析 DNS**，而不是你本機先解析。這樣 DNS 查詢也走 Tor，不會洩漏你真實的位置/DNS。
- `https://ipinfo.io/country` → 一個回傳「你看起來在哪個國家」的服務。**先驗證出口國家** = IS，再打目標（不然白打）。
**結果**：`ipinfo`(走 443)通、回 `IS`；但打目標失敗：
```
Have tried resolving or connecting to address at 3 different places. Giving up.
```
**情報**：目標在**非標準高 port**(28618/31265，一直換)。**Tor 出口政策多半只放行 80/443**，拒絕高 port。→ Tor 載不動,換工具。
 
### 重試時用到的 curl 參數
```sh
for i in $(seq 1 40); do
  r=$(curl -s -i --max-time 20 --socks5-hostname 127.0.0.1:9050 http://目標/ 2>/dev/null)
  if [ -n "$r" ]; then echo "第 $i 次通"; echo "$r"; break; fi
  printf '.'
done
```
- `-s`（silent）→ 不印進度條/雜訊，畫面乾淨
- `-i`（include）→ **回應連 HTTP headers 一起印**，這樣就算 body 空也看得到狀態碼，確認有沒有真的連上
- `--max-time 20` → 單次最多等 20 秒就放棄（Tor 慢，給它時間但不要卡死）
- `-o out.html` → 把回應**存到檔案**（配合 `[ -s out.html ]` 判斷有沒有內容）
- `2>/dev/null` → 把 Tor 的 notice 雜訊（stderr）丟掉
---
 
## 4. 成功：瀏覽器 VPN（不限 port）
 
VPN 出口**放行所有 port**，不像 Tor 有政策限制。用**免費瀏覽器 VPN 擴充**（有冰島）：
 
1. Chrome/Edge 裝 **Urban VPN**（或 VeePN）
2. 國家選 **Iceland 🇮🇸**
3. 同瀏覽器開 `ipinfo.io` → 確認 Country = Iceland
4. 確認後瀏覽器開目標網址（用**當前** port）→ flag 出現 🚩
`academy{g30_b453d_r0u71n9_5e58725f}`
 
> ⚠️ 瀏覽器 VPN 只路由**瀏覽器**流量 → 要用**瀏覽器訪問**，不能用 curl（curl 走的是系統網路，沒被擴充 VPN 接管）。
 
---
 
## 5. 工具對照（為什麼最後選瀏覽器 VPN）
 
| 工具 | 到冰島? | 連非標準 port? | 免費? | 結果 |
|------|--------|---------------|-------|------|
| X-Forwarded-For | — | — | ✅ | ❌ geoip 看真 IP |
| Tor（冰島出口） | ✅ | ❌（出口政策擋高 port） | ✅ | ❌ 載不動怪 port |
| 免費 proxy | ❌（冰島太小） | — | ✅ | ❌ 沒節點 |
| **瀏覽器 VPN** | ✅ | ✅（不限 port） | ✅ | ✅ **成功** |
| 付費 VPN（Mullvad/PIA） | ✅ | ✅ | 💰 | ✅（備案） |
 
---
 
## 6. 踩過的坑（下次省時間）
 
- **WSL 磁碟超小(136MB)** → 裝 tor 前先清空間(`apk del python3`、`rm -rf /var/cache/apk/*`)
- **`--ExitNodes '{is}'` 背景啟動可能沒生效** → 用 torrc 檔 `tor -f` 比較穩
- **`--socks5` vs `--socks5-hostname`** → 用 `-hostname` 讓 DNS 也走 proxy
- **題目 port 一直換** → 每次打前先抓**當前** port
- **Tor 出口政策** → 打非標準 port 時 Tor 常不行，直接上 VPN
---
 
## 7. 心法總結
 
| 心法 | 這題怎麼用 |
|------|-----------|
| **決策鏈：發現→下一步** | 每一步的結果決定下一步（見上面的表） |
| **讀源碼找贏的條件** | config 直接說「國碼 = IS 才進 flag」 |
| **Trust Boundary** | geoip 信任「你的 IP」判位置 → 換 IP 騙它 |
| **先試便宜招,失敗也是情報** | header 失敗 → 學到它看真 IP |
| **工具限制要認清** | Tor 改得了國家但擋非標準 port → 換 VPN |
| **答案簡單 ≠ 過程簡單** | 「用 VPN」是答案,但推到這答案 + 排除錯路才是技能 |
 
> **核心：這題考的不是會不會用 VPN,是能不能從 config 推出『要從冰島連』,再一步步排除行不通的工具。**
 
---
 
## 8. 修復建議（防守方）
 
- 地理限制別只靠 `$remote_addr`（可被 VPN/proxy 繞）；敏感資源不該只用地理當唯一門檻
- 有前置 proxy 時正確設定 `real_ip` 並**只信任自己 proxy 的 XFF**
- 地理路由當「方便」用,不要當「安全」用
---
 
*授權用途：CTF 學習 / 授權滲透測試 / 自己的機器。*
 
