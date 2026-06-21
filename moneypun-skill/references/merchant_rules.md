# MoneyPun — Merchant Rules / 商戶分類規則

This file contains classification logic that applies across any user's setup.
User-specific merchant lists should be added to `moneypun/config/accounts.json` rather than here.
本檔包含通用分類邏輯。用戶專屬的商戶清單應存放在 `config/accounts.json`，而非此檔。

---

## 1. Universal Merchant → Category Examples / 通用商戶對照示例

The following merchants appear globally and can be auto-categorised without asking the user.
以下商戶為全球通用品牌，可直接分類，無需問用戶。

| Merchant keyword (case-insensitive) 商戶關鍵字 | Category 分類 | Notes 備注 |
|-----------------------------------------------|--------------|------------|
| McDonald's / MCD | Dining 餐飲 | |
| KFC | Dining 餐飲 | |
| Starbucks | Dining 餐飲 | |
| Subway | Dining 餐飲 | |
| Uber Eats / DoorDash / Deliveroo | Dining 餐飲 | Food delivery 外賣 |
| Uber / Lyft / Grab / Bolt | Transport 交通 | |
| Apple / iTunes / App Store | Subscriptions 訂閱服務 | |
| Google / Google Play | Subscriptions 訂閱服務 | |
| Netflix | Subscriptions 訂閱服務 | |
| Spotify | Subscriptions 訂閱服務 | |
| Disney+ / Hulu / HBO Max | Subscriptions 訂閱服務 | |
| Amazon | Shopping 購物 | Unless it's Amazon Prime → Subscriptions |
| Airbnb | Travel & Entertainment 娛樂旅遊 | |
| Booking.com / Hotels.com / Agoda | Travel & Entertainment 娛樂旅遊 | |
| Trip.com / Expedia / Kayak | Travel & Entertainment 娛樂旅遊 | |
| Cathay / Singapore Airlines / Emirates | Travel & Entertainment 娛樂旅遊 | |
| PayPal | See §3 — passthrough 見 §3 |
| [Investment broker name] | internal_transfer 內部轉帳 | Exclude from expenses 排除出支出 |
| [Credit card repayment keyword] | internal_transfer 內部轉帳 | Exclude from expenses 排除出支出 |

**How to extend this list / 如何擴充**: During Phase 0 setup, ask the user which local merchants or subscription services they regularly use, and add them here or to `config/accounts.json`.

---

## 2. Category Decision Guide / 分類決策指引

When the merchant name alone is ambiguous, use these rules to decide:
當商戶名稱本身不夠清楚時，用以下規則判斷：

| Situation 情況 | Category 分類 | Reasoning 判斷理由 |
|---------------|--------------|-------------------|
| Supermarket / grocery store | Groceries 超市 | e.g. Walmart, Tesco, Carrefour |
| Pharmacy / chemist | Personal Care 個人護理 | e.g. Boots, CVS, Watsons |
| Hospital / clinic / dentist | Health 醫療 | |
| Online course / university | Education 學習 | |
| Gym / fitness | Personal Care 個人護理 | Or create a Fitness sub-category |
| Insurance premium | Insurance 保險 | Usually annual or monthly fixed amount |
| Rent / mortgage | Housing 住屋 | Usually a Standing Order |
| Electricity / gas / water / internet | Housing 住屋 | Utility bills |
| Unknown — user must confirm | Other 其他消費 (tentative) | Add to Phase 5 table |

---

## 3. eWallet Passthrough Merchants / 電子錢包泛稱商戶

When a bank notification shows only the eWallet name instead of the actual merchant (e.g. the bank shows "APPLEPAY" or "GOOGLEPAY MERCHANT XY" but not the real shop name), the actual merchant is unknown.

銀行通知只顯示電子錢包名稱而非真實商戶時（如「APPLEPAY」、「GOOGLEPAY MERCHANT XY」），無法自動分類。

**Rule / 規則**: Do **not** attempt extra Gmail searches for eWallet notification emails — results are inconsistent and waste tokens. Add these directly to the Phase 5 confirmation table.
不要嘗試額外搜尋 Gmail — 成效不穩定且浪費 token。直接加入 Phase 5 確認表。

| Common passthrough string 常見泛稱字串 | Action 處理 |
|---------------------------------------|------------|
| `APPLEPAY [MERCHANT]` | Ask user if merchant name is not recognisable 若商戶名不明則問用戶 |
| `GOOGLEPAY [MERCHANT]` | Same as above 同上 |
| `PAYPAL *[CODE]` | Ask user — PayPal codes don't map to merchants 問用戶，PayPal 代碼無法對應商戶 |
| Any eWallet string where real merchant is unclear | Phase 5 confirmation 加入確認表 |

---

## 4. Stored-Value / Transit Cards / 儲值卡 & 交通卡

Some users pay transit or small purchases via a stored-value card (e.g. contactless transit card, prepaid card). These create two types of bank charges:
部分用戶透過儲值卡（如交通卡、預付卡）乘車或小額消費，會產生兩種銀行扣帳：

| Transaction type 類型 | How it appears 銀行顯示 | Category 分類 |
|-----------------------|------------------------|--------------|
| Top-up / reload 增值 | `TOP-UP`, `RELOAD`, card name + amount | **internal_transfer** — do not double-count 勿重複計算 |
| Transit / purchase via card 用卡消費 | Transit operator or merchant name | Transport 交通 / Dining 餐飲 / etc. |

The top-up is a transfer of funds to the card; the actual spending happens at the point of use and typically does not generate a separate bank email. Count only the top-up (as a transfer), not the downstream spending, to avoid double-counting.
增值是把錢轉入卡內，真正消費在之後刷卡時產生，通常不再有銀行 email，不要重複計算。

---

## 5. Large Thread — Grep Patterns / 大 Thread Grep 提取模式

When a bank thread exceeds ~80 KB, `get_thread` hits the Gmail MCP token limit. Use the `Grep` tool directly on the temp file that the MCP saves to disk.
當銀行 thread 超過 ~80 KB，`get_thread` 會超過 token 上限。改用 Grep 工具直接搜尋 MCP 存到磁碟的 temp 檔。

**Temp file path format / 檔案路徑格式**:
`/var/folders/nt/.../tool-results/mcp-...-get_thread-TIMESTAMP.txt`

| What to extract 提取目標 | Grep pattern | Flags |
|--------------------------|-------------|-------|
| Local-currency amounts 本幣金額 | `[A-Z]{3}[0-9][0-9,\.]+` | `-o` (also matches foreign currencies 也會抓外幣) |
| Merchant name in HTML table 表格商戶名 | `<td>[A-Z][a-zA-Z0-9 \*\.\-\(\)\/]+</td>` | uppercase start 大寫開頭 |
| Merchant name (lowercase start) 小寫商戶 | `<td>[a-zA-Z][^<]{0,40}</td>` | catches URLs etc. 如網址 |
| Transaction date 交易日期 | `<td>[0-9]+ [A-Z][a-z]+</td>` | e.g. "15 Apr", "3 Jan" |

**Notes / 注意**:
- Bilingual emails repeat amounts twice (one per language). Take the first occurrence only.
  中英文雙語郵件金額出現兩次，取第一個即可。
- Each message in the thread needs separate Grep passes. Sort results by date to reconstruct order.
  Thread 中每個 message 分別 Grep，按日期排序重組。

---

## 6. Pre-authorisation vs Final Charge / 預授權 vs 最終扣帳

Some merchants (ride-hailing, hotels, car rentals, petrol stations) hold a pre-authorisation before settling the final amount. Both can appear in Gmail notifications on the same day.
部分商戶（網約車、酒店、租車、加油站）先發預授權，再發最終扣帳，同日可能出現兩筆。

**Rule / 規則**: Keep the final settled charge; discard the pre-auth.
保留最終扣帳，排除預授權。

| Keep 保留 | Discard 排除 |
|-----------|-------------|
| Final charge — lower or equal amount, arrives same day or next day | Pre-auth — often labelled "PENDING", "AUTH", "HOLD" |

If amounts differ, record the final charge amount and note the pre-auth in the `notes` field.
若金額不同，記錄最終扣帳金額，在 notes 補充預授權金額。

---

## 7. Salary / Payroll Handling / 薪資處理

Many banks send a "salary credited" notification without showing the amount.
許多銀行的「薪金已存入」通知不顯示金額。

Always add a row to the Phase 5 confirmation table:
Phase 5 確認表必定加一行：

> "Your salary was credited this month — what was the amount? / 本月薪資存入了，金額是多少？"

After confirmation, set: `type: "income"`, `category: "Salary"`, `source_account` = the receiving bank account.
確認後設定：`type: "income"`, `category: "Salary"`, `source_account` 為收款帳戶。
