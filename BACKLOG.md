# agent-sdlc pack — backlog / 待討論

> 開發／驗收過程中冒出、值得之後處理的項目。**管理方式尚未定案**：現在先記在本檔（repo TODO），還是等 pack push 到 GitHub 後改用 issue 追蹤——待與使用者討論。

## 待討論

### 1. external-ai-review 對「已過 built-in review 的 LEAF」的定位與價值敘述
- **觀察（2026-07-13，Tapioca-Accounting `import-visibility` 主驗收實測）**：對一個已經過 gate 6 `/security-review`(PASS) + gate 7 `/code-review`(0 findings) 的 **LEAF** 變更跑 external-ai-review，web Gemini 兩輪提出的 5 個 findings（1 blocker + 2 warn + 2 nit）**全部不是當前真實缺陷**——競態條件、覆蓋使用者手改分類、全表載入效能、金額負值排序，都因專案的實際約束（單人單行程、無手動編輯端點、頁面本就全量渲染、電子發票金額全正）而不成立；第 2 輪餵入真實約束後 Gemini **自行撤回全部**。
- **推論**：這一關的**實質防線不是「Gemini 抓到缺陷」，而是「Claude 逐條對真實碼驗真、駁回幻覺並記錄原因」**。Gemini 幻覺率偏高，盲目採納反而有害。
- **待討論怎麼辦**：
  - external-ai-review `SKILL.md` 要不要**明講**此關對「已過 built-in review 的 leaf」的定位＝加碼第二視角 + 驗證機制（而非期待它找 bug），並強化「Claude 必須逐條驗真、預設懷疑」的義務描述？
  - 或維持現狀（skill 已有「不盲信、逐條 verify」段落，只是沒點明對 leaf 的期望值）？
  - **流程問題**：這類「開發中發現」該用本 repo `BACKLOG.md` 管，還是等 pack push 到 GitHub 後改開 issue 追蹤？
- **注意**：依協作規範，修改自製 skill 前須先報告、經使用者同意——本項僅**記錄待討論**，未動 `SKILL.md`。

### 2. 全新 repo 上 gate 8 / gate 11 會 collapse（創世 commit 情境）
- **觀察（2026-07-13 同上）**：pack 的 commit-message(gate 8) 與 pr-prepare/人審合併(gate 11) 都預設 repo 已有歷史可 diff。跑在**零 commit 的全新 repo** 上時退化為：單一創世 commit（描述整個 app、非只 feature diff）＋直推 master（無 base 分支 → 無正式 PR/merge，發佈即核可）。
- **待討論**：SKILL.md 或 SOP.md 要不要補一段「創世 commit / 全新 repo」的處理指引，讓首次落地不必臨場推敲？（後續 feature 會拿到乾淨獨立的 feat commit，此問題只在 repo 首次。）
