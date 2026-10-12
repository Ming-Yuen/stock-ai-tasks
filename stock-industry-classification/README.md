# Stock Industry Classification

這個任務用來讓 ChatGPT 覆核及補齊股票產業分類，完成後由 Spring Boot 匯入 `STOCK_INDUSTRY_PROFILE`。

## CSV 格式

欄位順序必須完全保持不變：

```csv
figi,symbol,industry,sub_industry,valuation_industry
```

欄位用途：

- `figi`：股票唯一識別碼，只讀，不得修改。
- `symbol`：股票代號，只讀，不得修改。
- `industry`：主要產業分類。
- `sub_industry`：較細的子產業分類。
- `valuation_industry`：用於估值比較的產業群組，可與 `sub_industry` 不同，應以業務模式及可比公司為準。

## AI 修改規則

ChatGPT 只可以修改以下三個欄位：

- `industry`
- `sub_industry`
- `valuation_industry`

必須遵守：

- 不得修改 `figi` 或 `symbol`。
- 不得修改 CSV header。
- 不得增加或刪除 row。
- 不得改名、拆分或合併 CSV。
- 現有分類只可作為參考，仍需重新覆核。
- 無法可靠判斷時留空，不要猜測。
- 同一公司的分類應保持一致及可重複。

## 檔案流程

```text
pending
  ↓
processing
  ↓
finished
  ↓
imported
```

- `pending/`：Spring Boot 已匯出、等待 ChatGPT 處理。
- `processing/`：ChatGPT 正在處理。
- `finished/`：ChatGPT 已完成，等待 Spring Boot 匯入。
- `imported/`：Spring Boot 已成功匯入資料庫。
- `error/`：資料格式、識別碼或其他無法安全處理的永久性錯誤。

## 每次 ChatGPT 執行規則

每次執行都必須按檔名／時間由舊到新持續處理多個 `pending/` CSV，不得只完成一個 batch 就無故停止。

單次執行不要求一定清空整個 `pending/`。如果接近本次 ChatGPT 執行時間、工具調用或其他平台限制，應在完成並提交目前 CSV 後安全結束；下一次排程再由剩餘 `pending/` 繼續，直到最終全部完成。

每個 CSV 的流程：

1. 先把檔案從 `pending/` 移到 `processing/` 並 commit + push。
2. 覆核每一 row，只修改三個分類欄位。
3. 完成後把同名檔案從 `processing/` 移到 `finished/`。
4. commit + push 後，如果本次仍有可用執行能力，立即繼續下一個 `pending` CSV。

如果遇到暫時性的 GitHub、網路或服務錯誤，檔案應保留在 `processing/`，等待下一次重試，不得錯誤移到 `finished/` 或 `error/`。

## 匯入責任

`finished/ → imported/` 只由 Spring Boot Import Job 負責。ChatGPT 不應自行把檔案移到 `imported/`。
