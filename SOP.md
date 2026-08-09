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
| Δ | **枚舉這一輪自己改過的每一行**(只報告、不 auto-fix) | 無 skill,照下方定義手動跑 | 本地新增 |
| 11 | 🚦 人審合併(core 逐行;push/PR/merge 核可) | 人審閘 | 本地規範 |

- 🚦 **人審閘 ×4**:第 2、8、11(其中 8、11 含核可動作)。
- ⛔ **去敏閘 ×1**:第 10。
- **內建關卡 ×3**:第 5、6、7(Claude Code 內建、非本 pack,無法自回呼;由**第一個跑到的 pack 關卡**回呼時追認——依下方執行順序即第 9 關 pr-prepare)。

## 執行順序:`… → 7 → 9 → 10 → Δ → 8 → 11`

⚠️ **編號是名字,不是順序。** 實際執行順序把第 8 關(commit)往後挪,並在它前面插入 Δ:

```
0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 9 → 10 → Δ → 8 → 11
```

**為什麼 8 排到 9/10 後面**:`external-ai-review` 審的是 **local diff**。先 commit 再送審,
等於把「審出來的修正」堆成第二顆 commit,或者要 amend。commit 排最後,一顆 commit 就是收斂後的結果。
(上游 appleboy 的順序 8 → 9 → 10 成立,是因為他那兩關是 **GitHub PR bot**,本來就需要先有 commit 和 PR。
本 pack 的 reviewer 是同步的 local reviewer,前提不同,所以順序不同。)

### Δ:枚舉這一輪自己改過的每一行

**這一關存在的理由**,一句話:**抽樣式審查(外部 AI、人工 review、隨機探測)天生找不到
「你剛剛修的那條修得不夠寬」**——因為那看起來已經修過了,沒有人會回頭再抽同一個位置。
但那其實是**命中率最高的地方**:寫修正的人剛剛才證明自己在這條軸上想得不夠寬。

| | 內容 |
|:--|:--|
| **母體** | 第 5/6/7/9/10 關**這一輪改過的每一行**,不是整個 diff。剛改的那幾行當成最可疑的區域先掃 |
| **方法** | 對每個決策點**枚舉兄弟值**(相鄰拼法、邊界值、同軸的其他成員),不是再抽樣一輪 |
| **🔴 只報告,不 auto-fix** | 這是讓遞迴終止的那一條。Δ 產出的是一份發現清單,交給下面的門檻判斷 |
| **跳過條件** | 第 9/10 關零採納 ⇒ Δ 母體是空的 ⇒ 跳過,並在進度檔寫明「Δ 空,已跳過」 |

**升級門檻(按性質,不按數量)**:Δ 只要出現 **≥1 條**「**會實際走到危險路徑,或推翻某條已經寫下來的
保證**」的發現 → **升級成完整的第二次 gate 7**(`/code-review max -fix`),跑完再回 Δ。
nit、風格、可讀性建議**不算**,不論幾條。

> 用性質而不是數量,是因為「數量」對這類缺陷是壞指標:一條會帶著憑證出去的洞,本身就足以證明
> 這一輪的判斷寬度不夠;而三條 nit 湊出來的升級只是白跑一次 gate 7。

**依據**:Fagan inspection 的 **Follow-up 階段**——它的定義就是「確認缺陷都修好了,
**而且修的過程沒有引入新缺陷**」,而且它由**人**做驗證,不是再跑一次完整 inspection。
Δ 只報告、把裁決交給第 11 關人審,對的就是這個形狀。

**寫進進度檔**:Δ 一定要留下「母體是哪幾行 / 枚舉了什麼 / 發現幾條 / 有沒有達門檻」。
Δ 跑完沒發現,和 Δ 沒跑,在進度檔上必須長得不一樣。

## 導航機制

- **顯式**:隨時打 `/agent-sdlc:sdlc` 問「我到哪了、下一步是什麼」。
- **自動**:在跑完整流程時(進度檔存在),每個 pack skill(第 0/1/3/8/9/10 關對映的 6 支)收尾會回呼 `/agent-sdlc:sdlc`,自動勾選本關、刷新下一步;單獨用一支 skill 則不回呼、只在該支收尾提醒下一步。
- **進度檔**:每功能一支、放開發 repo `docs/sdlc/<feature>-sdlc-progress.md`、**gitignored working scratch**(同 mem-tmp 性質)、**結案(第 11 關 merge)即刪不歸檔**——commit + PR 本身就是永久紀錄。
