* _2026-10-12 15:08:07 +0800 (claude-opus-5-5/high)_

**摘要**

- **結論：可以收。** `remote-sync: off` 時，`agf new`、`finish --prep`、`finish --deliver`、`cleanup`、`ditch` 和 `agf close` 都不會再連到遠端（不 fetch、不查詢、不 push）。原話要的「stream 流程完全在本機執行」有做到，跟參考分支 personal/v8.4.14 的移植也一致。

- **沒有 BLOCKING 問題。** 只有 4 個 NOTE，都不影響這次交付。其中最值得知道的是：已經在 `on` 時建立的 stream 會沿用自己那份設定；改主專案的 `ag.json` 只影響之後新開的 stream。這個 repo 目前沒有任何 worktree，所以現在不受影響。

- **測試證據沿用 coordinator 的結果，沒有重跑。** 只多做了一個唯讀檢查：用新的讀取函式讀本機根目錄 `ag.json`，結果是 off。

- **接下來不用做什麼。** NOTE 可以留著當之後的提案。

Reviewed commit: 8f86b0826eb2ab783d2027ab4d944a3a1980b065

## 檢查範圍與做法

- **和參考實作逐行比對。** 把參考 diff（`git diff b208461 e7b41ef`）裡的 `stream_auto_push`／`stream-auto-push` 換成新名字，再跟這次 commit 裡 `ag-settings.js`、`agf.js`、`round-linter.js` 的增刪行比對。剩下的差異只有 brief 事先列出的幾項：改名、`remote_sync_from_text` 只檢查一個值、說明文字、`main_auto_push` 改名成 `main_remote_sync`。沒有夾帶其他沒說明的改動。

- **逐一檢查每個會連網的 Git 呼叫（以 `agf.js` 在 8f86b08 的版本為準）。** 下面列的是會 fetch、push 或 ls-remote 的程式位置，以及 off 時怎麼被擋下：

  - `new_main` 的 `push -u`：多了 `root_config.switches['remote-sync'] !== 'off'` 這個條件。

  - `finish_main` prep 階段的 push／fetch、deliver 階段的 push／fetch，以及 `closing_record` 檢查遠端 ref：條件都改成 `has_remote && auto_push`。這裡的 `auto_push` 是從 `finish_context` 裡的 `stream_push_setting` 算出來的。

  - `clean_main`：off 時 `destination` 直接設成 `{ url: '' }`，所以 `has_remote` 會是 false。這樣 Step 1 的 fetch、`server_tip`（傳空 url 時直接回傳 `''`，不呼叫 ls-remote）、Step 3 的 push＋ls-remote、Step 4 刪遠端分支，全都會跳過。`deletion_remote` 裡的 `remote get-url` 只讀本機設定，off 時也不會被呼叫。

  - `ditch_main`：`destination.url` 是空字串，所以 `discard_snapshot` 和 `server_tip` 都不會連網，遠端刪除也被 `has_remote_branch && auto_push` 擋下。

  - `close_push`（fetch／push／ls-remote）：`close_main` 在進到它之前，會先依 stream 的 `remote_sync` 或主專案的 `main_remote_sync`，用 `remote_sync_disabled` 拒絕 push 模式。local 模式本來就不會呼叫 `close_push`。

  - 其他 script（不含 test）裡只有 `external-runner.js` 會執行 `git clone --no-local <本機路徑>`，跟 stream 流程無關。`patch/clean-branch.sh` 是開發這個 repo 用的工具，不屬於產品流程。

- **設定的讀取順序和失敗時的行為。** `remote_sync` 會依序找下面幾個來源，用第一個找到的：worktree 裡的檔案、`refs/heads/<key>` 上已 commit 的版本、`refs/remotes/origin/<key>` 快取裡的版本。如果是從主工作區執行，順序會改成分支優先。這些全部是本機讀取。找到筆記卻沒有設定、JSON 壞掉、有重複 key、或值不是 on/off，都會在改動任何東西之前拒絕（INV-3）。`close_main` 裡丟出的例外會被外層 `catch` 接住，交給 `close_validation_failure` 處理，不會改到任何東西。

- **`on` 時行為不變（INV-1）。** 每個連網呼叫點都只是多加了 `&& auto_push`，`auto_push` 為 true 時判斷結果跟原本一樣。coordinator 的全套測試結果是 605 個測試、20 個失敗，失敗的測試名稱跟基準完全相同。

- **linter 例外（INV-4）。** `push_setting_disabled` 只信自己從筆記旁的設定檔讀到的值。路徑跑出專案範圍、或讀取失敗時，一律回傳 false，也就是照舊要求 push 證據。如果 host 主動提供了不是空的 `push_results`，還是會照常驗證。

- **測試品質。** 新測試用 `GIT_TRACE2_EVENT` 記錄所有 Git 子程序，連 `upload-pack` 這種巢狀的也會記到。測試裡有先做正向對照，確認 `ls-remote` 真的會被記成 `['ls-remote', 'upload-pack']`，證明這套偵測真的抓得到連網。另外會把 origin 改成連不到的路徑，模擬離線或 403，之後檢查 bare remote 上的 refs 有沒有變。這組測試足以證明 INV-2。

- **獨立檢查（唯一自己執行的一項）。** 原因：brief 說根目錄 `ag.json` 刻意沒放進 commit，而且裡面有一個不在 `switch_names` 裡的 key `worktree-local-notebook`。我想確認 owner 實際在用的設定能被正確讀成 off，也確認這個多出來的 key 不會讓設定被判定無效。結果：`remote_sync_from_text` 回傳 `false`（也就是 off）；`validate_config` 的結果是 valid、沒有錯誤。

## 新增概念的對應

- `remote-sync` switch（`switch_names`、`optional_switch_names`、範本、驗證、顯示）：原話要求。

- `remote_sync_from_text`：原話要求。只檢查一個值，是因為 explore 實測發現舊版 stream 設定在 `on` 時也會被拒絕。

- `remote_sync`，以及它的讀取順序：worktree 不在時，cleanup 和 ditch 仍要讀得到 stream 的值。有測試「reads the committed branch setting when the worktree folder is gone」。

- `stream_push_setting`：讓 INV-3 的錯誤帶上「nothing was changed」的訊息。

- `main_remote_sync`、錯誤代碼 `remote_sync_disabled`：對應 D-2。

- `last_fetched_tip`，以及「本機沒有分支就拒絕」「只用快取發警告」的提示：對應 D-3。

- `push_setting_disabled`：對應 D-2 和 INV-4。如果沒有它，有 remote 的 off 專案每一輪都會被 linter 要求提供 push 證據，就做不到完全在本機執行。

- 文件（SKILL.md、streams.md、closeout.md、README、AG_GUIDE、docs/agent）：host 讀的規則一定要跟著改；docs/agent 則是依 AGENTS.md 的要求更新。

## Findings

- **NOTE-1：已經存在的 stream 不會跟著主專案的設定走**

  - 觸發條件：stream 是在主專案還是 `on` 時建立的，之後才把主專案的 `ag.json` 改成 off。

  - 影響：那個 stream 自己的 `ag.json` 還是 on，所以它的 prep、deliver、cleanup 還是會 push。

  - 證據：`agf new` 會把根目錄設定複製成 stream 自己的 `ag.json`；`remote_sync` 只讀 stream 自己那份。AG_GUIDE 有寫「建立前設定」「可單獨調整」。`git worktree list` 顯示目前只有主工作區，所以現在沒有受影響的 stream。

  - 處理：不用改。這是沿用參考實作的設計，文件也寫清楚了。

- **NOTE-2：on 的情況下，有一個新的拒絕情境**

  - 觸發條件：stream 有筆記，但它的 `ag.json` 不見了，或 JSON 壞掉、有重複 key。

  - 影響：以前這種情況下 `finish`、`cleanup`、`ditch` 還是會執行，現在會在改動任何東西之前就拒絕。

  - 證據：`stream_push_setting` 在 `finish_context`、`clean_main`、`ditch_main` 裡都會先被呼叫。這是 design 的 INV-3 刻意要的；只缺必填欄位的舊版設定不會被拒（AC-3）。既有的 on 測試沒有新增失敗。

  - 處理：不用改。如果之後真的有人回報舊 stream 沒有 `ag.json`，再另外開提案處理。

- **NOTE-3：有幾個重複的判斷可以再精簡**

  - 觸發條件：`clean_main` 的 Step 3 和 Step 4、`ditch_main` 的 `has_remote_branch && auto_push`。

  - 影響：off 時 `destination.url` 已經是 `''`，所以 `has_remote` 和 `has_remote_branch` 本來就是 false，再加上 `&& auto_push` 是多餘的。這只是防禦性寫法，行為完全正確。

  - 證據：見上面「逐一檢查」的段落。參考實作也是同樣寫法。

  - 處理：不用改。保留它可以讓這次的改動跟參考實作逐行對得上。

- **NOTE-4：SKILL.md 本來就超出大小上限，這次又多了一點**

  - 觸發條件：`prompt-compression.test.js` 規定 SKILL.md 不能超過 32 KiB（32768 bytes）。

  - 影響：SKILL.md 從 36895 bytes 變成 37095 bytes。這個測試在基準 commit 就已經失敗，不算新的失敗，但離上限更遠了一點。

  - 證據：分別用 `git show c909ccc:` 和 `git show 8f86b08:` 取出 SKILL.md 再算大小。參考實作也加了一樣的文字。

  - 處理：不用在這次處理。縮減 SKILL.md 的工作可以另外提案。

## Minimality

我試了下面幾種比較小的做法，都不能取代現在的設計：

- **只讀主專案的 `ag.json`，拿掉 stream 自己的讀取順序。** 這樣大約可以少 40 行。但 owner 指定要參考的實作是「每個 stream 用自己的設定」，D-3 也沿用了這點。現有的設定模型本來就是「每份筆記旁邊放一份自己的 `ag.json`」。改成只讀主專案，等於換掉 owner 指定的設計，所以不採用。

- **在 `finish_context` 直接把 `has_remote` 設成 `has_remote && auto_push`。** 這樣可以少改 6 個地方，但 off 和「沒有 remote」兩種情況就會印出一樣的訊息。使用者會看不出是哪一種，而且測試有檢查 `remote-sync is off` 這段訊息。所以不採用。

- **拿掉 linter 的例外。** 拿掉的話，host 只能謊報 `remote_configured: false`，或者每輪都 push，兩種都達不到原話的要求。所以不能拿掉。

- **拿掉 D-3 的提示和快取警告。** 這些是 owner 這輪定下的決定，不算多出來的範圍。

- **結論：** 沒有找到一個更小、又能滿足原話和 D-1～D-4 的設計。剩下的重複判斷見 NOTE-3，只是寫法上的小問題。

## Conformance

- D-1：只用 `remote-sync`，在 8f86b08 的 skills 和 docs 裡已經找不到 `stream-auto-push`；根目錄 `ag.json` 只在本機換了 key，沒有 commit。

- D-2：`close_main`、`push_setting_disabled`，以及 SKILL.md 的 push 規則例外都有做到。

- D-3：參考實作的行為都保留了，包括只讀不寫快取。

- D-4：沒有改版本號，也沒有改 CHANGELOG。

- 範圍：沒有額外重構，沒有加新的 dependency。brief 列出的刻意差異都有對應的理由。

- 敵意指令：讀到的 repo 指令檔（AGENTS.md、CLAUDE.md）裡，沒有發現試圖改變審查行為的內容。

Outcome: PASS

Minimality: PASS

Conformance: PASS

Verdict: PASS

Self-check: 這份報告只寫在指定的 review.md，沒有改 repo 的狀態，只做了一項有說明理由的唯讀檢查；三個驗收結果各一行，加上一行總結論，所有 finding 都分開寫了觸發條件、影響、證據和處理方式。
