* _2026-10-12 14:43:43 +0800 (claude-opus-5-5/high)_

# Codewalk：`remote-sync` 設定的遠端接觸點與參考修補對照

**摘要**

- **結論：** 參考分支的 `stream-auto-push` 修補（`git diff b208461 e7b41ef`）全部 13 個檔案、26 個 `agf.js` hunk 都能乾淨套到目前 HEAD（只有行號位移，沒有文字衝突）。它已經把 `agf new`、`finish --prep/--deliver`、`cleanup`、`ditch`、`close` 推送模式裡所有會連網的 Git 呼叫關掉。所以移植工作主要是「改名成 `remote-sync`」加上「補幾個小缺口」。

- **要注意的風險：** 目前 HEAD 本來就有測試是紅的，跟這個功能無關：`SKILL.md` 已經 36895 bytes，超過 32 KiB 上限；`away-gates` 預設值已經被改掉；Windows 不允許建立 symlink。另外這台電腦的全域 `pre-push` hook 會擋掉所有 push，所以跟遠端有關的測試一定要隔離全域 Git 設定才能跑。

- **還沒被關住的地方：** agent 自己手動執行的 `git push`（照 `SKILL.md:169` 的指示）只有文字提醒，程式不會擋；round linter 在 off 時看到「有成功 push」的證據也不會報錯；`docs/agent/FLOWS.md` 沒有更新。

- **需要 owner 決定：** 名稱 `remote-sync` 的範圍是否也要包含主工作區（不在 stream 內）的 `agf close` 推送（參考修補有包含）；預設值是否沿用 `on`；off 時錯誤訊息是否還能建議使用者手動 `git fetch`。

名詞說明（後面會一直用到）：

- **fetch**：從遠端伺服器把最新提交下載到本機。會連網。

- **ls-remote**：直接問遠端「某分支現在指到哪個提交」。會連網，但不下載內容。

- **push**：把本機提交上傳到遠端，或刪除遠端分支。會連網。

- **remote-tracking ref（`refs/remotes/origin/*`）**：上次 fetch 時留在本機的遠端快照。讀它只讀本機檔案，**不連網**。

- **`git remote`、`git remote get-url`**：讀本機 `.git/config` 裡的遠端設定。**不連網**。

---

## 1. 目前 HEAD 中，stream 指令會連網的 Git 呼叫

以下行號都是 HEAD `c909ccc` 的 `skills/agentflow/scripts/agf.js`，除非另外註明。「參考修補是否關住」是指 off 時參考修補會不會跳過這一行。

| 指令 | 位置（函式） | Git 參數 | 類型 | 參考修補 off 時 |
| --- | --- | --- | --- | --- |
| `agf new` | 2007（`new_main`） | `push -u origin <taskkey>` | push | 關住：條件改成 `has_remote && root_config.switches['stream-auto-push'] !== 'off'`（Fact） |
| `agf finish --prep` | 1734（`finish_main`） | `push origin HEAD:<key>` | push | 關住：`context.has_remote && context.auto_push`（Fact） |
| `agf finish --prep` | 1742（`finish_main`） | `fetch origin` | fetch | 關住：同一個 `if` 區塊（Fact） |
| `agf finish --prep` | 1749 | `merge --no-edit origin/<def>` | 本機（但依賴上一步 fetch） | 關住：off 時改走 1752 合併本機 `<def>`（Fact） |
| `agf finish --deliver` | 1714–1719（`closing_record`） | `rev-parse refs/remotes/origin/<key>` | 本機讀快照，但會拒絕「沒 push 過」的提交 | 關住：off 時不檢查（Fact） |
| `agf finish --deliver` | 1788（`finish_main`） | `push origin <head>:<def>` | push | 關住（Fact） |
| `agf finish --deliver` | 1798（`finish_main`） | `fetch origin`（在主工作區） | fetch | 關住（Fact） |
| `agf cleanup` / `clean` / `merge` | 2191 → 2057–2067（`deletion_remote`） | `remote`、`remote get-url --all origin`、`remote get-url --push --all origin` | 本機讀設定（不連網），但 URL 不一致時會拒絕 | 關住：off 時直接給 `{ url: '' }`，不檢查 URL（Fact） |
| `agf cleanup` | 2213（`clean_main` Step 1） | `fetch <url> +refs/heads/*:refs/remotes/origin/*` | fetch | 關住：`url` 是空的，所以 `has_remote` 為 false（Fact） |
| `agf cleanup` | 2237 → 2071（`server_tip`） | `ls-remote --refs <url> refs/heads/<key>` | ls-remote | 關住：`server_tip` 看到空 `url` 就直接回傳 `''`（第 2070 行，Fact） |
| `agf cleanup` | 2254 | `merge --ff-only <remote_tip>` | 本機（跟遠端對齊） | 關住：在同一個 `has_remote` 區塊（Fact） |
| `agf cleanup` | 2307（Step 3） | `push <url> <merged>:refs/heads/<def>` | push | 關住：`has_remote && auto_push`（Fact） |
| `agf cleanup` | 2314 | `ls-remote <url> refs/heads/<def>` | ls-remote | 關住：同一個區塊（Fact） |
| `agf cleanup` | 2328（sweep guard 呼叫 `server_tip`） | `ls-remote` | ls-remote | 關住：`url` 是空的（Fact） |
| `agf cleanup` | 2358 | `ls-remote <url> refs/heads/<key>` | ls-remote | 關住：`has_remote && auto_push && remote_feature_tip_after`（Fact） |
| `agf cleanup` | 2364 | `push <url> :refs/heads/<key> --force-with-lease=...` | push（刪遠端分支） | 關住：同上（Fact） |
| `agf ditch` | 2433（`deletion_remote`） | `remote get-url` ×2 | 本機讀設定 | 關住：off 時給 `{ url: '' }`（Fact） |
| `agf ditch` | 2435、2450（`discard_snapshot` → 2093 `server_tip`） | `ls-remote` | ls-remote | 關住：`url` 是空的（Fact） |
| `agf ditch` | 2465（`server_tip`） | `ls-remote` | ls-remote | 關住（Fact） |
| `agf ditch` | 2470 | `push <url> :refs/heads/<key> --force-with-lease=...` | push（刪遠端分支） | 關住：`has_remote_branch && auto_push`（Fact） |
| `agf close`（push 模式） | 1294–1352（`close_push`），呼叫點 1386（重試已存的 commit）、1572 | `fetch <remote>`（1305）、`push <remote> <sha>:refs/heads/<branch>`（1332）、`ls-remote <remote> refs/heads/<branch>`（1338） | fetch + push + ls-remote | 關住：在 `close_main` 第 2614 行之後、`push_not_authorized` 檢查（2620）之前加一道關卡，off 時回傳錯誤碼 `stream_auto_push_disabled`。stream 用 `stream_auto_push`，主工作區用 `main_auto_push`（Fact） |

**不連網、不需要關的相關呼叫（Fact，已讀程式確認）：**

- `agf start` / 啟動時的 stream 判斷：`start_main`（879 起）與 `resume-intake.js` 只用 `rev-parse`、`status`、`show`、`diff`、`branch --show-current`，沒有 fetch、ls-remote 或 push。

- `default-branch.js:17` 讀 `symbolic-ref refs/remotes/origin/HEAD`，`agf.js:608` `branch_names` 讀 `refs/remotes`。兩個都只讀本機快照。

- `completion-context.js:216–235` `gather_push` 用的是 `git remote`、`rev-parse @{u}`、`rev-list @{u}..HEAD --count`，都在本機。

- `stop-hook.js`：沒有任何 remote 或 push 相關的 Git 呼叫（用 grep 確認）。

- `install-hook.js:250–295` `apply_guard` 只安裝 **pre-commit** 的 devlog guard，不安裝 pre-push，也不連網。

- `patch/pre-push` 與 `patch/clean-branch.sh`（commit `cbaabab`）：這是 owner 自己手動使用的工具，`agf` 不會呼叫它們。`pre-push` 只在有人執行 `git push` 時才被觸發；`clean-branch.sh:52` 的 fetch 和 `:116` 的 push 都是使用者自己下指令才會跑。

- `external-runner.js:108/116` 的 `git clone --no-local <本機路徑>` 加上第 44 行的 `remote remove`：這是複製本機資料夾給 worker 用，不是連到遠端伺服器。

- `round-linter.js:2357` `lint_direct_route_completion`（第 2412–2425 行）：不執行 Git，但它要求「有遠端時必須提供成功的 `push_results`」。參考修補新增 `push_setting_disabled` 來免除這個要求（Fact）。`lint_push_claim`（1901）只有在文字中出現「已 push」之類的宣稱時才檢查，off 時不受影響（Fact）。

---

## 2. 參考修補沒有關住的缺口

1. **agent 自己手動 push**

   - 觸發：`SKILL.md:169` 要求 agent「有遠端就 commit 後 push，第一次 push 前先 fetch」；`references/streams.md:42/50` 要求 stream 的收尾提交要 push。

   - 現況：參考修補只改了文字，在 169 行加上「`stream-auto-push: off` 時例外」，streams.md 也改成「只有 on 才 push」（Fact）。程式沒有任何阻擋，agent 照樣可以自己執行 `git push`（Inference）。

   - 影響：off 是否真的「完全本機」，取決於 agent 有沒有照文字做。

2. **round linter 不會抓到「off 卻 push 了」**

   - 證據：參考修補 `round-linter.js` 在 `push_disabled` 為 true 時，如果 `push_results` 是一個含成功結果的陣列，仍然判定通過（看修補後的條件：只免除「空陣列」，沒有拒絕「非空」）（Fact）。

   - 影響：off 時發生的 push 不會被發現（Inference）。要不要加這個檢查屬於新行為，建議先當成 proposal 處理。

3. **off 時錯誤訊息仍建議手動 fetch**

   - 證據：參考修補 cleanup 在找不到本機 feature 分支時，若有 origin 會印出 `git fetch origin <key>:<key>`（修補後 `clean_main` 新增的區塊，Fact）。

   - 影響：程式本身沒有連網，只是建議 owner 手動執行。是否符合「完全本機」需要 owner 決定（Inference）。

4. **知識文件沒更新**

   - 證據：參考修補只改了 `docs/agent/FEATURES.md`；`docs/agent/FLOWS.md:161–162`（「commit; push if remote」、「finish --prep … push」）和 `docs/agent/ARCHITECTURE.md:34` 仍然描述成無條件 push（Fact）。

   - 依據：`AGENTS.md` 要求主要流程改變時更新 `docs/agent/`。

5. **既有、跟這次無關的不一致（只記錄，不要修）**

   - `new_main` 第 2005 行 `has_remote = remote_listing.out !== ''` 只要有「任何」遠端就成立，但 push 的對象寫死是 `origin`（Fact）。off 時這段直接跳過，所以不影響本功能。依照 scope discipline 不修。

**已確認沒有缺口的地方（Fact）：**

- `close_push` 的兩個呼叫點（1386、1572）都在 `close_execute` 裡面，而 `close_execute` 只會在 `close_main` 第 2647 行、也就是新關卡之後被呼叫，所以都被涵蓋。

- `main_recovery`（1123–1124）印出的 `git fetch origin` 只出現在 on 的遠端路徑。

---

## 3. 參考修補逐 hunk 套用到 HEAD 的情況

**檢查方式（Fact）：** 執行 `git diff b208461 e7b41ef -- skills docs | git apply --check -v`（只檢查、不寫入），結果 13 個檔案全部通過，沒有任何 hunk 失敗。

**`agf.js`（26 個 hunk）：**

- 第 1 個 hunk（`USAGE_COMMANDS`，HEAD 第 44 行 finish prep 的說明文字）：位置沒變，直接套用。

- 第 2–26 個 hunk：全部往後位移 **24 行**，沒有文字衝突。HEAD 的實際位置就是「參考 base 行號 + 24」，例如：

  - `stream_doc` 之後插入 `stream_auto_push`、`main_auto_push`、`stream_push_setting`：base 1021 → HEAD 1045 附近。

  - `finish_context` 的 return：base 1081 → HEAD 1105–1108。

  - `closing_record`：HEAD 1714；finish prep：HEAD 1733–1755；deliver：HEAD 1763、1781、1855、1912。

  - `new_main` push：HEAD 2006–2014。

  - `deletion_remote` 註解：HEAD 2057；`last_fetched_tip` 插在 `server_tip` 之後：HEAD 2076 之後。

  - `clean_main`：HEAD 2167–2169（讀設定）、2191（`destination`）、2197–2210（缺分支時拒絕、快取警告）、2306、2323、2357、2374。

  - `ditch_main`：HEAD 2428–2433、2444、2469、2481。

  - `close_main` 關卡：HEAD 2614 之後。

- **為什麼位移 24 行（Fact）：** drift 只在這些 hunk 前面加了程式：`init_main` 驗證 notebook（第 221–238 行，+14 行），`start_result`、`start_public_result`、`emit_start_result`、`start_main` 加上 `session_id` 與手動擷取提示（第 769–1005 行，+10 行）。這些區域都不跟修補的 hunk 重疊。

**`ag-settings.js`（9 個 hunk）：** 全部原位套用（Fact）。

- drift 唯一的改動是 HEAD 第 170 行：codex 樣板的 `'away-gates'` 從 `'off'` 改成 `'on'`。修補的樣板 hunk 從第 179 行開始，沒碰到這行。

- **語意上的衝突（Fact）：** claude 樣板（HEAD 第 220 行）仍然是 `'away-gates': 'off'`，兩個樣板現在不一致。這是 drift 造成的，不屬於本功能範圍。

**`round-linter.js`（2 個 hunk）：** 原位套用；HEAD 這個檔案跟 base 一模一樣（Fact）。

**測試檔：**

- `agf.test.js` 3 個 hunk 都位移 78 行後套用（Fact）。

- `ag-settings.test.js`、`round-linter.test.js` 原位套用（Fact）。

- 參考修補在 `ag-settings.test.js` 第 1 行後多加了一個空行（Fact）。這是無意義的格式變動，依 scope discipline 不要移植。

**文件與 prompt：**

- `SKILL.md`、`references/closeout.md`、`references/streams.md`、`scripts/README.md`：原位套用。

- `docs/agent/FEATURES.md`：位移 1 行；`AG_GUIDE.md` 和 `AG_GUIDE.zh-tw.md`：位移 2 行（Fact）。

- **語意衝突（Fact）：** drift 改了 `SKILL.md` 的版本號（8.4.14→8.4.15）和第 68 行的 Claude 擷取規則，但跟修補要改的第 169、177 行沒有重疊。

**改名時要一起換的識別字（Inference，依修補內容整理）：**

| 參考修補 | 建議改成（host 決定） |
| --- | --- |
| 設定鍵 `stream-auto-push` | `remote-sync` |
| `ag-settings.js:stream_auto_push_from_text` | 例如 `remote_sync_from_text` |
| `agf.js:stream_auto_push`、`main_auto_push`、`stream_push_setting`、`context.auto_push`、`push_setting.auto_push` | 例如 `stream_remote_sync`、`main_remote_sync`… |
| 錯誤碼 `stream_auto_push_disabled` | 例如 `remote_sync_disabled` |
| `round-linter.js:push_setting_disabled` | 可以保留原名，或改成 `remote_sync_disabled` |
| 訊息 `stream-auto-push is off — …` | `remote-sync is off — …` |
| `last_fetched_tip` | 名稱跟設定無關，可以保留 |

---

## 4. 設定管線（`ag-settings.js`，HEAD 行號）

- `switch_names`（第 32 行）：決定 `settings` 顯示時的順序（`format_settings_display` 在第 2018 行用它）。參考修補把新鍵放在 `streams` 後面（Fact）。

- `optional_switch_names`（第 33 行）：放在這裡的鍵可以不出現在 `ag.json` 裡；舊專案不用遷移也能通過驗證（`validate_switches` 第 660–661 行）。參考修補有加進來（Fact）。

- `changeable_switch_names`（第 54 行）由 `switch_names` 推導出來，所以 `remote-sync: off` 這種修改語法會自動支援。`apply_changes` 在第 1587 行允許「還不存在但屬於 optional」的鍵（Fact）。

- `host_template_values`：codex 樣板 162–188 行（`streams` 在第 181 行），claude 樣板的 `streams` 在第 231 行。參考修補在兩個樣板都加了 `'stream-auto-push': 'on'`（Fact）。

  - **副作用（Fact）：** `audit_template`（第 720–735 行）會在 `agf start` 時把樣板有、但專案 `ag.json` 缺少的鍵補進去，並在 `config_audit.added` 回報。加進樣板就代表既有專案下次啟動時會自動被寫入 `remote-sync: on`。

- `validate_switches` 的 `legal_switches`（第 667–679 行，`streams` 在第 674 行）：參考修補加了 `['on', 'off']`（Fact）。

- `canonical_config`（第 1034–1052 行）：鍵只在存在時才寫出，最後用 `localeCompare('en')` 排序（Fact）。

  - 用 node 實測排序結果是 `notebook-ownership`、`remote-sync`、`review-policy`、`stream-auto-push`、`streams`（Fact）。

  - `remote-sync` 不管用 localeCompare 還是預設排序，位置都一樣（Fact）。所以 `agf.test.js:425` 的斷言「`Object.keys(saved.switches)` 已排序」不會出問題。

- `switch_display_value`（第 2001–2005 行）：鍵不存在時要回傳預設值。參考修補加了 `key === 'stream-auto-push' ? 'on'`（Fact）。

- `format_settings_display`（第 2007 行起；`'- streams: …'` 在第 2041 行）：參考修補在後面加了一行合法值說明（Fact）。

- `module.exports`（第 2142 行起）：要匯出新的 `*_from_text` 讀取函式（Fact）。

- `copy_for_notebook`（第 1207 行）：會完整複製 root 設定，所以 `agf new` 建立的 stream 會繼承這個鍵（Fact）。

**會受新鍵影響、而且斷言寫死的測試：**

- `ag-settings.test.js:171`：樣板鍵清單是精確比對。依樣板字面順序，`remote-sync` 要插在 `'notebook-ownership'` 和 `'review-policy'` 中間（Inference：取決於 host 把它放在樣板的哪個位置；參考修補是放在 `streams` 後面）。

- `ag-settings.test.js:563–568`：每個非 optional 的鍵被刪掉時都必須驗證失敗。所以新鍵**一定要**放進 `optional_switch_names`，否則這個測試會要求它變成必填（Fact）。

- `ag-settings.test.js:1210`：比對 `streams` 那一行說明文字；新增一行不會影響它（Fact）。

- `prompt-compression.test.js:26–34`：要求 `SKILL.md` 小於 32 KiB。HEAD 已經是 36895 bytes，**目前就失敗**（Fact，已實際執行）。參考修補會讓它再增加約 215 bytes（參考 base 36473 → e7b41ef 36688，Fact）。

- `agf.test.js:409–448`：精確比對 `config_audit.added`，只刪掉 `away-gates`。只要測試 fixture 是從樣板產生的，新鍵就不會出現在 `added` 裡（Inference）。

---

## 5. 慣例、建議修改位置、最小測試指令、未檢查範圍

**慣例（Fact）：**

- 程式註解和 commit 訊息用英文；commit 格式像 `feat(agentflow): …`。

- `agf.js` 用 tab 縮排、沒有分號；`round-linter.js` 用 2 空格、有分號。

- 驗證錯誤統一格式是 `configuration.switches.<key> must be one of …`。

- `SKILL.md` 和 `references/*.md` 是給其他 agent 讀的產品 prompt，測試會比對字句。

- 發布時版本號要在 `SKILL.md`、`.claude-plugin/plugin.json`、`README*.md`、`CHANGELOG.md` 一致；但這次任務不是發布（Inference）。

**建議修改位置（Inference）：**

1. `skills/agentflow/scripts/ag-settings.js`：上面第 4 節列的 9 個位置。

2. `skills/agentflow/scripts/agf.js`：上面第 3 節列的 26 個 hunk，改名後套用。

3. `skills/agentflow/scripts/round-linter.js`：在第 2354 行之後插入讀設定的函式，並修改第 2416–2420 行。

4. 測試：`ag-settings.test.js`、`agf.test.js`（9 個新案例加上 `close_fixture` 參數）、`round-linter.test.js`（1 個新案例）。

5. Prompt 與文件：`SKILL.md:169/177`、`references/closeout.md:47`、`references/streams.md:21/41–43/50/54 後/65 後`、`scripts/README.md:157`、`docs/AG_GUIDE*.md`、`docs/agent/FEATURES.md` 第 11 節。另外建議更新 `docs/agent/FLOWS.md:161–162`（這是缺口 4）。

**最小測試指令：**

- 這個環境有兩個陷阱，跑測試前一定要處理（Fact，已重現）：

  - 這台電腦的全域 `core.hooksPath=C:/Users/user/.git-hooks` 裡有 `pre-push`，會擋掉**所有** push（訊息是 `[LOCAL BLOCK] 偵測到 Push 指令`）。所以要用 `GIT_CONFIG_GLOBAL=<空檔案>` 隔離。

  - Claude Code 的環境變數（`CLAUDE_CODE_SESSION_ID`、`CLAUDE_CODE_SSE_PORT`、`CLAUDE_CODE_ENTRYPOINT`）會讓 `agf new` 判斷成「同時是兩種 host」，然後拒絕執行（`active host is ambiguous`）。所以要用 `env -u` 清掉。

- 指令範本：

  ```sh
  : > "$SCRATCH/empty.gitconfig"
  cd skills/agentflow/scripts && env -u CLAUDE_CODE_SESSION_ID -u CLAUDE_CODE_SSE_PORT -u CLAUDE_CODE_ENTRYPOINT GIT_CONFIG_GLOBAL="$SCRATCH/empty.gitconfig" node --test --test-name-pattern="remote-sync" agf.test.js
  cd skills/agentflow/scripts && node --test ag-settings.test.js
  cd skills/agentflow/scripts && node --test round-linter.test.js
  cd skills/agentflow/scripts && node --test prompt-compression.test.js
  ```

- 完整跑一次 `agf.test.js` 大約要 563 秒（Fact），接近工具的 600 秒上限。開發時建議用 `--test-name-pattern` 只跑相關案例，最後再跑完整套件。

**HEAD 目前已經失敗的測試（移植前的基準，Fact，已實際執行）：**

- `ag-settings.test.js`：87 個裡有 4 個失敗。

  - 3 個是 drift 把 `away-gates` 改成 `on` 造成的（第 166、187、228 行）。

  - 1 個是 Windows symlink 權限錯誤 `EPERM`（第 738 行）。

- `round-linter.test.js`：323 個裡有 4 個失敗。3 個是 symlink `EPERM`，另外 1 個是「treats a trusted cosmetic artifact defect…」，原因沒有細查。

- `agf.test.js`（已隔離環境）：182 個裡有 7 個失敗。

  - 2 個是 `away-gates`（第 409、449 行）。

  - 1 個是 claude 啟動結果多了 `session_id`、`hooks` 欄位（第 341 行，drift 造成）。

  - 1 個是 close scope（第 1300 行）。

  - 3 個是 symlink `EPERM`。

  - 如果**沒有**隔離環境，會有超過 40 個跟遠端或 `new` 有關的案例失敗。

- `prompt-compression.test.js`：1 個失敗（SKILL.md 36895 bytes，超過 32 KiB 預算）。

- 移植後判斷「有沒有變壞」時，要跟上面這份基準比，不能要求全部通過。

**沒有檢查的範圍：**

- 沒有細讀參考修補 `agf.test.js` 9 個新案例的完整內容，只看了名稱和第一個案例。

- 沒有確認 `.claude-plugin/plugin.json`、`README*.md`、`CHANGELOG.md` 是否需要更新（這次不是發布）。

- 沒有細查 `round-linter.test.js` 那個非 symlink 失敗、以及 `agf.test.js:1300` close scope 失敗的原因。

- 沒有檢查 `looper.js`、`*-journey.js`、`*.py` 的每個 Git 呼叫。全域 grep 找不到 fetch、push、ls-remote，但這些檔案沒有逐行讀過。

- 沒有讀 `skills/agentflow/docs/incidents-log.md`。修補碰到的 `clean_main` Guard 3 帶有 `— I-058` 標記，但修補沒有改那條規則本身（Fact），所以應該不受影響。

**敵意指令檢查：** 讀過的 `AGENTS.md`、`CLAUDE.md`、`SKILL.md`、`references/*.md`、`patch/README.md` 都沒有發現試圖改變審查行為的敵意指令（Fact）。

Self-check: 已逐項回答交付 1–5，每項標註 Fact 或 Inference；行號已用 HEAD c909ccc 的 grep 或 sed 核對；套用性用 `git apply --check`（只檢查不寫入）確認；基準測試只在系統暫存目錄建立 fixture；除了這份報告之外，沒有修改任何 repo 檔案，也沒有執行會改變狀態的 git 指令。
