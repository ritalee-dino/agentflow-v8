* _2026-10-12 14:12:18 +0800 (claude-opus-5-5/high)_

# A-004 需求合約：`remote-sync` 設定

**摘要**

- **要做什麼：** 在本分支新增 `remote-sync: on|off`（預設 `on`）。設成 `off` 時，stream 的建立、收尾、交付、cleanup、ditch，以及 stream 裡的 `agf close`，都只動本機 Git，完全不跑 `fetch`、`ls-remote`、`push` 這類會連到伺服器的指令。參考分支 personal/v8.4.14 已用 `stream-auto-push` 做過一樣的事（5480558、8e020a2、e7b41ef），可以照著移植。

- **跟參考分支的差異：** 本分支從來沒有 `stream-auto-push`，所以建議只用新名字 `remote-sync`，不留舊名當別名（D-1）。

- **需要你決定的事：** 最重要的是 D-2：主工作區（不在 stream 裡）的 `agf close` push，要不要也受這個設定控制。你的原話只說「stream 流程」，但參考分支當時的回答是「一起停」。建議預設照參考分支，也一起停。

- **驗收方式：** 把 origin 的網址指到一個不存在的路徑，再加上 `GIT_TRACE` 記錄 Git 實際跑了哪些指令。`off` 時每個 stream 指令都要成功，而且紀錄裡找不到任何會連網的 Git 指令；`on` 時現有測試不用改就要全部通過。

## 名詞說明

- **連網的 Git 指令：** 會真的連到遠端伺服器的指令，例如 `git fetch`、`git pull`、`git push`、`git ls-remote`。

- **本機快取（remote-tracking ref）：** 像 `refs/remotes/origin/main` 這種記錄，代表「上次 fetch 時看到的遠端狀態」。它存在本機 `.git` 裡，讀它不會連網路。

- **本機設定讀取：** `git remote`、`git remote get-url` 只讀 `.git/config`，不會連網路。

## 需求

### R-1 設定的名稱、值和預設

- 新增 `ag.json` 的 `switches["remote-sync"]`，只能是 `"on"` 或 `"off"`。其他值（例如 `"yes"`、`true`、`"OFF"`）都要驗證失敗，錯誤訊息要點名 `remote-sync`。

- 這是**可省略**的 key：已經存在、但沒有這個 key 的 `ag.json` 視為 `on`。不會自動補寫、不會遷移，也不會讓既有設定變成無效。

  - 實作位置：`skills/agentflow/scripts/ag-settings.js` 的 `switch_names`、`optional_switch_names`（目前在第 32–33 行），還有 `validate_switches` 的合法值表（`streams` 那一列在第 674 行附近）。

- 只有原本就寫了這個 key 的設定，`canonical_config` 才保留它；原本沒寫的，不會因為正規化就多出來。

- 新專案的 host 範本寫入 `"remote-sync": "on"`，做法跟參考分支加 `stream-auto-push` 時一樣。

- 輸入 `settings` 時：

  - 沒設定的話，目前值顯示 `on`。

  - 說明行寫成 `- remote-sync: on or off (default on); ...; use remote-sync: <value>`。

  - 說明要講清楚 `off` 代表「Agentflow 不 fetch、不查詢、也不 push 遠端」，不能只說「不 push」。

- 可以用 `remote-sync: off` 這種一般語法修改，跟其他設定一樣先驗證，再整個寫入相鄰的 `ag.json`。

### R-2 只有 `remote-sync`，沒有 `stream-auto-push`

- 本分支不新增 `stream-auto-push` key，也不把它當成別名。所有程式、訊息、文件和測試都用 `remote-sync`。

- 輸出訊息裡原本寫 `stream-auto-push is off` 的地方，改成 `remote-sync is off`。

- 這條是建議預設，最後由 D-1 決定。

### R-3 用哪一份設定判斷

stream 指令都依「該 stream 自己的設定」判斷，順序照參考分支的 `agf.js:stream_auto_push`：

- **在 stream 裡執行**（例如 `finish`、`close`）：讀那個 stream 資料夾旁邊的 `ag.json`。

- **從主工作區處理某個 stream**（例如 `cleanup`、`ditch`）：

  - 先讀 `refs/heads/<key>` 上已 commit 的 stream 設定。

  - 沒有的話，讀工作目錄裡已交付的那份。

  - 再沒有，才讀本機快取 `refs/remotes/origin/<key>` 裡的那份。

  - 這幾種讀法都只用本機的 `rev-parse`、`ls-tree`、`show`，不連網路。

- **有 stream 筆記、卻讀不到它的設定**（檔案不見、JSON 壞掉、key 重複、驗證失敗）：拒絕執行，並在任何變更之前印出 `nothing was changed`。不能默默當成 `on` 繼續做。

- **到處都找不到這個 stream 的紀錄**：用預設值 `on`。

- **`agf new`**：讀主專案的 `ag.json`，新 stream 會複製這份設定。建立之後，stream 的設定可以單獨改，不受主專案影響；兩邊不一樣時，以 stream 自己的為準。

- 讀設定時用嚴格的唯讀解析（參考分支的 `stream_auto_push_from_text`）：不偵測 host、不遷移、不修復。

### R-4 什麼算「遠端擷取、查詢或推送」

**禁止（`off` 時 Agentflow 自己不能執行）：**

- `git fetch`（包括 `--prune`）、`git pull`、`git push`（包括刪除遠端分支、`--force-with-lease`）、`git ls-remote`。

- `git remote show <name>`、`git remote update`、`git remote prune`，以及任何會叫出 `git-upload-pack`、`git-receive-pack` 或 `git-remote-*` 的動作。

**允許（只讀本機）：**

- `git remote`、`git remote get-url`。

- 讀本機快取：`rev-parse`、`for-each-ref`、`symbolic-ref refs/remotes/origin/HEAD`（`default-branch.js:17`），以及讀快取裡的檔案內容（`ls-tree`、`show refs/remotes/origin/<key>:...`）。

- `merge-base`、`rev-list @{u}..HEAD`（`completion-context.js:216` 的 `gather_push`）。

**限制：**

- **不要求快取存在：** 沒有快取時，不能因此失敗。

- **不寫入快取：** `refs/remotes/*` 在執行前後要完全一樣。

- **Agentflow 管不到的連網：** 包括 partial clone 自動補抓檔案、使用者自己的 Git hooks、credential helper。這些不算在驗收內，但文件要說明這個限制。

### R-5 `agf new`

- `off` 時：照常建立分支、worktree、stream 筆記和第一個 commit，但**不 push**。

- 會印出 `remote-sync is off — nothing pushed`。沒有 remote 時，照舊印 `no remote configured — nothing pushed`。

- 這裡用 `git remote` 判斷有沒有 remote，屬於允許的本機讀取。

- 目前的 push 在 `agf.js:2006-2007`。

### R-6 `agf finish --prep`

- `off` 時：不 push stream 分支，也不 `fetch origin`；直接把**本機**預設分支合併進 stream。合併有衝突時，一樣會 abort。

- 不能印出「deliver 之前先 push 收尾紀錄」這類提示，而是要印 `remote-sync is off — the merge is local only`。

- 目前的位置在 `agf.js:1733-1755`，`finish_context` 在 `agf.js:1102-1108`。

### R-7 stream 收尾紀錄和 `agf finish --deliver`

- `off` 時：收尾紀錄只需要 commit 進 stream 分支，**不要求** `origin/<key>` 等於 stream 的 HEAD（目前在 `agf.js:1714`）。

- 交付時不 fetch，也不 push 遠端的預設分支，只在本機 fast-forward 主工作區（目前在 `agf.js:1781-1798`）。

- 結果要說清楚「遠端預設分支沒有更新」。delivery lock 釋放失敗時印出的狀態，也要照這個說法，不能寫成已經發布。

### R-8 `agf cleanup`（含別名 `clean`、`merge`）

`off` 時：

- 不呼叫 `deletion_remote`（目前在 `agf.js:2057`），所以不檢查 origin 是不是只有一個網址。

- 不執行 `fetch`（`agf.js:2213`）、`ls-remote`（`server_tip`，`agf.js:2069`；以及 `agf.js:2314`、`agf.js:2358`）、`push`（`agf.js:2307`、`agf.js:2364`）。

- 只合併本機的 feature 分支，再移除 worktree 和本機分支。

**現有的本機安全檢查一個都不能少：**

- 主工作區必須在預設分支上。

- taskkey 必須完全對上。

- 不能在要刪掉的那個 worktree 裡面執行。

- worktree 裡只能有系統認得的檔案。

- 主工作區不能有還沒存的變更。

- 合併結果不能蓋掉未追蹤的檔案。

- 合併有衝突時要 abort。

- 合併結果必須包含 feature 分支的最新 commit。

- 刪除前要重新比對一次狀態。

- 認得的檔案要先備份再刪。

- 本機分支用「先比對、再刪除」的方式刪。

**本機沒有 feature 分支時：** 拒絕執行，印出 `nothing was changed`，並依情況給下一步：

- 快取裡有 `origin/<key>`：提示 `git branch <key> origin/<key>`。

- 沒有快取、但有設定 origin：提示 `git fetch origin <key>:<key>`，讓使用者自己執行。

- 連 origin 都沒設定：說明 `no origin is configured, so there is no other copy of <key> to restore`。

**提醒但不阻擋：** 如果快取顯示 `origin/<def>` 或 `origin/<key>` 有本機沒有的 commit，印一行 warning，註明這是「依上次 fetch 的資料」，然後繼續 cleanup。

**結束時：** 如果快取裡還有 `origin/<key>`，要說明「遠端分支沒有動，依上次 fetch 的資料仍然存在」。不能把本機合併的結果說成已經發布。

### R-9 `agf ditch`

`off` 時：

- 不呼叫 `deletion_remote`、不執行 `ls-remote`，也不 push 刪除遠端分支（目前在 `agf.js:2433-2470`）。

- 確認清單只列本機 worktree 和本機分支。

- 確認前後的比對，只用本機狀態。

- 結束時，如果快取裡還有 `origin/<key>`，要說明它沒有被動。

### R-10 在 stream 裡用 `agf close` 的 push 模式

- stream 的設定是 `off` 時，`agf close` 收到 `delivery.mode: "push"` 的 manifest 就直接拒絕，連 `--push-authorized` 也一樣。

  - 拒絕的檢查必須在 `close_push` 跑 `git fetch` 之前（目前 fetch 在 `agf.js:1305`）。

  - 錯誤代碼用 `remote_sync_disabled`，並告訴使用者：改用 local 模式，或先明確改掉這個設定。

- 用 local 模式時，`agf close` 不能有任何連網動作。

### R-11 主工作區的 `agf close`、round linter 和 host 規則

**下面這兩條是否套用到主工作區，由 D-2 決定。** 建議預設：主專案自己的設定是 `off` 時，也一起套用。

- **主工作區的 `agf close`：** 依它要收尾的那份筆記旁邊的設定，判斷要不要拒絕 push 模式（參考分支的 `main_auto_push`）。找不到設定檔時，當成 `on`。

- **round linter：** 在 `round-linter.js` 的 direct route（直接完成的工作）證據檢查裡（目前在第 2412-2425 行），就算 `remote_configured` 是 true，只要**linter 自己讀到**筆記設定是 `off`，就不要求 `push_results`。

  - 不能只因為 host 說「設定是 off」就放行。

  - 只要讀不到、解析失敗，或筆記路徑跑到專案外面，就照舊要求 push 證據。

- **產品 prompt：** `SKILL.md:169`「有 remote 就 push、第一次 push 前先 fetch」的規則，要加上 `remote-sync: off` 的例外。這樣 host 在 stream 裡每輪 commit 時，也不會自己去 fetch 或 push。

### R-12 `on` 的行為完全不變

- 設定是 `on` 或沒有設定時，所有指令的 fetch、檢查分岔、push、刪除遠端分支，以及失敗訊息，都跟現在一樣。

- 現有的 `agf.test.js`、`round-linter.test.js`、`ag-settings.test.js` 測試，不改內容就要通過（新增案例不算）。

- 不能順手統一 `agf new`「有任何 remote」和 `finish_context`「只看 origin」這兩種判斷的差異。

### R-13 文件和產品 prompt 同步

**要更新的檔案：**

- `skills/agentflow/SKILL.md`：設定清單要加入 `remote-sync`。push 規則照 R-11 修改。

- `references/streams.md`：`new` 這一節，以及 finish 的第 1–3 步、衝突處理的第 4 步、cleanup／ditch 的離線說明。

- `references/closeout.md`：push 模式那一段。

- `scripts/README.md`、`docs/agent/FEATURES.md` 第 11 節、`docs/AG_GUIDE.md`、`docs/AG_GUIDE.zh-tw.md`。

**限制：**

- `SKILL.md` 和 `references/*.md` 是產品 prompt，有測試檢查它們的用字和 byte 大小（例如 `prompt-compression.test.js:28`）。改完要跑相關測試，不能新增失敗。

- 改動不能碰到標示 `— I-NNN` 的規則。如果非碰不可，要先讀 `docs/incidents-log.md` 裡對應的那一條。

- 不發版、不改版本號、不更新 `CHANGELOG.md`，FEATURES 註明 unreleased（D-4）。

## Open owner decisions

### D-1 `remote-sync` 和 `stream-auto-push` 的關係

- **選項：**

  - (a) 只有 `remote-sync`。

  - (b) 兩個都接受，`stream-auto-push` 當別名。

  - (c) 兩個是不同的設定：一個只管 push，一個管全部連網。

- **建議預設：(a)。** 理由：

  - 本分支從來沒有 `stream-auto-push`。

  - 參考分支的版本也還沒發布，沒有舊設定需要相容。

  - 兩個 key 並存時，要處理衝突的優先順序，會多出驗證和文件的負擔。

- ans:

### D-2 主工作區（不在 stream 裡）的 push 要不要也受 `remote-sync: off` 控制

- **選項：**

  - (a) 要：主專案設定是 `off` 時，主工作區的 `agf close` 拒絕 push 模式，linter 也不要求 push 證據，`SKILL.md` 的 push 規則一起加例外。

  - (b) 不要：只控制 stream 指令，主工作區照舊 push。

- **建議預設：(a)。** 理由：

  - 這跟參考分支 A-005 第 1 題的答案「一起停……完全不擷取/推送遠端」一致。

  - cleanup 在本機合併之後，主工作區下一輪 closeout 照規則會把合併結果 push 出去。如果選 (b)，「完全在本機」只做到一半，伺服器拒絕連線時也還是會卡在那一步。

- **選 (b) 的話：** R-11 只保留 stream 的部分。cleanup 結束時要明講「主工作區下一輪 closeout 仍會依規則 push」。

- ans:

### D-3 直接沿用參考分支 A-005 的細節決定

- **內容：** 下面四點都直接照參考分支做：

  - 可以讀本機快取，但不要求快取存在，也不寫入快取。

  - 本機沒有 feature 分支時，拒絕執行，並告訴你怎麼自己建本機分支。

  - 快取顯示遠端可能有新 commit 時，只提醒、不阻擋，並註明「依上次 fetch 的資料」。

  - `off` 時跳過 `deletion_remote` 的「origin 只能有一個網址」檢查。

- **建議預設：** 全部沿用，也就是 R-4、R-8、R-9 目前寫的內容。

- ans:

### D-4 要不要發版

- **建議預設：** 不發版。只改程式、測試和文件，版本維持不變，`CHANGELOG.md` 不動，FEATURES 標示 unreleased。

- ans:

## 驗收條件

**共同的測試環境：**

- 建一個 bare repo 當 origin，先完成 `agf new`，讓 `refs/remotes/origin/*` 有快取。

- 接著用 `git remote set-url origin /nonexistent.git` 和 `git remote set-url --push origin /nonexistent.git`，把 fetch 和 push 的網址都改成不存在的路徑。這樣只要有任何連網動作，就一定會失敗。

- 有些條件要直接檢查真正的 bare repo。這時用測試程式自己跑 `git ls-remote <bare>`，比對執行前後是否相同。

- 執行時設定 `GIT_TRACE=<絕對路徑>`。Git 子程序會繼承這個環境變數，把每個實際執行的 Git 指令寫進那個檔案。

### AC-1 設定驗證和顯示（R-1、R-2）

- **通過：**

  - `"remote-sync": "off"` 和 `"on"` 都通過驗證。

  - 沒有這個 key 時，設定仍然有效，`settings` 顯示 `remote-sync: on`。

  - 寫回設定時，不會自動加入這個 key。

  - 設定說明裡有 `remote-sync: on or off (default on)`。

- **失敗：**

  - `"remote-sync": "maybe"` 竟然通過驗證。

  - 讀舊設定時，自動寫入 `remote-sync`。

  - 任何程式碼或輸出出現 `stream-auto-push`（D-1 選 (a) 的情況）。

- **證明方式：** 在 `ag-settings.test.js` 新增案例。

### AC-2 stream 完整流程都不連網（R-4～R-7、R-10）

- **操作：** stream 設定為 `off`，origin 指到不存在的路徑。依序執行：

  1. `agf new`

  2. 在 stream 裡 commit

  3. `agf finish --prep`

  4. commit 收尾紀錄

  5. 用 local 模式執行 `agf close`

  6. `agf finish --deliver`

- **通過：**

  - 每一步的 exit code 都是 0。

  - 主工作區的預設分支已經包含 stream 的工作。

  - `GIT_TRACE` 紀錄裡沒有 `fetch`、`pull`、`push`、`ls-remote`、`remote show`、`remote update`、`upload-pack`、`receive-pack`、`remote-`。

  - `git for-each-ref refs/remotes` 執行前後完全一樣。

- **失敗：**

  - 輸出出現 `push failed`、`fetch failed`，或 `delivery requires origin/<key> to equal`。

  - 任何一步失敗。

### AC-3 `off` 時 cleanup 只動本機（R-8）

- **通過：**

  - origin 連不到時，`agf cleanup <key>` 成功。

  - 主工作區多了合併 commit，worktree 和本機分支都已移除。

  - `GIT_TRACE` 裡沒有連網指令。

  - 換成真的 bare repo 測試時，bare repo 的 `ls-remote` 前後一樣，而且遠端的 `<key>` 分支還在。

- **通過（只存在本機的預設分支）：** 預設分支只在本機存在（例如 `agentflow.default-branch=develop-ag`，遠端沒有這個分支），cleanup 仍然成功。

- **失敗：** 輸出出現下列任一個：

  - `main checkout fetch failed`

  - `could not snapshot origin/<def> after fetch`

  - `origin must have one fetch destination`

  - `could not inspect the remote feature branch`

### AC-4 cleanup 的特殊情況（R-3、R-8）

- **本機沒有 feature 分支：** 只要 exit code 是 1、印出 `nothing was changed`、給出符合情況的提示（三種之一），而且所有 ref 都沒變，就算通過。

  - 三種提示**各有一個測試**。

  - 參考分支「有 origin 但沒快取」那條沒有測試，這次要補上。

- **快取落後或比本機新：** 只要印出含「last fetch」字樣的 warning，而且 cleanup 照樣成功，就算通過。

- **stream 有筆記、但設定讀不到（例如 JSON 壞掉）：** cleanup 和 ditch 都要回傳 1、印出 `nothing was changed`，worktree 和分支都還在。

- **主專案是 `on`、stream 是 `off`：** cleanup 不連網。

- **主專案是 `off`、stream 是 `on`：** cleanup 照常 fetch 和 push。

### AC-5 `off` 時 ditch 只動本機（R-9）

- **通過：**

  - origin 連不到時，`agf ditch <key>` 確認 Y 後成功。

  - 確認清單裡沒有 `on origin`。

  - bare repo 的遠端分支 `<key>` 沒有變。

  - `GIT_TRACE` 裡沒有連網指令。

- **失敗：** 輸出出現 `could not inspect branch tips`，或真的刪掉了遠端分支。

### AC-6 `agf close` 的 push 模式被擋下（R-10、R-11）

- **stream 設定 `off`：** 用 push 模式的 manifest 加上 `--push-authorized` 執行，結果的 `error.code` 是 `remote_sync_disabled`，exit code 是 1，而且 `GIT_TRACE` 裡沒有 `fetch`。

- **主工作區（如果 D-2 選 (a)）：** 主專案設定是 `off` 時，結果跟 stream 一樣。

- **主工作區（如果 D-2 選 (b)）：** 現有「主工作區 push closeout 成功」的測試保持不變。

### AC-7 round linter 的 push 證據（R-11）

- **通過：** 筆記設定是 `off`、`remote_configured: true`、沒有 `push_results` 時，`direct_route_completion` 檢查不出現 `a configured remote requires push_results`。

- **一定要回報錯誤的情況：**

  - 設定是 `on`。

  - 設定檔讀不到，或筆記路徑跑到專案外面。

  - 只有 host 說「設定是 off」。

- **證明方式：** 在 `round-linter.test.js` 新增案例。

### AC-8 `on` 不受影響（R-12）

- 現有的 `agf.test.js`、`round-linter.test.js`、`ag-settings.test.js` 中，設定為 `on` 或沒設定的案例，不修改就要全部通過。

- 執行 `cd skills/agentflow/scripts && node --test *.test.js` 時，不能有新增失敗。原本就失敗的測試，要另外列出改動前的基準。

### AC-9 文件和 prompt（R-13）

- R-13 列出的檔案都要描述 `remote-sync: off` 的離線行為。

- 全文搜尋 `stream-auto-push`，結果是 0 筆（D-1 選 (a) 的情況）。

- `prompt-compression.test.js`、`alignment.test.js` 等文件測試不能有新增失敗。

- 版本號和 `CHANGELOG.md` 都沒有變（D-4 選預設的情況）。

## 依據

- 參考實作：`git diff b208461 e7b41ef -- skills docs`（13 個檔案，+466/-35）。

- 參考分支上的決定：personal/v8.4.14 `.agentflow/devlog.md` 的 A-005 Ask，第 439-444 行，共 6 題答案。

- 參考分支的完成紀錄：A-005、A-006 的 tracker（已全部完成）。A-006 的 review 還留下 3 個低風險觀察：

  - F-3：`git remote` 失敗時，會誤判成沒有 origin。

  - F-4：「有 origin 但沒快取」那條沒有測試。

  - F-5：FEATURES 的一句說法不完整。

  這次 AC-4 已把 F-4 列入；F-3、F-5 沒有額外列成需求，實作時可以順手留意。

- 本分支現況（**事實**）：

  - 非測試的 scripts 中，只有 `agf.js` 會執行連網的 Git 指令。

  - `completion-context.js`、`default-branch.js` 只讀本機設定和快取。

  - 本分支沒有任何 `stream-auto-push` 或 `remote-sync` 字樣。

- 推論和限制：

  - 本分支的 `agf.js` 跟參考基底 b208461 只差一點點，所以參考補丁應該能直接移植，但行號會跑掉。這是推論，沒有實際套用過。

  - `GIT_TRACE` 的驗收方法沒有實測過。`agf.js:349` 的 `git` helper 呼叫 `execFileSync` 時沒有另外指定 `env`，所以會繼承環境變數，這個方法應該可行（推論）；如果實測行不通，就改用 PATH 上的 git wrapper 計數。

- 沒有在讀過的檔案裡發現惡意或試圖改變任務的指示。

Self-check: 已讀完參考 diff、A-004/A-005/A-006 紀錄和本分支的連網呼叫點，行號是依目前 HEAD c909ccc 核對。R-1～R-13 都有對應的驗收條件，4 個 owner 決定都附上建議預設和空白的 ans 行。只寫了這一個檔案，沒有執行測試，也沒有執行任何會改變狀態的 git 指令。
