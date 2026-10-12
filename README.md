# stock-ai-tasks

`stock-ai-tasks` 是 Spring Boot、ChatGPT Scheduled Task 與 GitHub Actions 共用的 AI 任務佇列。

## 核心責任

- Spring Boot：建立 `pending/`，匯入 `finished/`，成功後移到 `imported/`。
- ChatGPT：讀取 batch、依各 task README 完成 AI 分析，最後只提交 result bundle。
- GitHub Actions：唯一負責把 result bundle 套用到 queue，刪除來源並建立 `finished/`，再 commit + push。

ChatGPT 不再直接做 batch state move，也不應自行建立 `finished/` 或修改 `imported/`。

## Repository 結構

```text
README.md
.github/workflows/finalize-ai-output.yml
ai-output/
  execution-YYYYMMDD-HHmmss.json
<task-name>/
  README.md
  pending/
  processing/   # legacy/recovery only
  finished/
  imported/
  error/
```

根目錄 README 只定義 framework。每個 task 的 schema、可修改欄位、業務規則、驗證及 task-specific error 規則只放在該 task 的 README。

## 每次 execution 開始

1. 讀取 `main` 最新 root README，作為 framework 唯一正式規則來源。
2. 讀取最新 HEAD 與 recursive Git tree，動態找出所有具 README 的頂層 task folders。
3. 快取各 task 的 `pending/`、legacy `processing/`、`finished/`、`imported/`、`error/` 與 README blob SHA。
4. 第一次處理某 task 前讀取該 task 最新 README；同一次 execution 內 SHA 未變可重用快取。

不得 hardcode task 名稱或 batch 路徑，不得依賴 `manifest.json` 或 code search 判斷 queue 狀態。

## Batch 選擇

先處理 legacy `processing/`，再按檔名／時間由舊到新處理 `pending/`。

新的流程不再把 `pending` 移到 `processing`。`processing/` 只保留舊架構遺留與恢復用途；ChatGPT 直接讀取它並完成分析，GitHub Actions finalize 後再刪除來源。

ChatGPT 可在一次 execution 內持續分析多個 batch，不設固定 batch 上限。正常情況不要每完成一個 batch 就重新 full scan。

## Result bundle

如果本次至少完成一個 batch，建立 bundle 前必須重新讀取最新 HEAD + recursive tree，確認每個來源仍存在且 blob SHA 未改變。已改變或已被其他 writer 處理的來源必須從 bundle 排除。

每次 execution 正常只建立一份：

```text
ai-output/execution-YYYYMMDD-HHmmss.json
```

Schema：

```json
{
  "schema_version": 1,
  "execution_id": "YYYYMMDD-HHmmss",
  "created_at": "ISO-8601 timestamp",
  "results": [
    {
      "task": "task-name",
      "filename": "batch-file.csv",
      "source_path": "task-name/pending/batch-file.csv",
      "source_blob_sha": "blob-sha-used-for-analysis",
      "content": "complete processed file content"
    }
  ]
}
```

Bundle 規則：

- `task` 必須是實際 task folder。
- `filename` 必須等於來源 basename。
- `source_path` 只可為 `<task>/pending/<filename>` 或 `<task>/processing/<filename>`。
- `source_blob_sha` 必須是實際讀取並用於分析的來源 blob SHA。
- `content` 必須是依 task README 完成並驗證後的完整檔案內容，不是 diff。
- 同一 bundle 不得重複同一 task + filename。
- ChatGPT 只建立 bundle；不得自行刪除來源或建立 finished。
- bundle 使用一般檔案建立操作寫入；若寫入失敗，來源保持原狀，下一次 execution 可安全重做。

把多個 batch 集中到一份 bundle 的目的，是把 Connector 寫入次數降到每次 execution 通常一次。

## GitHub Actions finalizer

`.github/workflows/finalize-ai-output.yml` 是 queue state transition 的唯一執行者。

當 `ai-output/*.json` 出現後，finalizer 必須：

1. 串行處理，避免多個 finalizer 同時修改 queue。
2. 驗證 bundle schema、task/path、來源存在與目前 blob SHA 等於 `source_blob_sha`。
3. 驗證通過後，在同一個 Git commit 中刪除來源 `pending/` 或 legacy `processing/`，建立同名 `finished/` 結果，並刪除已套用 bundle。
4. 若同名 finished 已存在，只可在內容完全相同時視為已完成；內容不同時停止，不得覆蓋。
5. push 遇到一般並發更新時，先同步最新 main 再安全重試；不得覆蓋其他 writer 的變更。
6. 驗證失敗時不得部分猜測套用，workflow 應失敗並保留 bundle 供診斷。

Finalizer 不呼叫 OpenAI API，因此不產生 OpenAI API 用量。

## finished / imported

- `finished/` 由 GitHub Actions finalizer 建立，等待 Spring Boot 匯入。
- `imported/` 只由 Spring Boot Import Job 負責。
- ChatGPT 不得把 finished 移到 imported。

## Error 與停止

永久性資料問題依 task README 處理；不得把未完成 batch 假裝成 finished。

若 ChatGPT 因 Connector、網路、平台限制或規則不足而無法安全提交 bundle：

- 不修改 batch state。
- 最終報告說明可核實原因與尚未提交的 batch。
- 若 repository 寫入本身就是故障來源，不要求再做第二次無意義寫入來建立 execution report。

## 正常結束

成功建立 bundle 後，本次 ChatGPT execution 可結束；queue state transition 由 GitHub Actions 接手。

最終聊天報告至少說明：

- 本次完成 AI 分析的 batch 數。
- bundle 路徑。
- 因來源變更而排除的 batch 數。
- 當時仍可見的 pending / legacy processing 概況。

如果 execution 開始時沒有可處理 pending 或 legacy processing，則不通知，也不建立空 bundle。
