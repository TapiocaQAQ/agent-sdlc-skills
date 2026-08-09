# SDLC 進度:<feature 名>

> 此功能 SDLC 的單一真相來源。每跨一道關卡都要更新,直到結案。

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

- [ ] 0. prompt-audit — 需求把關
- [ ] 1. 規劃(superpowers / plan-feature)→ 產 plan/spec
- [ ] 2. 🚦 planning-exit checklist(人審)
- [ ] 3. classify-change — leaf/core 校準
- [ ] 4. 寫 code
- [ ] 5. /simplify
- [ ] 6. /security-review
- [ ] 7. /code-review max -fix
- [ ] 9. pr-prepare(第二檢查點 + 資料紅線)
- [ ] 10. ⛔ external-ai-review(去敏 + /loop 收斂)
- [ ] Δ. 枚舉這一輪自己改過的每一行(只報告、不 auto-fix)
      母體:<哪幾行> / 枚舉了什麼:<> / 發現:<N 條> / 達門檻升級 gate 7:<是|否>
      ⚠️ Δ 空(9/10 零採納)就寫「Δ 跳過」——跟「跑了但沒發現」在檔上必須長得不一樣
- [ ] 8. 🚦 commit-message(人審 + Co-Authored-By)
- [ ] 11. 🚦 人審合併(core 逐行;push/PR/merge 核可)

## 紀錄 / 連結
- commit(s):
- PR:
- external-review 摘要:
- Δ 摘要(母體 / 發現 / 有無升級):
- 決策 / 偏移(planned leaf 卻漂 core 等):
