# Stock Industry Classification

這個任務用來讓 ChatGPT 覆核及補齊股票產業分類，完成後由 Spring Boot 匯入 `STOCK_INDUSTRY_PROFILE`。

## Source CSV

正式 queue source 只存在 `main`：

```text
stock-industry-classification/pending/*.csv
```

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
- 不得改名、拆分或合併 source CSV 的資料內容。
- 現有分類只可作為參考，仍需重新覆核。
- 無法可靠判斷時留空，不要猜測。
- 同一公司的分類應保持一致及可重複。

## Output 規格

每個成功 source 必須產生 **一個** UTF-8 CSV output：

```text
stock-industry-classification/finished/<source 原檔名>
```

要求：

- output 檔名必須與 source basename 相同。
- `encoding` 必須是 `utf-8`。
- output 必須是完整 CSV 內容，不是 diff。
- header、row 數、row 順序、每一 row 的 `figi` 與 `symbol` 必須與 source 完全一致。
- 只有三個分類欄位可不同。

本 task 不產生第二個 output，也不使用 base64。

## Queue 流程

```text
Spring Boot
  ↓
main: pending/*.csv
  ↓
ChatGPT 只讀 main 並完成分析
  ↓
ai-results: ai-output/execution-*-cpNNN.json (schema v2 checkpoint bundle)
  ↓
GitHub Actions finalizer
  ↓
main: finished/*.csv
  ↓
Spring Boot import
  ↓
main: imported/*.csv
```

- `pending/`：Spring Boot 已匯出，等待 ChatGPT 分析。
- `ai-results/ai-output`：只用來傳遞 ChatGPT checkpoint result bundle；不是 finished / imported 狀態來源。
- `finished/`：GitHub Actions finalizer 依 bundle output 在 `main` 建立，等待 Spring Boot 匯入。
- `imported/`：只由 Spring Boot 成功匯入後建立。
- `error/`：永久性資料錯誤的診斷位置。

ChatGPT 不得修改 `main`，不得自行建立 `finished` 或執行 `finished → imported`。

## Schema v2 result entry

每個成功 source 在 checkpoint bundle 中使用：

```json
{
  "task": "stock-industry-classification",
  "source": {
    "path": "stock-industry-classification/pending/example.csv",
    "blob_sha": "actual-main-git-blob-sha-used-for-analysis"
  },
  "outputs": [
    {
      "path": "stock-industry-classification/finished/example.csv",
      "encoding": "utf-8",
      "content": "complete processed CSV content"
    }
  ]
}
```

實際執行時不得 hardcode `example.csv`；必須使用真實 source path、`main` source blob SHA 與原檔名。

## 每次 ChatGPT execution

1. 依 root README 從 `main` 動態發現工作並建立 execution cache。
2. 按檔名／時間由舊到新處理 `pending/`。
3. 對每個 source CSV 覆核每一 row，只修改三個分類欄位。
4. 驗證 header、row 數、row 順序、`figi`、`symbol` 均保持不變。
5. 將完整處理後 CSV 放入該 source 的唯一 `outputs[]` entry。
6. 正常每完成 **3 個**成功 source，就依 root README 建立一個 checkpoint bundle；最後不足 3 個亦要提交，不等待下一次 execution。
7. 若 transport safety ceiling 比 3-source checkpoint 更早觸發，先依 root README 提交目前完整結果，不得切割單一 source result。
8. checkpoint 前只針對 included source 驗證其在 `main` 仍存在且 blob SHA 未改變；不要為每個 checkpoint重新 full scan repository。
9. ChatGPT 唯一允許的 GitHub write 類型是把 schema v2 checkpoint bundle用一般 `create_file` 寫到 `ai-results/ai-output/`；同一次 execution 可按 checkpoint 執行多次，但不得使用其他 intermediate Git write。
10. checkpoint 成功後，在 execution cache 將 included source 標記為 submitted，繼續處理下一組；不得等待所有 pending 都處理完才一次提交。
11. GitHub Actions finalizer 再從 `ai-results` 讀取一個或多個 bundle，套用 output 到 `main` 並刪除 source。
12. cache drain 後才重新讀取 `main` 最新 HEAD + recursive tree 做最終確認；若有真正的新 pending，再繼續處理。

如果 source 在 checkpoint 建立前已被改動或移走，該 result 必須從該 checkpoint 排除，不得覆蓋最新來源。

## 無法判斷

個別股票無法可靠判斷分類時，三個可修改欄位可保持空白；不要為了填滿而猜測。

## 永久性資料錯誤

若 CSV schema 錯誤、缺少 `figi`／`symbol`、row 結構損壞，或存在其他使整個 source 無法安全處理的永久性問題：

- 不產生該 source 的成功 result entry。
- 在最終報告指出實際問題。
- 是否需要額外 error 診斷檔依 root README framework 規則處理。

## 匯入責任

`finished/ → imported/` 只由 Spring Boot Import Job 負責。ChatGPT 與 GitHub Actions finalizer 都不得自行完成此步驟。
