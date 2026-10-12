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

ChatGPT 只可以修改：

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

## Queue 流程

新的 framework 流程：

```text
Spring Boot
  ↓
pending/
  ↓
ChatGPT 讀取及分析
  ↓
root ai-output/execution-*.json
  ↓
GitHub Actions finalizer
  ↓
finished/
  ↓
Spring Boot import
  ↓
imported/
```

- `pending/`：Spring Boot 已匯出，等待 ChatGPT 分析。
- `processing/`：只保留舊架構遺留／恢復用途；新的 ChatGPT execution 不再把 pending 移到 processing。
- `finished/`：GitHub Actions finalizer 建立的完成結果，等待 Spring Boot 匯入。
- `imported/`：Spring Boot 已成功匯入資料庫。
- `error/`：資料格式、識別碼或其他永久性資料錯誤的診斷位置。

ChatGPT 不得自行做 `pending → processing`、`processing → finished` 或 `finished → imported`。

## 每次 ChatGPT execution

1. 依 root README 的 framework 規則找出工作。
2. 優先處理 legacy `processing/`，再按檔名／時間由舊到新處理 `pending/`。
3. 對每個來源 CSV 覆核每一 row，只修改三個分類欄位。
4. 驗證：header 不變、row 數不變、每一 row 的 `figi` 與 `symbol` 不變。
5. 完成的完整 CSV 內容加入本次 execution 的 result bundle。
6. bundle entry 的 `source_path` 必須指向實際讀取的 pending 或 legacy processing 檔案，`source_blob_sha` 必須是實際用於分析的來源 blob SHA。
7. ChatGPT 不直接建立 finished；由 GitHub Actions finalizer 驗證來源 SHA 後套用 bundle。

如果來源在 bundle 建立前已被改動或移走，將該結果從本次 bundle 排除，不得覆蓋最新來源。

## 無法判斷

個別股票無法可靠判斷分類時，三個可修改欄位可保持空白；不要為了填滿而猜測。

## 永久性資料錯誤

若 CSV schema 錯誤、缺少 `figi`／`symbol`、row 結構損壞，或存在其他使整個 batch 無法安全處理的永久性問題：

- 不產生該 batch 的成功 result entry。
- 在最終報告指出實際問題。
- 是否需要額外 error 診斷檔依 root README framework 規則處理。

## 匯入責任

`finished/ → imported/` 只由 Spring Boot Import Job 負責。ChatGPT 與 GitHub Actions finalizer都不應自行完成此步驟。
