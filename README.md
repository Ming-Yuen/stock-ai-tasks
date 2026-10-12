# stock-ai-tasks

`stock-ai-tasks` 是 Spring Boot、ChatGPT Scheduled Task 與 GitHub Actions 共用的 AI 任務佇列。

## 核心責任

- Spring Boot：在 `main` 建立各 task 的 `pending/` 輸入，匯入 `finished/` 結果，成功後移到 `imported/`。
- ChatGPT：只讀 `main` 的 framework、task README 與 `pending/` source，完成 AI 工作後只把 result bundle 寫到專用 `ai-results` branch。
- GitHub Actions：唯一負責讀取 `ai-results` bundle、驗證 `main` source SHA、套用 outputs、刪除 source、commit + push `main`，成功後再清理已套用 bundle。

ChatGPT 不直接修改 `main`，不做 queue state move，不自行建立 `finished/` 或修改 `imported/`。

## Branch 與 Repository 結構

`main` 是正式 queue 與 framework：

```text
README.md
.github/workflows/finalize-ai-output.yml
<task-name>/
  README.md
  pending/
  finished/
  imported/
  error/
```

`ai-results` 是 AI result inbox，只作傳輸用途：

```text
ai-output/
  execution-YYYYMMDD-HHmmss.json
```

queue state 一律以 `main` 為準；不得用 `ai-results` 判斷 pending / finished 狀態。

root README 只定義 framework。每個 task 的輸入格式、輸出格式、AI 業務規則、可修改範圍、輸出檔名／路徑規則與驗證方式，只能由該 task README 定義。

## ChatGPT GitHub 存取邊界

分析階段對 GitHub 必須完全 read-only。

允許的讀取操作只有用來取得最新資料的 read/fetch 類操作，例如：

- 讀取 `main` root README。
- 讀取 `main` HEAD / recursive Git tree。
- 讀取 task README。
- 讀取 source file / blob。

禁止任何 intermediate Git write，包括但不限於：

- `create_blob`
- `create_tree`
- `create_commit`
- `update_ref`
- `update_file`
- `delete_file`
- force push
- 建立任何 temporary Git object 作暫存

每次 execution 唯一允許的 GitHub 寫入是最後一次：

```text
create_file(
  branch = "ai-results",
  path = "ai-output/execution-YYYYMMDD-HHmmss.json"
)
```

除了這一次 result bundle create，不得對 GitHub 做其他 write。

## 每次 execution 開始

1. 重新讀取 `main` 最新 root README，作為 framework 唯一正式規則來源。
2. 讀取 `main` 最新 HEAD 與 recursive Git tree，動態找出所有具 README 的頂層 task folders。
3. 快取各 task 的 `pending/`、`finished/`、`imported/`、`error/` 與 README blob SHA。
4. 第一次處理某 task 前讀取該 task 在 `main` 的最新 README；同一次 execution 內 README blob SHA 未變可重用快取。

不得 hardcode task 名稱、batch 路徑或 task-specific 規則，不得依賴 `manifest.json` 或 code search 判斷 queue 狀態。

## 工作選擇與持續 drain

按檔名／時間由舊到新處理所有 task 的 `main/<task>/pending/` source。

整個 framework 的目標是持續處理，直到 `main` 的所有可安全處理 `pending/` 最終為空。

單次 execution 規則：

- 不設定固定 source / batch 數量上限。
- 完成一個 source 後繼續下一個。
- 某 task 沒工作時繼續其他 task。
- 正常情況使用 execution cache，不為每個 source 無條件 full scan。
- 若本次所有可安全處理結果可放進安全 bundle，繼續處理直到 cache 判定 pending 為空，再重新讀最新 `main` HEAD + recursive tree 最終確認。
- 若最終確認又發現新 pending，繼續處理並再次確認。
- 若本次待提交 bundle 接近 transport safety ceiling，允許在 source 邊界停止增加本次結果、完成 SHA 驗證並提交 bundle；剩餘 pending 留給下一次 execution 繼續。

### Bundle transport safety ceiling

為降低 Connector 大 payload / safety failure 風險：

- 正常目標：序列化後 bundle 不超過約 **256 KiB UTF-8**。
- 加入下一個完整 result 會令 bundle 超過此目標時，不加入該 result；先提交目前已完成結果。
- 不得切割單一 source 的 result；一個 result 必須完整。
- 若單一合法 result 本身超過 256 KiB，仍可作為單一 bundle，但最終報告必須指出。

這是 transport safety boundary，不是固定 batch 上限。下一次 scheduled execution 會從仍在 `main/pending/` 的 source 繼續，因此最終仍會 drain 到 pending 為空。

## Result bundle schema v2

如果本次至少完成一個 source，在提交前重新驗證所有 included source 的 `main` blob SHA，然後只建立一份：

```text
ai-results branch:
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

- `task` 必須是 `main` 實際存在且具有 README 的頂層 task folder。
- `source.path` 只可位於該 task 的 `pending/` 下。
- `source.blob_sha` 必須是實際讀取並用於 AI 分析的 `main` Git blob SHA。
- 建立 bundle 前必須重新讀取最新 `main` tree，確認 included source 仍存在且 blob SHA 未改變。
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

## GitHub Actions finalizer

`.github/workflows/finalize-ai-output.yml` 在 `ai-results` branch 出現 `ai-output/*.json` 時觸發。

Finalizer：

1. 同時 checkout `main` queue 與 `ai-results` inbox。
2. 串行處理，避免多個 finalizer 同時修改 queue。
3. 驗證 schema v2、task、`pending/` source path、`main` source blob SHA、output path 與 encoding。
4. 在修改 `main` 前先驗證所有待處理 bundle；任何 bundle 驗證失敗則整次 workflow 失敗，不做部分套用。
5. 對每個 result 寫入其全部 outputs，然後刪除 `main` 的該 pending source。
6. 已存在的 output 只有內容完全相同時才可重用；內容不同則停止，禁止覆蓋。
7. commit + push `main` 成功後，才從 `ai-results` 刪除本次已套用 bundle。
8. push 遇到一般並發更新時可 rebase 最新 branch 後安全重試；不得 force push。

Finalizer 不呼叫 OpenAI API，因此不產生 OpenAI API 用量。

## task README 的責任

每個 task README 至少要清楚定義：

- 可接受的 source 檔案格式與 schema。
- AI 要做的工作與禁止修改的內容。
- 完成條件與驗證規則。
- 要產生幾個 output。
- 每個 output 的 `finished/` 路徑／命名規則、格式與 encoding。
- 無法可靠判斷與永久性資料錯誤的處理方式。

GitHub Actions finalizer 不理解任何 task 業務內容；它只執行通用 source SHA 驗證、output path 安全檢查、檔案套用與 Git 操作。

## finished / imported

- `finished/` 由 GitHub Actions finalizer 在 `main` 建立，等待 Spring Boot 匯入。
- `imported/` 只由 Spring Boot Import Job 負責。
- ChatGPT 與 finalizer 都不得自行完成 `finished → imported`。

## Error 與停止

永久性資料問題依 task README 處理；不得把未完成 source 假裝成成功 result。

若 ChatGPT 因 Connector、網路、平台限制或規則不足而無法安全提交 bundle：

- 不修改 `main` source queue state。
- 不改用任何其他 Git write 方法補救；尤其不得改用 blob/tree/commit/ref 操作。
- 最終報告說明可核實原因與尚未提交的 source。
- **不得因單次 execution 失敗而 disable、pause、delete 或修改本 scheduled task 的啟用狀態／schedule。**
- 下一次 scheduled execution 會重新從 `main` 最新狀態開始。

## 正常結束

### pending 已可在本次 drain 完

```text
讀取 main 最新 framework / tree
↓
持續處理所有 task pending
↓
cache 判定無工作
↓
重新讀取 main 最新 HEAD + recursive tree
↓
若有新工作：繼續處理並再次確認
↓
若 pending 為空：驗證 included source SHA
↓
create_file 到 ai-results/ai-output（唯一 GitHub write）
↓
GitHub Actions finalizer 套用到 main
↓
本次 ChatGPT execution 結束
```

### 達 bundle transport safety ceiling

```text
持續處理 pending
↓
下一個完整 result 會令 bundle 超過 safety ceiling
↓
停止增加本次 bundle
↓
重新驗證 included source SHA
↓
create_file 到 ai-results/ai-output
↓
本次結束；剩餘 pending 下一次繼續
```

最終聊天報告至少說明：

- 本次完成 AI 分析並提交的 source 數。
- `ai-results` bundle 路徑。
- output 總數。
- 因 source 變更而排除的 result 數。
- 是否因 bundle safety ceiling 留下 pending。
- 最終可見 pending 概況。

如果 execution 開始及最終確認都沒有任何可處理 source，則不建立空 bundle。
