---
name: external-ai-review
description: 用外部 AI 對 diff 做第二審並修正;model-agnostic 可插拔 reviewer(v1=sub-gemini,經瀏覽器驅 Gemini),單輪 check-fix 配 /loop,取代 appleboy 的 copilot-review/codex-review
---

> **這是 `SKILL.md`(英文)的中文鏡像,給中文語系維護者/讀者看。**
> **被 Claude 觸發/載入的權威版是英文 `SKILL.md`;本檔不被當 skill 觸發。兩版永遠一起更新。**
>
> ⚠️ **本支性質(與其餘 5 支不同)**:external-ai-review 是**全新自寫**,**非 vendor 自 appleboy**——它是我們對 appleboy `copilot-review` / `codex-review` 的**替代**(換底座:GitHub PR 機器人 → 本地 diff + 同步 reviewer)。
> **搬過來的 appleboy 精神**:①不盲信、逐條核實 ②`Fix:/Test:` 回報紀律(改記 mem-tmp)③cap-10→轉人審 ④明確終止條件。**刻意不搬**:appleboy 那套「防讀到舊 review」的 race 機制(對同步 Gemini 不需要)。
> **紅線(appleboy 沒有、我方獨有)**:diff 送出給外部 reviewer 前,**必先剝除真實資料/密鑰/個資**。
> **Roadmap**:`copilot-review` / `codex-review` 日後照「reviewer 介面」實作即可插入(需 Copilot Pro / Codex Plus 訂閱)。

# External AI Review(外部 AI 第二審)

給一份**已完成、待 PR** 的變更,用當下可用的外部 AI 審查者做第二視角審查,再把它**確實抓到**的問題修掉。**reviewer 是介面,不是硬綁某模型**——新增審查者=實作介面,不改主流程。

這是我們對 appleboy `copilot-review` / `codex-review` 的對應方案。那兩支驅動真正的 GitHub PR 機器人(Copilot / Codex)去審**已 push 的 PR**、用 `gh` 管理 review threads。本 skill 改成用**同步** reviewer 審**本地 diff**,所以能在**還沒開 PR 之前**就跑,且除了你既有的 Gemini 存取外**不產生額外費用**。(`copilot-review` / `codex-review` 列 roadmap,作為替代的 reviewer 實作——見 pack README。)

## 何時用

- `/agent-sdlc:pr-prepare` 產出 PR 描述之後、或 commit 之前,想要一個獨立的第二審。
- 對映 appleboy 的 `/loop 3m /copilot-review`:配 `/loop` 反覆審到乾淨。

## reviewer 介面(可插拔)

一個 reviewer 需能:① 接收「變更摘要 + diff + 關注點」② 回「問題清單(`檔:行` + 嚴重度 + 建議修法)」。

- **v1 實作 = sub-gemini**(Gemini Pro,經你已登入的 Chrome,走 `sub-gemini` skill)。
- 日後 Copilot / Codex CLI(或任何其他 reviewer)可用時,實作同一介面即插入——主流程不動。

## 流程(單輪 check-fix)

每次呼叫跑**一**輪。配 `/loop` 重複到乾淨。

### 1. 備審查輸入

- 用 `git diff`(branch 用 `git diff main...HEAD`)取變更。
- 摘要:改了什麼、`classify-change` 判 leaf/core、要 reviewer 特別看什麼(如金額精度、auth、邊界、錯誤路徑)。
- ⛔ **敏感資料紅線——這是讓「外部」reviewer 安全的關卡。** 任何東西離開你的機器交給 reviewer 前,**先剝掉每一筆真實資料/密鑰/個資**:真實 CSV/DB 內容、載具號碼、驗證碼、`.env` 值、API key、token、真實使用者記錄。**只送程式碼 diff 與合成範例**。`copilot-review` / `codex-review` 不需要這步——它們的 diff 已在 GitHub 內;這裡 diff 是**往外送**給 Gemini,所以去敏是強制的。若某段 hunk 不去敏就沒法給,就把資料換成合成佔位、或略過該段並註記。

### 2. 委派 reviewer(v1: sub-gemini)

- 呼叫 `sub-gemini` skill 並照它整套 playbook(開新 `/app` tab、`fill` 後用 JS `eval` 送出、`get url` 驗證送出、輪詢到「停止回覆」控制項消失、取最後一則 `message-content`)。**不在這裡重抄那些瀏覽器細節**——那是 `sub-gemini` skill 的責任。
- prompt 要自包含:附變更摘要 + **去敏後**的 diff + 明確要求:「列出問題:`檔:行`、嚴重度(`blocker` / `warn` / `nit`)、如何修」。

### 3. 驗證 reviewer 的回覆(Claude 的責任——**不要盲信**)

- Gemini 比程式碼專用機器人更會幻覺,所以**每個 finding 都是「待查的主張」,不是「照做的指令」。** 逐條對照真實 diff,確認那段 code 與 `檔:行` 真的存在、問題屬實,才採納。
- 涉及事實、版本、API 契約的建議,**Claude 自行二次求證**,不照 Gemini 的話照收。
- 幻覺、過時、與架構衝突的 finding 一律剔除,並記下**每一條為何剔除**(這是 appleboy「resolve 前先回覆理由」的本地等價物)。

### 4. 修正

- 採納的 `blocker` / `warn` 逐一修 code;適合處補一個能釘住此修正的測試。
- `nit` 視情況處理。
- 修正累積在 working tree——**不做每輪 commit/push**(沒 PR 可推)。等 loop 收斂後才用 `/agent-sdlc:commit-message` 一次 commit;PR 交給 `/agent-sdlc:pr-prepare`。

### 5. 配 /loop 收斂

- 用 `/loop` 重跑本 skill,直到 reviewer 對**當前 diff 無新的 `blocker` / `warn`**。這個乾淨通過是**唯一**的停止條件。
- 每輪把「已修」與「略過(附理由)」記進 mem-tmp——這就是 audit trail(appleboy 記在 GitHub PR threads,我們記在 mem-tmp)。
- **cap 10 輪。** 超過通常代表架構級歧見,不是可修的 finding——停下,轉交使用者人審。

## 與 pack 其他 skill 的關係

- 通常在 `/agent-sdlc:pr-prepare` 之後跑;修完再更新 PR。
- **不取代人審。** `classify-change` 判 **core** 的變更仍需人逐行審——本 skill 是那道審查的輔助,不是替身。

## Context 檢查點

長 review loop 很吃 token。每輪結束檢查 context 用量;偏高先更新 mem-tmp 續接塊,再提醒使用者 `/compact`。

## 本關完成後

<!-- DELIBERATE DELTA(vs upstream appleboy/skills):回呼 sdlc 改為「進度檔存在才接」的條件式,單獨用一支不被逼進完整流程。刻意為之——別還原成無條件 REQUIRED 版。 -->

**在跑完整 agent-sdlc 生命週期?** 若有 `docs/sdlc/<feature>-sdlc-progress.md`(或你刻意啟動整條 SOP 鏈),呼叫 `/agent-sdlc:sdlc`——它勾掉本關並回報確切的 ⏭ 下一步。只導航,不替你跑下一關。

**單獨用這支 skill?** 那你已完成——**不要**呼叫 `/agent-sdlc:sdlc`(沒有進度檔可更新)。若想繼續,通常接的下一步是 **gate 11 🚦人審逐行 + 合併(無對應 skill,最終人審閘)**。
