# stock-ai-tasks

`stock-ai-tasks` 是 Spring Boot 與 ChatGPT 之間的 Git 任務佇列。

這個 repository 只定義 **AI task framework 的共通規則**。每一個頂層 task folder 都代表一個獨立任務；任務自己的資料格式、欄位定義、AI 可修改範圍、判斷規則及其他限制，必須寫在該 task folder 內的 `README.md`，不要寫在根目錄 README。

ChatGPT 透過已授權的 GitHub Connector 直接讀寫這個 private repository；正常排程不依賴 Desktop Commander 或本機 VM 才能存取 GitHub。

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

每次 execution 開始時，ChatGPT 必須：

1. 重新讀取 `main` 上最新的 repository 根目錄 `README.md`，並以本次讀到的內容作為 framework 唯一正式規則來源。
2. 讀取 `main` 最新 HEAD 與最新 recursive Git tree（Git Trees API，等同 `git/trees/main?recursive=1`），由 Git tree 動態找出所有頂層 task folders 與其實際狀態。
3. 在本次 execution 內建立工作狀態快取，至少保存目前已知的 `currentHead`、task folders、`processing/`、`pending/`、`finished/`、`imported/`、`error/` 狀態，以及已讀 README 的 blob SHA。
4. 第一次要讀取、恢復或處理某個 task 時，先閱讀該 task 最新 `README.md`；以該 README 為唯一 task-specific 規格來源。
5. root README 與 task README 在同一次 execution 內讀取後應快取並重用，不得因每完成一個 batch 就無條件重新讀取相同 README。
6. 不要把某一個 task 的規則套用到其他 task。
7. 如果 task folder 缺少 README，或 README 無法清楚判斷如何安全處理資料，停止處理該 task，不要自行猜測規則。

## Execution 內快取與 Repository 掃描規則

GitHub 上的最新 repository 狀態始終是最終真實來源，但同一次 execution 內不得為每個 batch 無條件重複拉取完整 recursive Git tree。

正常流程：

1. execution 開始時完整讀取一次最新 HEAD 與 recursive Git tree。
2. 依該 tree 建立本次 execution 的記憶體工作狀態。
3. 每次 Git commit / ref update 成功後，立即在本次 execution 內更新 `currentHead` 與受影響 batch 的快取狀態，例如 `pending → processing` 或 `processing → finished`。
4. 若前一次寫入已確認成功，且沒有競態、HEAD mismatch、non-fast-forward、來源／目的狀態異常或其他安全疑點，直接使用更新後的 execution 內狀態選擇下一個工作，不重新取得完整 recursive tree。
5. 只有在下列情況才需要中途重新取得最新 HEAD 與 recursive Git tree：
   - `expected SHA mismatch`、non-fast-forward 或其他證據顯示 main HEAD 已被其他合法 writer 更新。
   - batch 的來源／目的狀態與 execution 內快取不一致。
   - GitHub 回傳結果使目前狀態無法安全判定。
   - framework 其他規則明確要求重新確認實際 repository 狀態。
6. 若中途重新掃描後發現 root README 或某個已快取 task README 的 blob SHA **沒有改變**，應繼續使用已快取內容，不要重新讀取同一份 README；只有 blob SHA 改變時才重新讀取該 README 並以新規則為準。
7. 當 execution 內狀態判斷已沒有可處理工作時，結束前必須再完整讀取一次最新 HEAD 與 recursive Git tree 作最終確認。
8. 最終重新掃描若發現新增 pending、可安全恢復的 processing 或其他合法 writer 帶來的新工作，更新 execution 內狀態後繼續處理；只有最新完整掃描確認沒有工作時才正常結束。

必須遵守：

- 由 Git tree 動態找出所有頂層 task folders，不得 hardcode task folder 名稱。
- 由 Git tree 檢查每個 task 的 `processing/`、`pending/`、`finished/`、`imported/`、`error/` 實際內容。
- 不得因 code search 沒有結果就判定 repository 無法掃描或沒有工作。
- 不得猜測 batch 檔名或路徑。
- 不得依賴 `manifest.json`；repository 不需要以 manifest 作為工作發現或狀態來源。
- 重新取得 Git tree 後，若實際狀態與 execution 內快取不同，一律以最新 Git tree 為準，再依 Git 競態與安全規則 reconcile。

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
- `error/`：主要用於資料格式、識別碼、schema 或其他無法安全繼續的永久性問題；另外允許存放本 README 定義的 `execution-report-*.md` 執行層錯誤報告。執行報告不代表 batch 本身進入 error 狀態。

若某個 task README 明確定義不同流程，以該 task README 為準；但 `execution-report-*.md` 屬 framework 層的執行診斷紀錄，除非 task README 明確禁止非 CSV 檔案，否則可依下方規則寫入該 task 的 `error/`。

## ChatGPT 共通執行規則

對每一個可處理的 task：

1. 第一次進入該 task 時先閱讀 task README；同一次 execution 內依上方快取規則重用。
2. 按檔名／時間由舊到新處理 `pending/`。
3. 每次 execution 必須持續處理多個可處理的 pending files，不得只完成一個 batch 就無故停止。
4. 正常目標是持續處理到所有具有效 README 的 task 都沒有可處理 pending，且沒有可安全恢復的 processing 工作；**不得設定固定 batch 上限**。只有實際時間、工具、平台、GitHub、網路、服務限制，無法安全解決的競態／衝突，task 規則不足，或 framework 明確要求停止時，才可提前安全結束。
5. 開始處理一個檔案前，先把它從 `pending/` 移到 `processing/` 並 commit + push。
6. 嚴格按照 task README 修改內容。
7. 完成後，把同名檔案從 `processing/` 移到 `finished/`，再 commit + push。
8. 每次成功完成狀態轉移後，更新本次 execution 的 `currentHead` 與工作狀態快取；若沒有競態或安全疑點，立即依快取繼續下一個工作，不無條件 full rescan。
9. 如果本次仍有可用執行能力，立即繼續下一個 pending file；不要因完成一個 batch 就主動停止。
10. 當快取顯示工作已清空時，再依「Execution 內快取與 Repository 掃描規則」做一次最新 full rescan；若出現新工作則繼續，否則正常結束。

如果遇到暫時性的 GitHub、網路或服務錯誤：

- 不要把未完成檔案標記為 `finished`。
- 保留檔案在符合 framework 與實際安全狀態的位置；若已成功領取至 `processing/`，不得為了美化狀態而猜測性移回或前移。
- 若錯誤使 repository 狀態無法安全判定，重新取得最新 HEAD/tree；若仍無法安全繼續，停止本次 execution。
- 若仍有未完成工作，依下方「提前停止執行錯誤報告」規則記錄本次可核實狀態。

如果遇到資料本身的永久性問題：

- 按 task README 的規則處理。
- 若沒有更具體規則，可把該 batch 檔案移到 `error/` 並記錄原因，然後繼續其他可處理的 pending files。
- batch 是否進入 `error/`、以及是否增加 `errorCount`，只由 task README 與 batch 的實際資料結果決定；不得因 execution report 的存在而增加 `errorCount`。

## Git 原則

- 所有 AI task 變更直接針對 `main`，除非某個 task README 另有規定。
- 每次狀態轉移或完成處理後都要 commit + push，讓 repository 狀態可追蹤及可恢復。
- `pending → processing` 與 `processing → finished` 必須是兩個分開的狀態轉移，不得一次跨越兩個狀態。
- 同一 repository 內的 batch 狀態 move 應以單一 Git commit 同時完成來源刪除與目的建立，避免「先 create 再 delete」造成中間重複狀態。
- 不得 force push。
- 每次準備寫入時，把當下 execution 已知的 `currentHead` 記為本次寫入的 `baseHead`。新的 tree 與 commit 必須以該 `baseHead` 為 base／parent 建立。
- 對 `main` 執行 `update_ref` 時，**必須**使用新 commit SHA 作為 `sha`、建立該 commit 時的 `baseHead` 作為 `expected_sha`，並保持 `force=false`。不得省略 `expected_sha`，不得使用 `force=true` 繞過競態或安全檢查。
- `update_ref` 成功後，立即把新 commit SHA 設為新的 `currentHead`，並更新 execution 內受影響 batch 的快取狀態。
- 只有明確出現 `expected SHA mismatch`、non-fast-forward，或重新讀取後確認 `main` HEAD 已被其他合法 writer 推進，才視為**可重試的 Git race**。此時必須重新讀取最新 HEAD 與 recursive Git tree，確認來源／目的路徑、batch 內容與實際狀態仍可安全判定；若安全，必須以新的 HEAD 重新建立 tree + commit，再嘗試 `update_ref`，不得反覆提交原本以舊 parent 建立的同一 commit。每一個狀態轉移最多重試 3 次。
- 若 `update_ref` 被 GitHub Connector 的 safety check、policy 或其他寫入安全機制阻擋，但重新讀取後 `main` HEAD **沒有改變**，且沒有 `expected SHA mismatch`／non-fast-forward，則**不屬於 Git race**。可重新確認一次最新 HEAD/tree 與 batch 實際狀態；若相同安全阻擋再次出現，不要把相同操作硬重試到 3 次、不要 force push，應保留 batch 在最後可核實的安全狀態並停止本次 execution，依「提前停止執行錯誤報告」記錄原因，下一次 execution 再從最新狀態恢復。
- 若來源與目的同時存在、內容不一致、batch 狀態無法安全判定，或真實 Git race 連續超過 3 次，停止本次 execution；不得猜測清理或覆蓋。
- 不要修改目前 task 以外的檔案，除非是本 README 明確要求的 repository-level 掃描或 execution report。
- 不要修改其他 task 的 README 或資料，除非該次任務明確要求。
- 不要自行改變 task folder 結構、batch 檔名規則或資料 schema。

## 提前停止執行錯誤報告

若一次 AI execution 在仍有 `pending/` 或未完成工作時，因可核實的真實阻礙而提前停止，必須把一份獨立的執行錯誤報告寫入**實際發生阻礙的 task**之 `error/` 目錄。

適用的真實阻礙包括：

- 實際執行時間、工具或平台限制。
- 無法安全解決的 GitHub 競態、non-fast-forward、expected SHA mismatch 或內容衝突。
- GitHub Connector safety check／policy 阻擋，且重新確認 HEAD/tree 後仍無法安全完成寫入。
- 暫時性的 GitHub、Connector、網路或服務錯誤。
- task 缺少 README，或規則不足以安全處理。
- framework 明確規定必須停止的其他情況。

報告規則：

1. 使用 Markdown，不得使用 CSV，避免被 Spring Boot importer 誤認為 batch。
2. 檔名格式：`execution-report-YYYYMMDD-HHmmss.md`。
3. 寫入位置：`<task-name>/error/`。
4. 報告是 execution-level 診斷紀錄，不得取代、移動、改名或修改任何 batch CSV。
5. 產生 execution report 本身**不得增加 `errorCount`**，也不得將正常 batch 標記為 error。
6. 寫報告前必須重新取得 `main` 最新 HEAD 與最新 recursive Git tree，以最新可核實狀態為準。
7. 報告本身應使用獨立 Git commit 寫入 `main`，以最新 HEAD 為 base，且同樣必須使用 `expected_sha=baseHead`、`force=false` 的安全 ref update 規則。
8. 若寫入報告時遇到真正的合法 Git race，重新取得最新 HEAD 後，在內容仍成立的前提下依 Git 原則安全重建 commit 並最多重試 3 次。
9. 若 GitHub／Connector／工具故障或 safety block 嚴重到連報告都無法安全寫入，不得猜測修復或 force push；最終聊天報告必須明確指出 execution report 未能寫入及可核實原因。
10. 如果本次正常處理到所有可處理工作清空，不建立 execution error report。

每份 execution report 至少包含：

- task 名稱。
- 本次 execution 時間。
- 本次全 repository `completedCount`。
- 本次全 repository `errorCount`。
- 該 task 本次成功完成的 batch 數。
- 全 repository 最後一次掃描時剩餘 pending 合計。
- 該 task 最後一次掃描時剩餘 pending 數。
- 可核實的提前停止原因，不得以推測代替事實。
- 若涉及 Git 寫入，應註明屬於 `expected SHA mismatch/non-fast-forward`、Connector safety block/policy，或其他已核實類型；不得把 safety block 猜成 Git race。
- 是否留下 `processing/` 工作；若有，列出實際檔名。
- 最後一次重新掃描時，與阻礙相關 batch 的實際狀態。
- 最後可核實的 `main` HEAD／commit SHA（工具可取得時）。
- 下一次 execution 應如何安全恢復，例如重新掃描最新 tree、優先恢復既有 processing，再繼續最舊 pending。

如果同一次 execution 有多個 task 各自發生獨立且可核實的阻礙，則各受影響 task 各寫一份 execution report；不要把與該 task 無關的推測寫進其報告。

## 結束報告

如本次有實際處理，最終聊天報告至少應包含：

- `completedCount`
- `errorCount`
- 所有 task 合計剩餘 pending 數
- 各 task 本次完成／剩餘結果

若仍有 pending 或未完成工作而停止，還必須說明：

- 可核實的提前停止原因。
- 是否留下 processing 工作。
- execution report 寫入的實際 task/error 路徑；若無法寫入，明確說明原因。

若 execution 開始時已確認所有 task 都沒有可處理 pending，且沒有可安全恢復的 processing，則不需要通知，也不要建立 execution error report。

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