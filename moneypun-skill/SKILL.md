---
name: moneypun
description: >
  MoneyPun personal finance monthly pipeline. Trigger when the user says
  "run MoneyPun", "做月結", "跑 X 月財務報告", "generate transactions",
  "整理本月支出", or any similar request to process monthly spending.
  Fetches bank / credit card / eWallet Gmail notifications, injects Standing
  Orders, resolves low-confidence items with the user BEFORE generating output,
  then produces YYYY-MM_transactions.json and an HTML report.
  Pipeline base path: moneypun/ (relative to the user's project folder)
---

# MoneyPun Pipeline

## Quick Reference / 快速索引

| Resource | Path |
|----------|------|
| Merchant rules, large-thread handling, eWallet patterns | `references/merchant_rules.md` |
| Output folder 輸出目錄 | `moneypun/output/` |
| Category list 分類清單 | `moneypun/config/categories.json` |
| Account list 帳戶清單 | `moneypun/config/accounts.json` |
| Standing Orders | `moneypun/config/standing_orders.json` |
| User preferences 用戶偏好 | `moneypun/config/user_prefs.json` |

---

## PHASE 0 — First-time Setup / 首次設定

**Trigger / 觸發條件**: `moneypun/config/user_prefs.json` does not exist.

Ask the user all three questions at once (don't split into separate messages):
一次過問以下三件事（不要分開問）：

1. **Accounts & notification status / 帳戶 & 通知狀態**
   List all your bank accounts and credit cards. For each one, does your bank send a Gmail notification when you make a purchase?
   列出你所有銀行帳戶和信用卡。每張卡有沒有在 Gmail 收到消費通知？
   Cards without email notifications will need a monthly statement upload to capture transactions.
   沒有通知的卡每月需要上傳月結單補數。

2. **eWallet sources / 電子錢包**
   Which eWallets or mobile payment apps do you use? (e.g. Apple Pay, Google Pay, PayPal, Venmo, local apps)
   你用哪些電子錢包或手機支付？Gmail 有收到通知嗎？

3. **Category confirmation / 分類確認**
   Do the default categories work for you, or do you want to add/rename any?
   默認分類是否適用？有無要新增或改名？
   Default / 默認: Dining, Transport, Shopping, Groceries, Travel & Entertainment, Health, Education, Subscriptions, Housing, Insurance, Personal Care, Pets, Other.

After receiving answers, write `moneypun/config/user_prefs.json`:
收到回覆後建立設定檔：

```json
{
  "home_currency": "HKD",
  "accounts_with_notifications": [
    { "id": "BankA_Credit", "display_name": "Bank A Credit Card", "sender": "alerts@banka.com" },
    { "id": "BankB_Debit",  "display_name": "Bank B Debit",       "sender": "notify@bankb.com" }
  ],
  "accounts_no_notifications": [
    { "id": "BankC_Credit", "display_name": "Bank C Credit Card — statement only" }
  ],
  "ewallet_sources": ["ApplePay", "PayPal"],
  "investment_accounts": ["Broker_X"],
  "custom_categories": []
}
```

---

## PHASE 1 — Pre-run Checklist / 執行前問卷

**Every run, before any Gmail fetch.** Ask these four questions in one message:
**每次執行，在 Gmail 搜尋之前**，一次問以下四題：

> **MoneyPun Pre-run Checklist / 執行前確認**
>
> 1. Which month to process? / 要跑幾月？ (e.g. 2026-07)
> 2. Current investment / brokerage account balance?
>    本月投資或證券帳戶餘額是多少？
> 3. Do you have any monthly statements to upload? (especially cards with no email notifications)
>    有月結單要上傳嗎？（特別是沒有 email 通知的卡）
> 4. Any manual items to add? (cash purchases, peer-to-peer received, reimbursements, etc.)
>    有現金或其他要手動加的項目嗎？

**Why the statement matters / 月結單的作用**: A statement covers (a) cards that don't send email notifications, (b) actual foreign-currency exchange rates charged, (c) eWallet merchant identification that doesn't appear in bank emails — saving significant guesswork and back-and-forth later.
月結單可以補全（a）無通知卡的交易、（b）外幣實際匯率、（c）電子錢包泛稱商戶，省去大量後期追問。

Proceed to Phase 2 only after receiving answers.
收到回覆後才開始 Phase 2。

---

## PHASE 2 — Inject Standing Orders / 注入固定支出

Read `moneypun/config/standing_orders.json`. For every entry where `enabled: true`, inject a transaction on its `day_of_month`. Set `confidence: "confirmed"` — no email verification needed.

---

## PHASE 3 — 5-Route Gmail Search / 5 路 Gmail 搜尋

Build each query using the sender addresses from `user_prefs.json`. Replace YYYY/MM with the target month.
查詢條件從 user_prefs.json 的寄件人地址動態組合，YYYY/MM 替換為目標月份。

| Route | Query structure | Purpose |
|-------|----------------|---------|
| S1_bank | `from:(sender1 OR sender2 OR ...) after:YYYY/MM/01 before:YYYY/MM/30` — senders from `accounts_with_notifications` | Bank & card debit notifications 銀行扣帳通知 |
| S2_merchant | `from:(booking.com OR airbnb OR amazon OR apple OR google OR uber OR netflix OR spotify) after:YYYY/MM/01 before:YYYY/MM/30` — expand with any merchants the user frequently shops at | Merchant confirmation emails 商戶確認信 |
| S3_receipt | `(subject:receipt OR subject:invoice OR subject:order OR subject:confirmation) after:YYYY/MM/01 before:YYYY/MM/30` | Receipts & order confirmations 收據 |
| S4_ewallet | `from:(sender3 OR sender4 OR ...) after:YYYY/MM/01 before:YYYY/MM/30` — senders from `ewallet_sources` in user_prefs | eWallet notifications 電子錢包 |
| S5_utility | `(subject:bill OR subject:statement OR from:(electricity OR gas OR water OR internet OR phone)) after:YYYY/MM/01 before:YYYY/MM/30` | Utility bills 公用事業帳單 |

Fetch each thread with `get_thread`. See Phase 4 for large-thread handling.
每個 thread 用 `get_thread` 取內容，大 thread 處理見 Phase 4。

---

## PHASE 4 — Extract Transactions / 提取交易

### 4.1 Multi-message Threads / 多訊息 Thread

Some banks fold same-day charges into a single Gmail thread with multiple messages (one per charge). You must:
- Iterate **all** `thread['messages']`, not just `[0]`
- If a thread exceeds ~80 KB, `get_thread` will hit the token limit. Switch to the `Grep` tool on the macOS temp file (path format: `/var/folders/nt/.../tool-results/mcp-...-get_thread-TIMESTAMP.txt`). See `references/merchant_rules.md` §5 for exact Grep patterns.
  
部分銀行將同日多筆交易折疊成一個 thread，需迭代所有 messages。Thread 超過 ~80 KB 時改用 Grep 工具，見 `references/merchant_rules.md` §5。

### 4.2 Merchant-First Logic / 商戶優先邏輯

When both a bank notification and a merchant confirmation email exist for the same charge:
- **Description** → use the **merchant email** (richer detail: order number, booking ref, etc.)
- **Amount** → use the **bank email** (final settled amount, authoritative)

同一筆消費有銀行通知和商戶確認信時：描述取商戶信（更豐富），金額取銀行信（最終扣帳最準確）。

### 4.3 eWallet Generic Merchant Strings / 電子錢包泛稱商戶

Some eWallets (e.g. mobile payment apps, QR-code wallets) show only a generic string in the bank notification rather than the actual merchant name. When you encounter a charge where the merchant is clearly an eWallet passthrough but the real merchant is unknown, **do not attempt extra Gmail searches** — they rarely return useful results and waste tokens. Add these directly to the Phase 5 confirmation table.

部分電子錢包在銀行通知只顯示泛稱（如「ALIPAY」、「GOOGLEPAY」）而非真實商戶。不要嘗試額外搜尋 Gmail，直接加入 Phase 5 確認表。

### 4.4 Exclusions / 排除項目

Record the following as `type: "internal_transfer"` and exclude from the expense total:
以下交易記為 `internal_transfer`，排除在支出合計之外：
- Investment / brokerage auto-pay 投資帳戶月供
- Credit card repayments 信用卡還款
- Bank-to-bank transfers 帳戶轉帳

### 4.5 Pre-authorisation Deduplication / 預授權去重

Some merchants (e.g. Uber, hotels, car rentals) send a pre-authorisation charge before the final charge. If the same merchant shows both a pending pre-auth and a final settled charge on the same day, keep only the final settled charge. Note the pre-auth amount in `notes` if they differ.

部分商戶（如 Uber、酒店）先發預授權，後發最終扣帳。同日出現兩筆時，只保留最終扣帳，在 notes 記錄預授權金額。

### 4.6 Foreign Currency Transactions / 外幣交易

- **With statement / 有月結單**: use the actual home-currency settlement amount from the statement.
- **Without statement / 無月結單**: use an estimated exchange rate from a public source (note the date of the rate used). Set `confidence: "medium"` and annotate `notes` with the foreign amount and estimated rate (e.g. `"USD 15.00 × estimated 7.82"`).

有月結單：取月結單上的本幣結帳金額。無月結單：用估算匯率，在 notes 標註外幣金額和匯率，confidence 設為 medium。

---

## PHASE 5 — Confirmation Gate / 待確認項目（先問清再建檔）

**This is the key token-saving step.** Before writing any JSON or HTML, compile all uncertain items into one table and ask the user in a single message:

| # | Date 日期 | Raw merchant 原始商戶 | Amount 金額 | Tentative category 暫定分類 | Question 問題 |
|---|-----------|----------------------|-------------|----------------------------|---------------|
| 1 | MM/DD | MOBILEPAY MERCHANT123 | 124.78 | ? | What is this charge? 這是什麼消費？ |
| 2 | MM/DD | FOREIGN VENDOR XYZ | 65.00 (USD) | Shopping (tentative) | Please confirm category. 確認分類？ |

Items that always need confirmation / 需要確認的情況:
- eWallet generic merchant strings (actual shop unknown) 電子錢包泛稱商戶
- Foreign currency transactions with uncertain category 外幣交易分類不確定
- Charges on statement but no matching email found 月結單有但找不到 email
- Travel bookings where payment method is unclear (prepaid? card? cash?) 旅遊訂單付款方式不清
- Salary / payroll deposits where amount isn't shown in the notification email 薪資金額未顯示在通知

Wait for the user's reply, update the transaction list, then proceed to Phase 6.
If all items are already confirmed (high confidence), skip straight to Phase 6.

---

## PHASE 6 — Generate Output / 生成輸出

### 6.1 JSON Schema (`YYYY-MM_transactions.json`)

```json
{
  "id": "YYYY-MM-001",
  "date": "YYYY-MM-DD",
  "description": "Human-readable merchant description",
  "merchant_raw": "Raw bank / email merchant string",
  "amount_local": 123.45,
  "currency_local": "HKD",
  "amount_original": "USD 15.82",
  "category": "Dining",
  "type": "expense",
  "source_account": "BankA_Credit",
  "source_type": "credit_card",
  "confidence": "high",
  "sources": ["bank_email", "merchant_email"],
  "notes": "Optional notes"
}
```

`amount_local` is always in the user's home currency (from `user_prefs.home_currency`).
`amount_original` is only populated for foreign-currency charges.
`amount_local` 始終為用戶本幣，`amount_original` 只在外幣消費時填寫。

Confidence levels / 信心等級:
- `high` — bank + merchant dual confirmation, or user-confirmed
- `medium` — single source only, or foreign-currency estimated rate
- `low` — should not appear after Phase 5 clears all uncertain items

Include a `_meta` block (period, pipeline_version, fx_rates_estimated if applicable, flags) and a `_summary` block (by_category totals, by_account totals, total_expense, investment_total).

### 6.2 HTML Report (`YYYY-MM_report.html`)

Use Chart.js 4.4.1 from cdnjs. Include:
- Summary cards: total spend, largest item, investment total, transaction count, salary (TBC if unknown)
- Category bar chart 分類橫條圖
- Category breakdown table with percentages 分類明細表
- Full transaction list with category badges; mark medium-confidence items with ⚠
- Foreign-currency estimated amounts in orange with `[CCY] XX.XX × estimated rate` annotation
- Excluded items table (investments, repayments, transfers, etc.)

---

## PHASE 7 — Month-end Follow-up / 月底補充（可選）

If the month is already over, optionally ask:
- Has the credit card been repaid? 信用卡已還款了嗎？
- Did savings target hit? 本月儲蓄目標達成了嗎？
- Any portfolio changes to log? 投資組合有變動嗎？（更新 `portfolio_history.json`）

---

## Output / 輸出

| File | Path |
|------|------|
| Transaction data | `moneypun/output/YYYY-MM_transactions.json` |
| HTML report | `moneypun/output/YYYY-MM_report.html` |
| User prefs (Phase 0) | `moneypun/config/user_prefs.json` |

When done, use `present_files` so the user can open the report directly.
