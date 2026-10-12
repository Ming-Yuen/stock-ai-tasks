# stock-ai-tasks

`stock-ai-tasks` 是 Spring Boot 與 ChatGPT 之間的 Git 任務佇列。

Spring Boot 負責產生待處理資料並推送到 GitHub；ChatGPT 從 `pending/` 取得任務、完成 AI 處理後放到 `finished/`；Spring Boot 再匯入資料庫並把成功檔案移到 `imported/`。

## 工作流程

```text
Spring Boot export
        ↓
pending/
        ↓
ChatGPT
        ↓
processing/
        ↓
AI 處理完成
        ↓
finished/
        ↓
Spring Boot import
        ↓
imported/
```

處理失敗且屬於資料格式、識別碼或其他不可安全繼續的問題時，檔案移到 `error/`。暫時性的 GitHub、網路或服務錯誤不應移到 `error/`，應保留目前狀態等待重試。

## ChatGPT 執行規則

每次 ChatGPT 任務執行時，必須由舊到新持續處理 `pending/` 內所有可處理的 CSV，直到 `pending/` 為空；不要只處理一個 batch 就停止。

每個檔案都遵循以下生命週期：

```text
pending → processing → finished
```

開始處理前先移到 `processing/` 並提交；完成後移到 `finished/` 並提交及 push。若遇到暫時性錯誤，保留在 `processing/`，避免誤標記為完成。

## 任務目錄

目前任務：

```text
stock-industry-classification/
  pending/
  processing/
  finished/
  imported/
  error/
```

各任務的欄位、AI 可修改範圍及資料規則，以任務目錄內的 `README.md` 為準。

## Stock Industry Classification

CSV schema 必須保持：

```csv
figi,symbol,industry,sub_industry,valuation_industry
```

ChatGPT 只可修改：

- `industry`
- `sub_industry`
- `valuation_industry`

`figi` 與 `symbol` 是識別欄位，不得修改。檔名、header、row 數量不得改變，不得拆檔或合併檔案。不確定的分類留空，不要猜測。

## Git 原則

- Spring Boot export 後 commit + push 到 `main`。
- ChatGPT 處理檔案時 commit + push 到 `main`。
- Spring Boot import 成功後把 `finished/` 檔案移到 `imported/`，再 commit + push。
- 不要修改與目前 AI task 無關的檔案。
