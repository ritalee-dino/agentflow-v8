* _2026-10-12 14:37:25 +0800 (claude-opus-5-5/high)_

# A-004 explore：把 `remote-sync`（off = stream 全程本機）移植到本分支的風險

**TL;DR**

- **事實：參考實作的 CLI 在 off 時真的完全不連網。** 我把 `e7b41ef` 的程式匯出到 scratch，實際跑了 `agf new` → `finish --prep` → `finish --deliver` → `cleanup`，另外再開一個 stream 跑 `ditch`，同時用 `GIT_TRACE2_EVENT` 記下每一個 Git 子程序。off 時連網程序是 0 個，bare remote 的 refs 也完全沒變；on 的對照組每一步都看得到 push、fetch、ls-remote。

- **事實：真正剩下的連網來源不在 CLI，而在 host 自己。** `SKILL.md:169` 叫 host「有 remote 就 push，第一次 push 前先 fetch」；`round-linter.js:2416-2420` 在有 remote 時要求 push 證據。參考實作是靠 prompt 加一句例外、linter 加豁免來處理。移植時這兩處一定要一起帶，不然 stream 裡每一輪收尾還是會 push。

- **風險（事實＋推論）：** 參考版的設定讀取函式會用整份 `ag.json` 做嚴格驗證。所以就算設定是 on 或沒寫，只要 stream 的 `ag.json` 有不相干的錯（例如 schema 是舊版），`finish`、`cleanup`、`ditch` 都會拒絕執行，這是 HEAD 目前不會發生的新失敗。另外，repo 根目錄的 `ag.json:26` 已經寫著 `"stream-auto-push": "off"`。改名成 `remote-sync` 之後，這個舊鍵會被默默忽略，實際效果變成 on。

- **要請 owner 決定兩件事：** 第一，主工作區的 closeout 要不要也歸 `remote-sync` 管？這次的需求字面上只說「stream 流程」，參考分支的 A-005 ans 1 則說要一起停。第二，舊名稱 `stream-auto-push` 要不要相容，還是只在文件裡註明？

---

## 1. off 時，參考實作還有沒有任何路徑會連網？

**結論（事實）：** CLI 本身完全沒有。剩下的連網都是 host 依照 prompt 自己去做的，參考實作用 prompt 和 linter 擋住。

**實測（事實）**

- 方法：scratch 腳本 `exp2/probe.js` 載入 `e7b41ef` 的 `agf.js`，建一個本機 bare remote，跑完整個生命週期。每一步都用 `GIT_TRACE2_EVENT` 記錄 `cmd_name`，然後篩出 `fetch`、`push`、`ls-remote`、`upload-pack`、`receive-pack`、`remote-*`。

- 結果：

  | 步驟 | off：連網程序 | on：連網程序（對照組） |
  | --- | --- | --- |
  | `agf new` | 無 | push |
  | `finish --prep` | 無 | push、fetch |
  | `finish --deliver` | 無 | push、fetch |
  | `cleanup` | 無 | fetch、ls-remote ×4、push ×2 |
  | `agf new`（第二個） | 無 | push |
  | `ditch` | 無 | ls-remote ×3、push |
  | bare remote refs 有沒有變 | 沒變 | 有變 |

- 限制：`agf close` 沒有跑實測，只讀了程式碼（見下表）。

**逐條路徑對照**（「參考」是 `git show e7b41ef:skills/agentflow/scripts/agf.js` 的行號，「HEAD」是目前分支的行號）

- **`agf new`：建分支前不會 fetch。**

  - HEAD：`agf.js:1963` 從本機 HEAD 執行 `worktree add -b`，沒有 fetch。只有 `agf.js:2005-2007` 會在有任何 remote 時 `push -u`。

  - 參考：`agf.js:2037` 加上條件「主專案設定不是 off」。這裡讀的是主專案的 `root_config`，不是 stream 自己的設定。

- **`finish --prep`：off 時不會 fetch `origin/<default>`。**

  - HEAD：`agf.js:1733-1742` 先 push 再 fetch，然後合併 `origin/<def>`。

  - 參考：`agf.js:1764` 改成要 `has_remote && auto_push` 才走這段；off 時改走 `else` 分支，合併本機的 `<def>`。

- **`finish --deliver`：off 時不 fetch，也不檢查遠端分支。**

  - HEAD：`closing_record` 在 `agf.js:1714` 要求 `origin/<key>` 等於 HEAD。這一步只讀本機 ref，但等於逼你先 push。`agf.js:1781-1798` 會 push 到遠端預設分支再 fetch。

  - 參考：`agf.js:1745`、`:1794`、`:1812`、`:1943` 都改成 `has_remote && auto_push`。

- **`cleanup` Step 1「讓主分支跟上伺服器」：off 時整段跳過。**

  - HEAD：`agf.js:2191` 呼叫 `deletion_remote`；`agf.js:2213` fetch；`agf.js:2237` 用 `server_tip` 跑 ls-remote；`agf.js:2307`、`:2314` push 後再用 ls-remote 驗證；`agf.js:2328` sweep 前再查一次；`agf.js:2358`、`:2364` 刪除遠端分支。

  - 參考：`agf.js:2233` off 時把 `destination` 設成 `{ url: '' }`，所以 `has_remote` 是 false，Step 1（`:2271`）、Step 3（`:2366`）、刪遠端（`:2417`）都不會跑。

- **`ditch` 裡的 `discard_snapshot`：網址是空字串時不會跑 ls-remote。**

  - `server_tip` 一開頭就是 `if (!url) return ''`（參考 `agf.js:2102`，HEAD `agf.js:2070`）。所以 off 時傳進空網址，就不會呼叫 ls-remote。

  - 參考 `agf.js:2497` 傳空網址；`:2533` 刪除遠端前要 `auto_push`。確認前後兩次 snapshot 和刪除前的檢查（`:2529`）都只是比對 `'' === ''`。

  - HEAD：`agf.js:2433` 拿真正的網址，`:2435`、`:2450`、`:2465` 共跑 3 次 ls-remote，`:2470` 刪除遠端。

- **在 stream worktree 裡跑 `agf close`：**

  - local 模式：不管 HEAD 還是參考，都不會連網。只有 `close_push`（HEAD `agf.js:1305`、`:1332`、`:1338`）會 fetch、push、ls-remote，而它只在 push 模式才會被呼叫。

  - push 模式：參考 `agf.js:2686` 在 `close_execute` 之前就拒絕。所以「重送已存好的 commit」那條路（參考 `agf.js:1411-1417`）也一樣被擋住。

- **stop-hook／round-linter：本身不連網，但會逼 host 去 push。**

  - `completion-context.js:216-235` 的 `gather_push` 只跑 `git remote`、`rev-parse @{u}`、`rev-list`，全部是本機操作。`stop-hook.js` 沒有任何 push 或 remote 相關的程式碼。

  - HEAD `round-linter.js:2413-2420`：只要有設定 remote，就要求 `push_results` 不能是空的。這等於逼 host 在 stream 裡 push。參考版用 `push_setting_disabled`（參考 `round-linter.js:2360`、`:2435`）讀 notebook 旁的設定來豁免。

  - `lint_push_claim`（HEAD `round-linter.js:1901-1917`）只在 notebook 裡聲稱「已 push」時才檢查，off 時不受影響。

- **`agf start` 的 stream 判斷：不連網。**

  - `start_main`、`resume-intake.js` 都找不到 fetch、ls-remote 或 push。`default-branch.js:17` 讀的是本機的 `refs/remotes/origin/HEAD`。

- **pre-push guard（`patch/pre-push`）：不會連網。**

  - 它只在真的 push 時才會執行，用的 `rev-list --remotes`、`ls-tree` 都是本機操作。off 時 agf 不會 push，所以它根本不會被觸發。

  - 小落差（事實）：`patch/README.md` 的注意事項第一條說「`agf new` 在有 remote 時會自動 push」，移植後這句就不完全對了。這在本次範圍之外，只列為提案。

- **host 依 prompt 做的連網（推論，影響最大）：**

  - HEAD `SKILL.md:169`：「commit each meaningful unit and push when a remote exists. Before the first pushed commit, fetch…」。在 stream worktree 裡，host 每一輪都會照這句去 push 和 fetch。

  - `references/streams.md:21`、`:41-43`、`:50` 也寫著「push when a remote exists」，`:43` 還說 deliver 會 fetch。

  - 參考實作改了 `SKILL.md:169`（加上 off 的例外）、`:177`（設定說明）、`streams.md`、`closeout.md:47`。這部分要一起移植，否則 CLI 雖然不連網，host 還是會連。

## 2. 參考分支記錄的已知缺陷與限制

**移植時要修或要避開的**

- **整份設定嚴格驗證，會擋住 on 模式（事實，這次新發現）。**

  - 觸發條件：stream 的 `ag.json` schema 是舊版，或缺少某個必填欄位。

  - 影響：參考版的 `finish_context`（`agf.js:1137`）、`clean_main`（`:2208`）、`ditch_main`（`:2493`）都會先讀這個設定，讀不到就拒絕執行。設定是 on 或沒寫也一樣。HEAD 現在完全不讀這個設定，所以這是新增的失敗。

  - 證據：我在 scratch 呼叫 `stream_auto_push_from_text`。schema 7 會丟出 `schema-version must be integer 8`，少了 `ask-names` 也會丟錯。

  - 建議：只解析 JSON 並檢查 `remote-sync` 這一個值，不要對整份設定做驗證。

- **改名後，舊鍵會被默默忽略（事實）。**

  - repo 根目錄的 `ag.json:26` 已經是 `"stream-auto-push": "off"`。驗證時遇到不認得的鍵不會報錯，所以改名後這個舊鍵沒有任何效果，實際行為是 on。

  - `canonical_config` 只保留它認得的鍵。所以移植時一定要把 `remote-sync` 加進 `canonical_config`（參考 `ag-settings.js:1059`），否則 `agf start` 重寫設定時會把它刪掉（這點是推論）。

  - 建議：請 owner 決定要不要相容舊名稱。不相容的話，至少要在文件裡註明，並請 owner 自己把根目錄的 `ag.json` 改成新名稱。

- **`SKILL.md` 大小（事實）：** HEAD 已經是 36895 bytes，超過 `prompt-compression.test.js:29` 的 32 KiB 上限，這個測試在改動前就失敗了。參考版在這裡多加了大約 215 bytes。建議移植時盡量讓位元組數不增加。

- **A-006 F-3（事實，低）：** 參考 `agf.js:2244` 沒有檢查 `git remote` 有沒有成功（`.ok`）。萬一這個指令失敗，會誤報「no origin is configured」。修法很小，可以順手修。

**目前可以接受、記錄下來即可的**

- **A-005 F-7：** 主工作區的 `ag.json` 格式錯誤時，push closeout 會提早失敗。這是偏安全的失敗方式。

- **A-005 F-10、A-006 F-4：** 「origin 有兩個網址也會跳過檢查」和「有 origin 但沒有快取時的提示」這兩個情況沒有測試。

- **A-006 F-5：** `docs/agent/FEATURES.md` 有一句說明不完整。

- **A-004 explore 列出的行為取捨**（off 時刻意不處理）：

  - 隊友推到遠端功能分支的 commit 不會被發現。

  - 本機主分支和遠端會越差越多。

  - 用來提醒的 `refs/remotes/origin/*` 快取可能過期，提醒可能漏報或誤報。

  - 這三種都不會讓資料消失，只會讓本機和遠端不同步。

- **參考分支 devlog STATUS 的 Open 項目：** 「prompt hook still captures non-owner messages」「agf close commit conflicts with the I-039 guard」。這兩件和移植無關，但測試環境會碰到 I-039 的全域 pre-commit guard（見第 4 節）。

## 3. 設定來源的優先順序：是不是比「stream 流程」需要的還多？

**參考版的 `stream_auto_push(dir, key)`（`agf.js:1029-1064`）怎麼找設定（事實）**

- 在 stream worktree 裡：依序讀 worktree 裡的 `features/<key>/ag.json` → 本機分支 `refs/heads/<key>` 上已 commit 的那份 → 快取的 `refs/remotes/origin/<key>` 上的那份。

- 在主工作區、而且 worktree 已經不在時：依序讀分支上已 commit 的那份 → 主工作區裡已交付的那份 → 快取的遠端分支那份。

- 找到 notebook 卻沒有對應的 `ag.json`，就拒絕執行。三個地方都沒有這個 stream 的紀錄時，當作 on。

- 另外兩個來源：`agf new` 讀主專案的 `root_config`（`:2037`），主工作區的 closeout 用 `main_auto_push` 讀 notebook 旁的設定（`:1067`）。

**風險**

- **事實：** 一個設定散在 4 種地方，分別是主專案、worktree 檔案、分支上的 commit、遠端快取。owner 在主專案改了設定，已經開好的 stream 不會跟著變，因為 stream 有自己的那一份。這是刻意的設計，但文件要寫清楚。

- **推論：** 從遠端快取讀設定，跟「完全只在本機」的直覺不太一致。它沒有連網，但讀的是上次 fetch 留下的遠端內容。

**可以簡化的地方（只是提案，不擴大範圍）**

- 拿掉 `refs/remotes/origin/<key>` 這個來源：只在本機分支被刪掉、但遠端還留著時才會用到，而這種情況 off 時本來就會拒絕 cleanup。

- 只驗證 `remote-sync` 這一個值，不驗證整份設定（理由見第 2 節）。

- 如果 owner 確認「只管 stream」：就不需要 `main_auto_push`，`close_main` 也只要在 stream worktree 裡檢查即可。linter 的豁免同樣只適用 stream notebook。

- 保留 worktree → 分支 commit 這兩層：worktree 已經刪掉時，cleanup 和 ditch 只能靠分支上那份設定，這是最少要有的。

## 4. 測試策略：怎麼在 Windows 的 node 測試裡便宜又可靠地證明「沒碰遠端」

**既有測試怎麼做（事實）**

- `agf.test.js:894` 的 `make_repo({ remote: true })` 在 `os.tmpdir()` 建一個 bare repo，然後 `push -u` 上去。

- 參考版的離線測試（參考 `agf.test.js:1824`、`:1864`）是把 origin 改成一個不存在的路徑，然後只檢查「指令成功」和「bare refs 沒變」。

  - 弱點：有些程式碼路徑遇到連網失敗也只會記一行 log 就繼續（例如 `agf new` 的 push 失敗）。這種「偷偷連了網但失敗了」的情況，這個方法抓不到。

- `make_git_wrapper`（`agf.test.js:1537`）是靠 shebang 執行的腳本，在 Windows 上會被跳過（`agf.test.js:19` 的 `unix_fixture_skip`），所以不能用。

**建議做法（已在這台機器上驗證可行）**

1. **主要判斷依據：`GIT_TRACE2_EVENT`。** 測試在呼叫 `agf.main` 前設定 `process.env.GIT_TRACE2_EVENT = <絕對路徑>`，跑完再刪掉這個變數，然後解析檔案裡的 `cmd_name`。

   - 斷言裡面沒有 `fetch`、`push`、`ls-remote`、`pull`、`upload-pack`、`receive-pack`、`remote-*`。

   - 驗證結果：Git 2.54.0.windows.1 加上 node `execFileSync`，用 Windows 的反斜線路徑也能正確寫入；連 `upload-pack` 這種子程序都抓得到。

2. **輔助判斷：** bare remote 的 `for-each-ref` 前後一樣，本機 `refs/remotes` 前後也一樣。

3. **選用：** 再加 `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.allow GIT_CONFIG_VALUE_0=never`，讓任何連線都直接失敗。我驗證過，本機路徑的 ls-remote、fetch、push 都會回 `transport 'file' not allowed`。有了 trace，這一步不是必要的。

4. **保留一個 on 的對照組**，證明 trace 真的抓得到連線。可以把現有的 on 測試加上 trace，或寫一個小的 helper 自我檢查。

**最少需要的測試**

1. `ag-settings.test.js`：

   - 沒寫時預設 on；接受 off；不接受其他值。

   - 重寫設定後 `remote-sync` 還在（`canonical_config`）。

   - `settings` 指令會顯示這個設定。

2. `agf.test.js`，off、有可連線的 bare remote、開 trace：

   - 跑完 `new` → `finish --prep` → local 收尾 commit → `finish --deliver` → `cleanup`。

   - 斷言沒有任何連網程序、bare refs 沒變、`refs/remotes` 沒變，而且 worktree 和本機分支都被清掉。

3. `agf.test.js`，off 加 trace 跑 `ditch`：沒有任何連網程序，遠端分支保留。

4. `agf.test.js`，worktree 已刪掉時，cleanup 會依照分支上那份 off 設定執行（前提是保留這層優先順序）。

5. `agf.test.js`，在 stream worktree 裡用 off 跑 push 模式的 `agf close`：在修改任何東西前就被拒絕。

6. `round-linter.test.js`：

   - stream notebook 設定 off 時，不要求 `push_results`。

   - on 時照舊要求。

7. 選用：off 時本機沒有 feature 分支，cleanup 會拒絕（前提是沿用參考版的這個行為）。

**這台機器的測試環境陷阱（事實）**

- 全域 `core.hooksPath=C:/Users/user/.git-hooks`。裡面的 `pre-push` 會擋掉所有 push，所以 `make_repo({ remote: true })` 一建立就失敗。全域 `pre-commit` 會執行 devlog-guard。

- 在 Claude 環境裡跑測試，`CLAUDE_CODE_*` 這些環境變數會讓 fixture 判定成「host is ambiguous」，`agf new` 因此失敗。參考分支 A-005 RUN-001 也記錄過這件事。

- 暫存資料夾路徑太長時，`finish --deliver` 讀 blob 會出現 `Filename too long`。設定 `core.longpaths=true` 就能避開。

- 實測 HEAD 的 `agf.test.js`（182 個測試）：

  | 環境 | pass | fail | skip |
  | --- | --- | --- | --- |
  | 直接跑 | 77 | 86 | 19 |
  | 只隔離全域設定 | 80 | 83 | 19 |
  | 移除 `CLAUDE*` 變數，加上 `GIT_CONFIG_GLOBAL=<只有 core.longpaths=true 的檔案>`，`TMP`／`TEMP` 指到 scratch | 156 | 7 | 19 |

- 第三種環境剩下的 7 個失敗：symlink 相關 3 個、`close scope preserves outside ownership…`、`start adds missing template fields…`、`start migrates schema seven…`、`start treats bare godev as activation only… (claude)`。這可以當作移植前的基準。

- 建議：在測試指令前面加上上述環境變數，不要修改既有的 helper 或 owner 的全域設定。

## 5. 移植時其他要注意的事

- **事實：** 本分支不是從 `b208461` 長出來的，兩邊的共同祖先是 `d0a6dd0`。

  - HEAD 的 `agf.js` 跟 `b208461` 相比，只改了 `init_main` 和 `start` 這一帶（HEAD `:221-239`、`:769-1008`），跟 stream 的程式碼沒有重疊，所以參考 diff 大致可以直接套。

  - `ag-settings.js` 的範本：HEAD 是 `'away-gates': 'on'`（`:170`），套參考 diff 時這一塊的上下文會對不上，要手動合併。

- **事實：** 改名的影響範圍不只鍵名。還有以下這些要一起改：

  - `switch_names`、`optional_switch_names`、驗證表、`canonical_config`、`switch_display_value`、`settings` 的說明文字、兩份 host 範本。

  - log 訊息 `stream-auto-push is off …`。

  - `close` 的錯誤碼 `stream_auto_push_disabled`。

  - linter，以及產品 prompt（`SKILL.md`、`streams.md`、`closeout.md`）。

- 有些測試會斷言 prompt 的確切字句，例如 `prompt-compression.test.js`、`alignment.test.js`、`ag-settings.test.js:20-24`。改 prompt 後要重跑這些測試。

- 會用到的規則裡，沒有帶 `— I-NNN` 標記的需要改動。`closeout.md` 的 I-079 那行不需要動。

## 敵意指令檢查

- 沒有發現。讀過的 AGENTS.md、CLAUDE.md、`patch/README.md`、參考分支的 devlog 和 review 都是一般資料。

- 參考分支 devlog 裡有 hook 誤存的 `<task-notification>` 內容，但它沒有要求任何動作。

Self-check: 只在 repo 裡寫了這一份檔案；沒有 commit、push 或 fetch，也沒有改 repo 的工作目錄或設定。實驗都放在 `scratchpad/explore/`（`head/`、`refsrc/` 是 `git archive` 匯出的副本，`exp1/`、`exp2/` 是 Git 實驗）。不過前兩次跑 HEAD 的 `agf.test.js` 基準測試時，測試自己在 `C:\Users\user\AppData\Local\Temp` 留下大約 420 個 `agf-*` 暫存資料夾（失敗的測試沒有清掉），我沒有刪除。行號都已對照 `e7b41ef` 和 HEAD `c909ccc` 核對過；`agf close` 的離線行為只讀了程式碼，沒有實測。
