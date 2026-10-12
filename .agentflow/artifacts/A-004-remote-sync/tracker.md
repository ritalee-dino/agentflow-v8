# Tracker

## Identity

- **Work key:** A-004-remote-sync.

- **Active Ask:** A-004.

- **Goal:** 新增 `remote-sync: on|off`；off 時 stream 流程與主工作區 closeout 完全不 fetch、不查詢、不 push。

- **Last update:** 2026-10-12 15:15:00 +0800.

- **Evidence commit:** 8f86b0826eb2ab783d2027ab4d944a3a1980b065.

## Overall state

- **State:** complete.

- **Reason:** All accepted tasks are proven.

- **Total:** 4.

- **Completed:** 4.

- **Remaining:** 0.

## Accepted task checklist

- [x] **T-1:** 套用改名後的參考補丁（5480558、8e020a2、e7b41ef，`stream-auto-push` → `remote-sync`）到 ag-settings.js、agf.js、round-linter.js、對應測試與文件；`remote_sync_from_text` 改成只檢查 `remote-sync` 這個值；`settings` 說明寫清楚 off 不 fetch、不查詢、不 push。範圍為補丁原本的 13 個檔案，加上 codewalk 指出仍寫無條件 push 的 docs/agent/FLOWS.md、ARCHITECTURE.md。Proof: `ag-settings.test.js` 新案例 PASS。Source: A-004.

- [x] **T-2:** 補測試：off 完整流程與 ditch 用 `GIT_TRACE2_EVENT` 證明沒有連網程序；舊版／缺欄位 stream 設定在 on 時不被拒絕；off 時有 origin 但沒有快取的 cleanup 提示。只改 `agf.test.js`。Proof: 新案例 PASS，三個相關測試檔與 550/20/22 基準比對沒有新增失敗。Source: A-004.

- [x] **T-3:** 本機根目錄 `ag.json`：只把 `"stream-auto-push": "off"` 換成 `"remote-sync": "off"`，其他 bytes 不動；`ag.json` 已被 Git 追蹤，這一行改動隨實作 commit。Proof: 讀回檔案並跑 settings 驗證。Source: A-004（owner 選項 D-1）。

- [x] **T-4:** commit、獨立審查、closeout。本專案 `ag.json` 已是 `remote-sync: off`，依 D-2 主工作區只做本機交付、不推送。Proof: review.md 三項 PASS（Reviewed commit 8f86b08）；agf close local 模式成功。Source: A-004.

## Accepted scope changes

- T-4 原訂推送分支；owner 選 D-2「一起停」並同意本機 ag.json 改為 remote-sync: off。Effect: 交付改為本機 commit、不推送。Source: A-004.

## Current recovery

- **Current item:** none.

- **Last proven result:** 12 個 remote-sync focused 測試 PASS（含 GIT_TRACE2 無連網斷言）；文件測試與完整 HEAD 快照同樣 2 個既有失敗；本機 ag.json 驗證通過且讀到 off。

- **Active blocker or running process:** None.

- **Next safe action:** none.

- **Expected changed files:** docs/agent/FEATURES.md; skills/agentflow/SKILL.md; skills/agentflow/docs/AG_GUIDE.md; skills/agentflow/docs/AG_GUIDE.zh-tw.md; skills/agentflow/references/closeout.md; skills/agentflow/references/streams.md; skills/agentflow/scripts/README.md; skills/agentflow/scripts/ag-settings.js; skills/agentflow/scripts/ag-settings.test.js; skills/agentflow/scripts/agf.js; skills/agentflow/scripts/agf.test.js; skills/agentflow/scripts/round-linter.js; skills/agentflow/scripts/round-linter.test.js; docs/agent/FLOWS.md; docs/agent/ARCHITECTURE.md; ag.json; .agentflow/artifacts/A-004-remote-sync/*.

## Completion proof

- **All accepted tasks checked:** yes.

- **Blocking accepted decision:** none.

- **Operation running:** no.

- **Next action remaining:** none.

- **Evidence status:** complete.

- **Judgment:** complete.
