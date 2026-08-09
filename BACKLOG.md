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

### 2. 中文鏡像的同步紀律（漂移已補平，紀律未建立）

- **觀察（2026-08-10）**：各 `SKILL.zh-TW.md` 檔頭寫「與同目錄 SKILL.md 同步」，但實際上沒有人在維持。最嚴重的是 `external-ai-review`：主檔 **406 行**、鏡像只有 **80 行** —— 計分卡、跨輪覆蓋帳本、四道提案檢驗、fan-out 透鏡設計整段都沒翻。其餘 6 支結構是齊的（heading 數逐支相同）。
- **✅ 2026-08-10 已補平**：`external-ai-review` 鏡像完整重譯（80 → 400+ 行）、新增 `delta-enumerate` 鏡像、`sdlc` 鏡像補上 Δ 段與「內建關卡在第 9 關追認」。**8 支主檔對 8 支鏡像，heading 結構逐支一致。**
- **待討論的是「怎麼不再漂移」**：
  - 加一個 CI／pre-commit 檢查（比對兩邊 heading 清單，不一致就擋）？
  - 還是把鏡像**降級成導讀**（只留概念與指令表、不複製規則），讓它結構上不可能漂移？
  - 現況靠「改主檔時記得改鏡像」——**這正是本 pack 自己記錄過會失效的那個模式**（「靠人記得」＝沒有機制）。
- **注意**：漂移一旦發生就是靜默的——鏡像不會報錯，只會在有人讀它時給出過期的規則。

### 3. 已提出但**撤回**的兩條簡化（2026-08-10 SDLC review）

留在這裡標明作廢，避免下次又被當成好點子重提。

| 撤回的提案 | 為什麼撤回 |
|:--|:--|
| **external-ai-review 輪數上限 10 → 3** | 過不了該 skill 自己的四道檢驗：**①replay** —— PR-B 第 3 輪產出整關最好的一條（D17），砍到 2 輪會失去它；**③** 移除覆蓋要有機制化證明，不能拿分數當理由。而且該 skill 已寫著實測反例：2026-08-06 第 10 輪採納率最高、產出全 run 最大的一條修正，並把「提早停」列為計分卡的作弊向量。|
| **gate 9 的 leaf/core 重判降成 trip-wire** | 依據只有「三輪 0/3 沒改變判定」——**那是分數不是證明**（同 ③）。而且 PR-B 那次重判把 Q5／Q6 拆開（可用性 blast radius 零、但閘一旦 fail-open 就是寫入），**那正是「可以先合但仍要逐行」的依據**，降成 trip-wire 就沒有了。|

📌 這一輪**真正採納**的六條見 `SOP.md` 與各 skill 的 2026-08-10 修訂。

### 4. 全新 repo 上 gate 8 / gate 11 會 collapse（創世 commit 情境）
- **觀察（2026-07-13 同上）**：pack 的 commit-message(gate 8) 與 pr-prepare/人審合併(gate 11) 都預設 repo 已有歷史可 diff。跑在**零 commit 的全新 repo** 上時退化為：單一創世 commit（描述整個 app、非只 feature diff）＋直推 master（無 base 分支 → 無正式 PR/merge，發佈即核可）。
- **待討論**：SKILL.md 或 SOP.md 要不要補一段「創世 commit / 全新 repo」的處理指引，讓首次落地不必臨場推敲？（後續 feature 會拿到乾淨獨立的 feat commit，此問題只在 repo 首次。）
