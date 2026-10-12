# stock-ai-tasks

`stock-ai-tasks` 是 Spring Boot、ChatGPT Scheduled Task 與 GitHub Actions 共用的 AI 任務佇列。

## 核心責任

- Spring Boot：建立 task 的 `pending/` 輸入，匯入 `finished/` 結果，成功後移到 `imported/`。
- ChatGPT：讀取來源、依各 task README 完成 AI 工作與驗證，最後只提交 result bundle。
- GitHub Actions：唯一負責驗證 bundle、套用 outputs、刪除來源、commit + push。

ChatGPT 不直接做 queue state move，不自行建立 `finished/` 或修改 `imported/`，也不使用 `create_tree`、`create_commit`、`update_ref` 或 force push 推進 queue。

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

root README 只定義 framework。每個 task 的輸入格式、輸出格式、AI 業務規則、可修改範圍、輸出檔名／路徑規則與驗證方式，只能由該 task README 定義。

## 每次 execution 開始

1. 重新讀取 `main` 最新 root README，作為 framework 唯一正式規則來源。
2. 讀取最新 HEAD 與 recursive Git tree，動態找出所有具 README 的頂層 task folders。
3. 快取各 task 的 `pending/`、legacy `processing/`、`finished/`、`imported/`、`error/` 與 README blob SHA。
4. 第一次處理某 task 前讀取該 task 最新 README；同一次 execution 內 README blob SHA 未變可重用快取。

不得 hardcode task 名稱、batch 路徑或 task-specific 規則，不得依賴 `manifest.json` 或 code search 判斷 queue 狀態。

## 工作選擇與 execution drain

先處理 legacy `processing/`，再按檔名／時間由舊到新處理 `pending/`。

新的流程不再把 `pending` 移到 `processing`。`processing/` 只保留舊架構遺留與恢復用途；ChatGPT 可直接讀取 legacy source，GitHub Actions finalize 後刪除來源。

一次 execution 的目標是 drain 所有 task 中所有可安全處理的 legacy `processing/` 與 `pending/` 工作：

- 不設定固定 batch 上限。
- 完成一個 source 後繼續下一個。
- 某 task 沒工作時繼續其他 task。
- 正常情況使用 execution cache，不為每個 source 無條件 full scan。
- 當 cache 判斷已無工作時，再讀取最新 HEAD + recursive tree 最終確認。
- 若最終確認發現新工作，繼續處理並再次確認。
- 只有所有 task 已 drain，或遇到真實且無法安全繼續的阻礙，才結束分析階段。

## Result bundle schema v2

如果本次至少完成一個 source，只在 queue 已 drain 並完成最終 source SHA 驗證後建立一份：

```text
ai-output/execution-YYYYMMDD-HHmmss.json
```

通用 schema：

```json
{
  "schema_version": 2,
  "execution_id": "YYYYMMDD-HHmmss",
  "created_at": "ISO-8601 timestamp",
  "results": [
    {
      "task": "task-name",
      "source": {
        "path": "task-name/pending/input-file.ext",
        "blob_sha": "git-blob-sha-used-for-analysis"
      },
      "outputs": [
        {
          "path": "task-name/finished/output-file.ext",
          "encoding": "utf-8",
          "content": "complete output file content"
        }
      ]
    }
  ]
}
```

### source 規則

- `task` 必須是實際存在且具有 README 的頂層 task folder。
- `source.path` 只可位於該 task 的 `pending/` 或 legacy `processing/` 下。
- `source.blob_sha` 必須是實際讀取並用於 AI 分析的 Git blob SHA。
- 建立 bundle 前必須重新讀取最新 tree，確認 source 仍存在且 blob SHA 未改變。
- source 已被改動、移走或處理時，該 result 必須從 bundle 排除。

### outputs 規則

- 每個 result 必須至少有一個 output，可有多個 outputs。
- 每個 `outputs[].path` 必須位於同一 task 的 `finished/` 下；可使用不同檔名、副檔名或子路徑。
- output 路徑、數量、格式與內容要求由 task README 定義；framework 不假設 CSV、同名輸出或單一輸出。
- `encoding="utf-8"`：`content` 是完整 UTF-8 文字內容。
- `encoding="base64"`：`content` 是完整檔案 bytes 的標準 Base64；只在 task 規則與實際工具能力可安全產生及驗證二進位結果時使用。
- 不得用 output 修改 root README、workflow、其他 task 或 source queue。
- 同一 bundle 中不得讓不同 source 寫入同一 output path。
- output 是完整檔案內容，不是 diff。

多個已完成 source 集中在同一 bundle，目的仍是把 GitHub Connector 寫入降到每次 execution 通常一次。

## GitHub Actions finalizer

`.github/workflows/finalize-ai-output.yml` 是 result bundle 的唯一套用者。

當 `ai-output/*.json` 出現後，finalizer：

1. 串行處理，避免多個 finalizer 同時修改 queue。
2. 驗證 schema v2、task、source path、source blob SHA、output path 與 encoding。
3. 在修改 repository 前先驗證所有 bundle；任何 bundle 驗證失敗則整次 workflow 失敗，不做部分套用。
4. 對每個 result 寫入其全部 `outputs[]`，然後刪除該 source。
5. 已存在的 output 只有內容完全相同時才可視為可重用；內容不同則停止，禁止覆蓋。
6. 所有成功套用的 bundle 一併刪除。
7. 將所有變更放進 Git commit 並 push `main`；一般並發更新時可同步最新 main 後安全重試，不得 force push。

Finalizer 不呼叫 OpenAI API，因此不產生 OpenAI API 用量。

## task README 的責任

每個 task README 至少要清楚定義：

- 可接受的 source 檔案格式與 schema。
- AI 要做的工作與禁止修改的內容。
- 完成條件與驗證規則。
- 要產生幾個 output。
- 每個 output 的 `finished/` 路徑／命名規則、格式與 encoding。
- 無法可靠判斷與永久性資料錯誤的處理方式。

GitHub Actions finalizer不理解任何 task 業務內容；它只執行通用的 source SHA 驗證、output path 安全檢查、檔案套用與 Git 操作。

## finished / imported

- `finished/` 由 GitHub Actions finalizer依 bundle outputs 建立，等待 Spring Boot 匯入。
- `imported/` 只由 Spring Boot Import Job 負責。
- ChatGPT 與 finalizer 都不得自行完成 `finished → imported`。

## Error 與停止

永久性資料問題依 task README 處理；不得把未完成 source 假裝成成功 result。

若 ChatGPT 因 Connector、網路、平台限制或規則不足而無法安全提交 bundle：

- 不修改 source queue state。
- 最終報告說明可核實原因與尚未提交的 source。
- 若 repository 寫入本身就是故障來源，不要求再做第二次無意義寫入建立 execution report。

不得因已處理一個 source、已處理一個 task、達到人工數量或認為「已做足夠多」而提前停止。

## 正常結束

```text
讀取最新 framework / tree
↓
持續處理所有 task 的 legacy processing
↓
持續處理所有 task 的 pending
↓
cache 判定無工作
↓
重新讀取最新 HEAD + recursive tree
↓
若有新工作：繼續處理並再次確認
↓
若所有 task 已 drain：驗證所有 source SHA
↓
建立本次唯一 schema v2 result bundle
↓
GitHub Actions finalizer 套用 source + outputs[]
↓
本次 ChatGPT execution 結束
```

最終聊天報告至少說明：

- 本次完成 AI 分析的 source 數。
- bundle 路徑。
- output 總數。
- 因 source 變更而排除的 result 數。
- 最終確認時仍可見的 pending / legacy processing 概況。

如果 execution 開始及最終確認都沒有任何可處理 source，則不建立空 bundle。
