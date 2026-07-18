# agent-sdlc 預設 SDLC 流程(SOP)

這是 pack 的**預設完整 SDLC 流程**,忠實對應 appleboy(Bo-Yi Wu, MediaTek)Cloud Summit 2026 的 Agent-Skill SDLC 鏈。**不自行簡化關卡。**

用法:每個功能開一支進度檔(見 `plugins/agent-sdlc/templates/sdlc-progress.template.md`),由 `/agent-sdlc:sdlc` 導航——它讀進度檔 + git 狀態,更新勾選並回報「📍目前位置 / ⏭下一步 / 卡點」。你只要照著下一步做;每過一個 pack 關卡,該關 skill 會自動回呼 `sdlc` 更新進度。

## 12 步 master table

| # | 關卡 | 類型 | 來源 |
|---|---|---|---|
| 0 | prompt-audit — 需求把關 | pack skill | 本 pack |
| 1 | 規劃(雙軌 superpowers/plan-feature)→ 產 plan/spec | 規劃 | superpowers 或 pack |
| 2 | 🚦 planning-exit checklist | 人審閘 | pack README 檢查表 |
| 3 | classify-change — leaf/core 校準 | pack skill | 本 pack |
| 4 | 寫 code | 實作 | — |
| 5 | /simplify | 內建 | Claude Code |
| 6 | /security-review | 內建 | Claude Code |
| 7 | /code-review max -fix | 內建 | Claude Code |
| 8 | 🚦 commit-message(人審 + Co-Authored-By) | pack skill / 人審閘 | 本 pack |
| 9 | pr-prepare(第二檢查點 + 資料紅線) | pack skill | 本 pack |
| 10 | ⛔ external-ai-review(去敏 + /loop 收斂) | pack skill / 去敏閘 | 本 pack |
| 11 | 🚦 人審合併(core 逐行;push/PR/merge 核可) | 人審閘 | 本地規範 |

- 🚦 **人審閘 ×4**:第 2、8、11(其中 8、11 含核可動作)。
- ⛔ **去敏閘 ×1**:第 10。
- **內建關卡 ×3**:第 5、6、7(Claude Code 內建、非本 pack,無法自回呼;由第 8 關 commit-message 回呼時追認)。

## 導航機制

- **顯式**:隨時打 `/agent-sdlc:sdlc` 問「我到哪了、下一步是什麼」。
- **自動**:在跑完整流程時(進度檔存在),每個 pack skill(第 0/1/3/8/9/10 關對映的 6 支)收尾會回呼 `/agent-sdlc:sdlc`,自動勾選本關、刷新下一步;單獨用一支 skill 則不回呼、只在該支收尾提醒下一步。
- **進度檔**:每功能一支、放開發 repo `docs/sdlc/<feature>-sdlc-progress.md`、**gitignored working scratch**(同 mem-tmp 性質)、**結案(第 11 關 merge)即刪不歸檔**——commit + PR 本身就是永久紀錄。
