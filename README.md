# MoneyPun 💰

**A Claude skill for automated personal finance monthly reporting.**
**自動化個人財務月結的 Claude Skill。**

MoneyPun fetches your bank and credit card Gmail notifications, matches them against merchant confirmation emails, resolves ambiguous charges interactively, and produces a clean JSON transaction log and HTML report — all without manual data entry.

MoneyPun 從 Gmail 抓取銀行和信用卡通知，對比商戶確認信，互動式解決模糊項目，最終生成乾淨的 JSON 交易記錄和 HTML 報告，無需手動錄入。

---

## What it does / 功能概覽

1. **First-time setup** — asks which accounts have Gmail notifications enabled, which eWallets you use, and confirms your spending categories. Saves to `config/user_prefs.json` so you never answer this again.
   **首次設定** — 問你哪些帳戶有 Gmail 通知、用哪些電子錢包、確認分類。儲存後下次不再問。

2. **Pre-run checklist** — before fetching anything, asks for your investment account balance and whether you have a monthly statement to upload (useful for cards without email notifications).
   **執行前問卷** — 抓取前先問本月投資餘額和是否有月結單，省去後期猜算。

3. **5-route Gmail search** — bank debits, merchant confirmations, receipts, eWallet notifications, utility bills.
   **5 路 Gmail 搜尋** — 銀行扣帳、商戶確認信、收據、電子錢包、公用事業帳單。

4. **Interactive confirmation gate** — all low-confidence items (eWallet generic merchants, foreign currency charges, salary amounts) are shown to you in one table *before* any output is generated. No token waste on re-runs.
   **互動確認關卡** — 所有低信心項目（電子錢包泛稱商戶、外幣交易、薪資金額）在生成輸出前一次過確認，避免反覆重跑浪費 token。

5. **Output** — `YYYY-MM_transactions.json` and `YYYY-MM_report.html` with a category breakdown chart, full transaction list, and excluded items table.
   **輸出** — 交易 JSON 和 HTML 報告，含分類圖表、完整交易列表、排除項目表。

---

## Requirements / 系統需求

- [Claude Desktop](https://claude.ai/download) with **Cowork mode** enabled
- Gmail MCP connected (Settings → Connectors → Gmail)
- Your banks / cards must send spending notification emails to Gmail
  銀行／信用卡需在 Gmail 開啟消費通知 email

---

## Installation / 安裝

1. Download [`moneypun.skill`](./moneypun.skill)
2. Double-click the file — Claude Desktop will prompt you to install it
3. The skill appears in your Claude session automatically
   技能安裝後在 Claude 對話中自動啟用

---

## First run / 第一次使用

Just say:
> **"Run MoneyPun for [month]"** or **"跑 MoneyPun，[月份]"**

Claude will walk you through the one-time setup, then proceed with the pipeline.
Claude 會引導你完成一次性設定，然後開始執行 pipeline。

---

## Configuration files / 設定檔

These live in `moneypun/config/` in your project folder. They are **not included in this repo** (they contain personal account details).

| File | Purpose |
|------|---------|
| `user_prefs.json` | Created on first run — accounts, eWallets, categories |
| `accounts.json` | Bank account IDs and notification sender addresses |
| `categories.json` | Spending category list |
| `standing_orders.json` | Fixed monthly expenses (rent, etc.) auto-injected each run |

---

## Repo structure / 目錄結構

```
moneypun/
├── README.md
├── moneypun.skill              ← Install this in Claude Desktop
└── moneypun-skill/             ← Source files (human-readable)
    ├── SKILL.md                ← Full pipeline workflow
    └── references/
        └── merchant_rules.md   ← Merchant classification rules
```

---

## What NOT to commit / 不要上傳到 GitHub

If you fork this repo and customise it for your own setup, make sure to exclude:

```gitignore
# Personal data — DO NOT COMMIT
moneypun/config/user_prefs.json
moneypun/config/accounts.json
moneypun/output/
```

A `.gitignore` template is included below.

---

## .gitignore template

```gitignore
# MoneyPun — personal data
moneypun/config/user_prefs.json
moneypun/config/accounts.json
moneypun/output/

# Python cache
__pycache__/
*.pyc

# macOS
.DS_Store
```

---

## Customising for your setup / 個人化設定

MoneyPun is designed to work with any bank that sends Gmail spending notifications. After running Phase 0 setup, edit `config/accounts.json` to add your bank's exact notification sender email addresses and card identifiers.

MoneyPun 適用於任何在 Gmail 發送消費通知的銀行。完成 Phase 0 設定後，編輯 `config/accounts.json` 填入你的銀行通知寄件 email 地址和卡號識別碼。

---

## License

MIT — feel free to fork and adapt for your own banking setup.
