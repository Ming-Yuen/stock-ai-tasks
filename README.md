# stock-ai-tasks

`stock-ai-tasks` 是 Spring Boot 與 ChatGPT 之間的 Git 任務佇列。

這個 repository 只定義 **AI task framework 的共通規則**。每一個頂層 task folder 都代表一個獨立任務；任務自己的資料格式、欄位定義、AI 可修改範圍、判斷規則及其他限制，必須寫在該 task folder 內的 `README.md`，不要寫在根目錄 README。

ChatGPT 透過已授權的 GitHub connector 直接讀寫這個 private repository；正常排程不依賴 Desktop Commander 或本機 VM 才能存取 GitHub。

## Repository 結構

每個頂層資料夾代表一個 AI 任務，例如：

```text
stock-ai-tasks/
  README.md
  task-a/
    README.md
    pending/
    processing/
    finished/
    imported/
    error/
  task-b/
    README.md
    pending/
    processing/
    finished/
    imported/
    error/
```

新增任務時，應新增一個新的頂層 task folder，而不是把不同任務混在同一個 folder。

## ChatGPT 開始工作前

每次執行時，ChatGPT 必須：

1. 查看 repository 根目錄，找出所有 task folders。
2. 進入每一個 task folder 前，先閱讀該 folder 的 `README.md`。
3. 以該 task README 為唯一的任務專屬規格來源，包括 CSV schema、欄位意義、哪些欄位可以修改、判斷標準及禁止事項。
4. 不要把某一個 task 的規則套用到其他 task。
5. 如果 task folder 缺少 README，或 README 無法清楚判斷如何安全處理資料，停止處理該 task，不要自行猜測規則。

## 共通工作流程

一般 task 使用以下生命週期：

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

各 folder 的共通責任：

- `pending/`：Spring Boot 已匯出，等待 ChatGPT 處理。
- `processing/`：ChatGPT 已領取並正在處理。
- `finished/`：ChatGPT 已完成，等待 Spring Boot 匯入。
- `imported/`：Spring Boot 已成功匯入；ChatGPT 不應自行把檔案移入此 folder。
- `error/`：只用於資料格式、識別碼、schema 或其他無法安全繼續的永久性問題。

若某個 task README 明確定義不同流程，以該 task README 為準。

## ChatGPT 共通執行規則

對每一個可處理的 task：

1. 先閱讀 task README。
2. 按檔名／時間由舊到新處理 `pending/`。
3. 每次執行必須持續處理該 task 的所有可處理 pending files，直到 `pending/` 為空；不要只處理一個 batch 就停止。
4. 開始處理一個檔案前，先把它從 `pending/` 移到 `processing/` 並 commit + push。
5. 嚴格按照 task README 修改內容。
6. 完成後，把同名檔案從 `processing/` 移到 `finished/`，再 commit + push。
7. 完成一個檔案後繼續下一個，直到沒有可處理的 pending files。

如果遇到暫時性的 GitHub、網路或服務錯誤：

- 不要把未完成檔案標記為 `finished`。
- 保留檔案在 `processing/`，讓下一次執行可以重試。
- 停止本次執行，避免後續 commit 與 repository 狀態衝突。

如果遇到資料本身的永久性問題：

- 按 task README 的規則處理。
- 若沒有更具體規則，可把該檔案移到 `error/` 並記錄原因，然後繼續其他可處理的 pending files。

## Git 原則

- 所有 AI task 變更直接針對 `main`，除非某個 task README 另有規定。
- 每次狀態轉移或完成處理後都要 commit + push，讓 repository 狀態可追蹤及可恢復。
- 不要修改目前 task 以外的檔案。
- 不要修改其他 task 的 README 或資料，除非該次任務明確要求。
- 不要自行改變 task folder 結構、檔名規則或資料 schema。

## 任務專屬規格

根目錄 README 不應包含任何特定 task 的欄位或業務規則。

要了解某個任務，必須進入該 task folder 並閱讀：

```text
<task-name>/README.md
```

該 README 應至少說明：

- 任務目的
- 輸入／輸出格式
- 欄位定義
- AI 可以修改及禁止修改的內容
- 判斷／分類／生成規則
- 無法判斷時的處理方式
- task-specific 的驗證與錯誤處理規則
