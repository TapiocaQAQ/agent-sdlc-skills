# SDLC 進度:<feature 名> — <PR 名>

> **這一個 PR** 的 SDLC 單一真相來源。每跨一道關卡都要更新,PR merge 後**直接刪**。
> ⛔ 一個 PR 一支,不要一個 feature 共用一支(導航器每關都要讀它,共用的會長到數百行)。
> 跨 PR 才需要的東西(feature 層 gate 0 產出、跨 PR 決策、未關項目)留在 plan,這裡指回去。

- 開題:<YYYY-MM-DD>
- 規劃工具:superpowers | plan-feature
- 規劃產物:[plan/spec 連結]
- base branch:<x> / 工作 branch:<y>
- 嚴謹度(classify-change):leaf | core

## 📍 目前位置 → ⏭ 下一步    ← 永遠保持最新,這就是「SOP 報下一步」
- 目前:<第 N 步 名>
- 下一步:<明確指令/動作,如 `/simplify`>
- 卡點:<無 / 描述>

## 關卡進度(勾選 = 完成;🔄 = 進行中)

> ⚠️ **依執行順序排列,編號是名字不是順序**:第 8 關(commit)排在 9/10/Δ **之後**——
> external-ai-review 審的是 local diff,先 commit 會把審出來的修正堆成第二顆。詳見 SOP.md。

- [ ] 0. prompt-audit — 需求把關 ← **feature 層一次**;同 feature 的後續 PR 寫「不重跑,見 <哪裡>」
- [ ] 1. 規劃(superpowers / plan-feature)→ 產 plan/spec
- [ ] 2. 🚦 planning-exit checklist(人審)
- [ ] 3. classify-change — leaf/core 校準
- [ ] 4. 寫 code
- [ ] 5. /simplify            ← 發現先記「待 7 確認」,**第 7 關過了才編定案號**
- [ ] 6. /security-review     ← 同上(實測第 7 關推翻過 5/6 各一條)
- [ ] 7. /code-review max -fix
- [ ] 9. pr-prepare(第二檢查點 + 資料紅線)
- [ ] 10. ⛔ external-ai-review(去敏 + /loop 收斂)
- [ ] Δ. `/agent-sdlc:delta-enumerate` — 枚舉這一輪自己改過的每一行
      母體:<哪幾行,怎麼用 git diff 導出的> / 枚舉了什麼:<軸與兄弟值> / 發現:<N 條>
      達門檻升級第二次 gate 7:<是|否> / **Δ 修了**:<清單> / **留給人審**:<清單> / Δ′:<結果>
      ⚠️ Δ 空(9/10 零採納)就寫「Δ 跳過」——跟「跑了但沒發現」在檔上必須長得不一樣
- [ ] 8. 🚦 commit-message(人審 + Co-Authored-By)
- [ ] 11. 🚦 人審合併(core 逐行;push/PR/merge 核可)

## 紀錄 / 連結
- commit(s):
- PR:
- external-review 摘要:
- Δ 摘要(母體 / 發現 / 有無升級):
- 決策 / 偏移(planned leaf 卻漂 core 等):
