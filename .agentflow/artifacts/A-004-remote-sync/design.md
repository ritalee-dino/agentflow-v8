* _2026-10-12 14:44:57 +0800 (claude-opus-5-5/high)_

# A-004 設計：`remote-sync` 設定

**摘要**

- 新增 `remote-sync: on|off`（預設 `on`）。`off` 時，stream 的建立、收尾、交付、cleanup、ditch 和 `agf close` 都只動本機 Git，不 fetch、不查詢遠端、不 push；主專案設定為 `off` 時，主工作區 closeout 也一樣。

- 做法：把 personal/v8.4.14 的 5480558、8e020a2、e7b41ef 改名後移植（已確認可乾淨套用），再修掉 explore 找到的一個新失敗方式。

- 不升版、不改 CHANGELOG。

## 依據與決定

- Owner 原話：「參考分支 personal/v8.4.14 中修改的程式，在本分支中新增一個設定值，remote-sync，若為 off ，預期 stream 流程完全在本機執行，不進行任何遠端擷取、查詢或推送」

- D-1（owner 選）：只用 `remote-sync`，不加 `stream-auto-push` 別名；本機根目錄 `ag.json` 的 `"stream-auto-push": "off"` 換成 `"remote-sync": "off"`，其他 key 不動。

- D-2（owner 選）：主專案設定 `off` 時，主工作區 `agf close` 也拒絕 push 模式，linter 不要求 push 證據，SKILL.md 的 push 規則加例外。

- D-3（host 預設，沿用參考分支 A-005）：可讀本機快取但不寫；本機沒有 feature 分支時拒絕並給提示；快取顯示遠端較新時只警告；`off` 時跳過 origin 網址檢查。

- D-4（沿用 A-002 owner 指示）：不升版。

- 需求細節見 [requirements-report.md](requirements-report.md)，風險與實測見 [explore-report.md](explore-report.md)。

## 和參考實作的差異

- **改名：** 所有 `stream-auto-push`／`stream_auto_push` 改成 `remote-sync`／`remote_sync`；錯誤代碼 `remote_sync_disabled`。

- **只檢查一個值：** 參考版 `stream_auto_push_from_text` 會驗證整份 `ag.json`，舊版或缺欄位的 stream 設定會讓 `finish`、`cleanup`、`ditch` 連 `on` 都被拒絕（explore 實測）。改成：JSON 可解析、沒有重複 key、`switches` 是物件、`remote-sync` 不存在或是 `on`/`off`；其餘一律不檢查。

- **說明文字：** `settings` 說明改成講清楚 off 不 fetch、不查詢、不 push。

## Invariants

- **INV-1：** 設定為 `on` 或不存在時，所有指令的遠端行為與 HEAD c909ccc 相同。違反條件：既有 on 測試新增失敗。

- **INV-2：** stream 設定為 `off` 時，`new`、`finish --prep`、`finish --deliver`、`cleanup`、`ditch`、`close`（local）不啟動任何連網 Git 程序。違反條件：trace 裡出現 fetch、push、ls-remote、upload-pack、receive-pack 或 remote-*。

- **INV-3：** stream 有筆記但設定讀不到或值無效時，cleanup／ditch／finish 在任何變更前拒絕。違反條件：預設成 on 繼續或改動了 ref。

- **INV-4：** linter 只依自己讀到的設定免除 push 證據；讀不到時照舊要求。

## Acceptance criteria

- **AC-1（INV-1）：** `ag-settings.test.js` 新案例：缺 key 視為 on、接受 off、拒絕 `maybe`、寫回保留、`settings` 顯示。

- **AC-2（INV-2）：** `agf.test.js` 的 off 完整流程（new → prep → deliver → cleanup）與 ditch，用 `GIT_TRACE2_EVENT` 斷言沒有連網程序，bare remote refs 不變。

- **AC-3（INV-3）：** stream 設定損毀時 cleanup 拒絕且 `nothing was changed`；只缺必填欄位的舊版設定不會讓 on 被拒。

- **AC-4（D-2）：** 主專案 off 時 push 模式 `agf close` 回 `remote_sync_disabled`，且沒有 fetch。

- **AC-5（INV-4）：** `round-linter.test.js` 的 off／on／讀不到三種情況。

- **AC-6：** 三個相關測試檔與基準比對，沒有新增失敗；文件中 `stream-auto-push` 為 0 筆。

## Minimality check

- **最小結果：** 一個可省略的 on/off key，加上每個連網呼叫點前的同一個判斷。

- **考慮過的更簡單做法：** 只看主專案 `ag.json`，不看 stream 自己的設定。不採用：參考實作和 owner 先前決定都允許 stream 單獨調整，而 cleanup 在 worktree 已刪除時仍要讀到 stream 的值。

- **每個部分為什麼需要：**

  - 讀 stream 設定的來源順序（worktree → 已 commit 的分支 → 快取）：cleanup 從主工作區操作，worktree 可能已不存在。

  - linter 例外與 SKILL.md／streams.md／closeout.md 文字：不改的話 host 每輪 closeout 仍會 push，達不到「完全在本機」。

  - 參考版的 no-origin 提示與 last-fetch 警告：沿用 D-3，不另外擴充。

- **不做（列為提案）：** A-006 F-3（`git remote` 失敗時誤判為沒有 origin）、統一 `agf new` 與 `finish` 判斷 remote 的方式。

Self-check: 設計涵蓋 owner 原話、四個決定、與參考實作的兩處差異、INV-1～4 和 AC-1～6；沒有加入原話以外的新行為。
