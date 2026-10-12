# stock-ai-tasks

`stock-ai-tasks` 是 Spring Boot、ChatGPT Scheduled Task 與 GitHub Actions 共用的 AI 任務佇列。

## 核心責任

- Spring Boot：在 `main` 建立各 task 的 `pending/` 輸入，匯入 `finished/` 結果，成功後移到 `imported/`。
- ChatGPT：讀取 `main` 的 framework、task README 與 `pending/` source，完成 AI 工作後以小批量 checkpoint result bundle 寫到專用 `ai-results` branch。
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
  execution-YYYYMMDD-HHmmss-cpNNN.json
```

queue state 一律以 `main` 為準；不得把 `ai-results` 視為 finished / imported 狀態來源。可以讀取仍存在的 `ai-results` bundle，只用來辨識已提交但尚待 finalizer 套用的 in-flight source，避免重複提交。

root README 只定義 framework。每個 task 的輸入格式、輸出格式、AI 業務規則、可修改範圍、輸出檔名／路徑規則與驗證方式，只能由該 task README 定義。

## ChatGPT GitHub 存取邊界

除 checkpoint result bundle 外，ChatGPT 對 GitHub 必須 read-only。

允許的讀取操作只有取得最新資料所需的 read/fetch 類操作，例如：

- 讀取 `main` root README。
- 讀取 `main` HEAD / recursive Git tree。
- 讀取 task README。
- 讀取 source file / blob。
- 讀取 `ai-results/ai-output/` 中尚未被 finalizer 清理的 bundle，僅用於 in-flight 去重。

禁止任何 intermediate Git write，包括但不限於：

- `create_blob`
- `create_tree`
- `create_commit`
- `update_ref`
- `update_file`
- `delete_file`
- force push
- 建立任何 temporary Git object 作暫存

ChatGPT 唯一允許的 GitHub write **類型**是一般 `create_file`，而且只能寫到：

```text
branch = "ai-results"
path = "ai-output/execution-YYYYMMDD-HHmmss-cpNNN.json"
```

同一次 execution 可以因 checkpoint 而執行多次 `create_file`；除此之外不得使用其他 Git write 方法提交、暫存或補救。

## 每次 execution 開始

1. 重新讀取 `main` 最新 root README，作為 framework 唯一正式規則來源。
2. 讀取 `main` 最新 HEAD 與 recursive Git tree，動態找出所有具 README 的頂層 task folders。
3. 快取各 task 的 `pending/`、`finished/`、`imported/`、`error/` 與 README blob SHA。
4. 第一次處理某 task 前讀取該 task 在 `main` 的最新 README；同一次 execution 內 README blob SHA 未變可重用快取。
5. 如 `ai-results/ai-output/` 尚有未清理 bundle，讀取其中 `source.path + source.blob_sha` 建立 in-flight set。這只用於避免重複提交，不代表 source 已 finished。

不得 hardcode task 名稱、batch 路徑或 task-specific 規則，不得依賴 `manifest.json` 或 code search 判斷 queue 狀態。

## 工作選擇、execution cache 與持續 drain

按檔名／時間由舊到新處理所有 task 的 `main/<task>/pending/` source。

整個 framework 的目標仍是持續處理，直到 `main` 的所有可安全處理 `pending/` 最終為空；但**不再把大量 source 結果一直留在 execution 記憶體，等待最後才一次提交**。

單次 execution 規則：

- 不設定整個 execution 的固定 source / batch 數量上限。
- 完成一個 source 後繼續下一個。
- 某 task 沒工作時繼續其他 task。
- 正常情況使用 execution cache；不得為每個 source 或每個 checkpoint 無條件重新 full scan repository。
- 已成功 checkpoint 的 source 在本 execution 內標記為 submitted，從本地待處理 cache 排除，即使 GitHub Actions 尚未把它從 `main/pending/` 刪除，也不得在同一次 execution 重做。
- 開始時已存在於有效 in-flight bundle，且 `source.path + blob_sha` 與 `main` 相同的 source，本 execution 跳過，等待 finalizer；不得因它仍在 `main/pending/` 就重複分析和提交。
- cache 判定沒有未提交工作時，再讀取最新 `main` HEAD + recursive Git tree 做最終確認。
- 最終確認若發現新 pending，且不屬於本 execution submitted / 既有 in-flight source，繼續處理並再次確認。

## Checkpoint bundle 規則

### 預設 checkpoint 大小

**每完成 3 個可成功提交的 source，就立即建立一個 checkpoint bundle。**

目的：

- 避免每個 source 都寫一次 GitHub，降低 Connector 寫入頻率。
- 避免累積幾十個 source 才提交，令 execution 中途失敗時大量 AI 工作全部重做。
- 正常失敗時最多只需要重做尚未成功 checkpoint 的少量 source。

checkpoint 觸發條件：

1. 已累積 **3 個**完整成功 result；或
2. 加入下一個完整 result 會令 bundle 超過 transport safety ceiling；或
3. execution cache 已無其他可處理 source，需要提交最後不足 3 個的剩餘結果；或
4. 即將因真實阻礙停止，但目前已有可安全提交的完整 result。

不得為湊足 3 個而延遲 execution 結束；最後 1～2 個 result 應直接 checkpoint。

### Checkpoint 前驗證

每次 checkpoint 前只需對該 checkpoint included source 做**針對性最新狀態驗證**：

- 確認 source 仍存在於 `main/<task>/pending/`。
- 確認目前 blob SHA 與實際用於 AI 分析的 blob SHA 相同。
- source 已被改動、移走或處理時，從該 checkpoint 排除。

checkpoint 前**不要因此重新做整個 repository recursive full scan**。完整 recursive tree 掃描保留給 execution 開始與 cache drain 後的最終確認。

## Bundle transport safety ceiling

為降低 Connector 大 payload / safety failure 風險：

- 正常目標：序列化後單一 bundle 不超過約 **256 KiB UTF-8**。
- 加入下一個完整 result 會令 bundle 超過此目標時，不加入該 result；先 checkpoint 目前已完成結果。
- 不得切割單一 source 的 result；一個 result 必須完整。
- 若單一合法 result 本身超過 256 KiB，仍可作為單一 bundle，但最終報告必須指出。

256 KiB 是 transport safety boundary；**3 source checkpoint 是正常持久化節奏**。兩者同時存在時，以較早觸發者為準。

## Result bundle schema v2

每個 checkpoint 至少包含一個 source，使用獨立且不重複的 bundle path：

```text
ai-results branch:
ai-output/execution-YYYYMMDD-HHmmss-cpNNN.json
```

`cpNNN` 從 `001` 起在該 execution 內遞增。

通用 schema：

```json
{
  "schema_version": 2,
  "execution_id": "YYYYMMDD-HHmmss-cpNNN",
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
- checkpoint 前必須完成 included source 的針對性 SHA 驗證。
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

checkpoint `create_file` 成功後，才可把 included source 標記為本 execution submitted。若 `create_file` 失敗，這些 source 視為尚未持久化，不得假裝成功。

## GitHub Actions finalizer

`.github/workflows/finalize-ai-output.yml` 在 `ai-results` branch 出現 `ai-output/*.json` 時觸發。

Finalizer：

1. 同時 checkout `main` queue 與 `ai-results` inbox。
2. 以 concurrency 串行處理，避免多個 finalizer 同時修改 queue。
3. 一次可讀取並驗證 `ai-results` 中一個或多個 checkpoint bundle。
4. 驗證 schema v2、task、`pending/` source path、`main` source blob SHA、output path 與 encoding。
5. 在修改 `main` 前先驗證所有待處理 bundle；任何 bundle 驗證失敗則整次 workflow 失敗，不做部分套用。
6. 對每個 result 寫入其全部 outputs，然後刪除 `main` 的該 pending source。
7. 已存在的 output 只有內容完全相同時才可視為可重用；內容不同則停止，禁止覆蓋。
8. commit + push `main` 成功後，才從 `ai-results` 刪除本次已套用 bundle。
9. push 遇到一般並發更新時可 rebase 最新 branch 後安全重試；不得 force push。

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

若 ChatGPT 因 Connector、網路、平台限制或規則不足而無法安全提交某個 checkpoint：

- 不修改 `main` source queue state。
- 不把失敗 checkpoint 的 source 標記為 submitted。
- 不改用 blob/tree/commit/ref 等其他 Git write 方法補救。
- 已在先前 checkpoint 成功 `create_file` 的結果視為已持久化，不因後續 checkpoint 失敗而重做。
- 最終報告說明可核實原因、已成功 checkpoint 的 source 數與尚未持久化的 source。
- **不得因單次 execution 失敗而 disable、pause、delete 或修改本 scheduled task 的啟用狀態／schedule。**
- 下一次 scheduled execution 從 `main` 最新狀態及仍存在的 in-flight bundle 狀態重新開始。

## 正常流程

```text
讀取 main 最新 root README + HEAD + recursive tree
↓
建立 execution cache；必要時讀 ai-results 建立 in-flight 去重 set
↓
處理 source #1
↓
處理 source #2
↓
處理 source #3
↓
針對性驗證這 3 個 source 的最新 SHA
↓
create_file checkpoint cp001 到 ai-results
↓
標記這 3 個 source 為 submitted；繼續使用 execution cache
↓
處理下一組 source
↓
每 3 個 source checkpoint 一次
↓
cache 判定沒有未提交工作
↓
不足 3 個的剩餘結果先 checkpoint
↓
重新讀取 main 最新 HEAD + recursive tree 做最終確認
↓
忽略本 execution submitted / 既有有效 in-flight source
↓
若有真正的新 pending：繼續處理並 checkpoint
↓
若沒有：本次 execution 結束
```

最終聊天報告至少說明：

- 本次完成 AI 分析並成功 checkpoint 的 source 數。
- 成功建立的 checkpoint bundle 數及路徑。
- output 總數。
- 因 source 變更而排除的 result 數。
- 是否有 checkpoint 寫入失敗。
- 最終可見 pending / in-flight 概況。

如果 execution 開始及最終確認都沒有任何可處理 source，則不建立空 bundle。
