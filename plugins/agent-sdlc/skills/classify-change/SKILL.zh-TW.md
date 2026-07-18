---
name: classify-change
description: 把一個提議中的程式碼變更分類為 Leaf Node(葉節點)或 Core Code(核心程式),以決定該用多少 AI、以及多嚴的 review 才恰當。當使用者問「這是 leaf 還是 core」「這是哪種變更」「這個可以 vibe code 嗎」「AI 可以寫這個嗎」「review 要多嚴」,或在開始任何非瑣碎變更之前,就用這支 skill。也適合當作其他工作流(規劃、PR 準備、code review)裡的子步驟——當分類會影響後續做法時。它問 6 個診斷問題:傳播範圍、變更頻率、技術債容忍度、例子契合、故障成本、需要的 review 強度,然後輸出清楚的分類與建議做法。即使使用者只是描述一個變更、沒明說要分類,也要觸發——因為答案會改變他該怎麼往下走。
---

# classify-change(中文鏡像)

> 📌 本檔是 `SKILL.md`(英文權威版)的中文鏡像,給中文語系維護者/讀者閱讀用。**Claude 實際載入/觸發的是英文 `SKILL.md`;本檔不會被當 skill 觸發。** 兩版內容一一對應,**每次改動兩版必須一起更新**。

在 production 做 AI 輔助寫程式,最重要的單一規則:**把 AI 的大量產出侷限在葉節點(leaf node)**——在那裡,難免會產生的一點技術債不會擴散穿透整個系統。這支 skill 把「leaf vs core」的判斷變得具體且一致,讓個人的判斷不會在團隊裡飄移。

## 何時使用

只要「leaf vs core」的區分重要就用。觸發情境包括:

- 「這是 leaf 還是 core?」
- 「這個變更是哪種分類?」
- 「這個可以 vibe code / AI 可以寫嗎?」
- 「我需要多仔細 review 這個?」
- 開始任何非瑣碎工作之前,即使使用者沒問
- 在 `plan-feature` 或 `pr-prepare` 之前或之中——當 leaf/core 的判斷應該明確、且與本框架一致時

只有在變更明顯瑣碎時才略過(錯字修正等)。

## 兩個類別

**葉節點(Leaf node)。** 沒有其他東西依賴它的程式碼——功能端點、UI 元件、腳本、報表、一次性 migration。故障是局部的。這裡的技術債不會擴散。**適合高 AI 涉入、較輕的 review。**

**核心程式(Core code)。** 很多東西依賴它的程式碼——auth、payment、資料 schema、公開 API、共用框架、orchestrator。故障是全系統的。這裡的技術債會複利累積。**即使有 AI 協助,也需要人來主導並逐行 review。**

當一個變更同時跨到兩者,它算 **core**(較嚴的規則勝出)。

## 六個診斷問題

依序走過這些。答案若無法從 codebase 直接看出,就取得使用者的輸入。

### Q1 — 傳播範圍

> 如果我們改這個,有什麼依賴它?

- 少數呼叫者(一兩個特定地方)→ 偏 leaf
- 很多呼叫者 / 很多模組 → 偏 core

### Q2 — 預期變更頻率

> 這塊 codebase 會持續演進,還是已接近「完成」?

- 可預見的未來大概穩定 → 偏 leaf
- 會持續演進、需要保持可擴充 → 偏 core

### Q3 — 技術債容忍度

> 如果這段程式碼變得有點醜,未來的工作會受害嗎?

- 一點債沒關係、被侷限住 → 偏 leaf
- 這裡的債會卡住其他工作 → 偏 core

### Q4 — 契合的例子

> 這個變更最像下列哪一個?
>
> - 報表、儀表板、單一端點、UI 元件、腳本、一次性 job → **leaf**
> - Auth、payment、資料 schema、公開 API、共用工具、orchestration → **core**

### Q5 — 故障成本

> 如果這段程式碼出錯,故障會擴散多遠?

- app 的某個區域、容易 rollback → 偏 leaf
- 全系統、影響使用者 / 資料 → 偏 core

### Q6 — 需要的 review 強度

> 這個變更可以靠「介面 + 測試」就信任,還是必須有人讀每一行?

- 介面 + 測試就夠 → 偏 leaf
- 需要逐行閱讀 → 偏 core

## 決策邏輯

走完這些問題後:

- **全部 / 多數偏 leaf** → 分類為 **leaf**
- **全部 / 多數偏 core** → 分類為 **core**
- **混合 / 跨兩者** → 分類為 **core**(最嚴的規則勝出;考慮把變更拆開)

## 輸出格式

```markdown
## Classification: [LEAF NODE | CORE CODE]

## Why
- Q1 (propagation): <answer> → <leaf/core>
- Q2 (frequency): <answer> → <leaf/core>
- Q3 (tech debt): <answer> → <leaf/core>
- Q4 (examples): <answer> → <leaf/core>
- Q5 (failure cost): <answer> → <leaf/core>
- Q6 (review intensity): <answer> → <leaf/core>

## Recommended approach

### If LEAF
- 適合高 AI 涉入
- 1 個 reviewer 就夠
- 信任測試 + 介面;不需要讀每一行
- 至少 3 個 e2e 測試(1 happy + 2 errors)
- 仍需要 plan,但 plan.md 可以輕量

### If CORE
- AI 可協助,但必須由人主導設計並負責 review
- 2+ reviewer,含模組 owner
- 逐行 review diff
- PR 描述必須標明哪些檔案經過人工逐行審查
- 若變更很大,拆成「人主導的 core 變更」+「leaf 功能變更」
```

## 下游契約(接線)

本 skill 擁有的是 leaf/core 的**準則(rubric)**——那 6 個問題,加上「較嚴者勝」的決勝規則。下游 skill 套用**同一套準則**,而不是自己另發明一套,所以判斷保持一致:

- **`plan-feature`** 對**意圖 scope**(程式碼還不存在時)套用準則,以設定 review 強度與 e2e 數(leaf → 3 個 e2e 就夠;core → 更重的驗證 + 人工逐行)。
- **`pr-prepare`** 對**真實 diff**(程式碼寫完後)**重套同一套準則**。這是刻意設計的第二檢查點,**不是重工**:規劃時的判斷憑的是「意圖」,PR 時的判斷憑的是「證據」。如果實作把一個原本規劃為 leaf 的變更漂移成了 core(它現在動到了原本不該動的共用模組),PR 時的分類**必須抓到它**——較嚴者勝,這條規則在時間軸上也成立。

一致性來自「共用同一套準則」,而不是「只算一次答案」。PR 時的分類若**升級**(leaf → core)= 一個要浮上檯面的發現,絕不無聲蓋掉;若要**降級**(core → leaf)則需人簽核。

## 邊界案例

### 「它是 leaf,但順帶碰到了 core 程式碼」

如果變更**讀**了 core 程式碼(import、呼叫)但沒有**改** core 程式碼,它仍然是 leaf。分類看的是「哪些行改了」,不是「哪些行存在」。

### 「它是腳本 / 一次性 / migration」

即使它讀 core 資料,這些仍是 leaf——故障被單次執行框住。但:如果腳本**寫入 production 資料**,那麼腳本的**驗證**需要 core 級(別只靠抽查就信任它)。

### 「程式碼在 leaf 路徑,但故障模式是全系統的」

例如:一個不小心寫入每一筆使用者記錄的腳本。依**故障成本**重新分類,不看檔案位置。這個框架的重點是**故障侷限(failure containment)**。

### 「隨著模型變強,這條邊界會一直移動」

沒錯。隨著模型變得更可靠,今天的 core 可能是明天的 leaf。技術主管應該每季重新檢視 leaf/core 邊界。但**別憑期望移線——要憑實績移線**。

## 常見的誤分類

- **一個新端點是 leaf,即使「這個 API」是 core。** 單一新端點是局部的。修改「所有端點都用的那個框架」才是 core。
- **一個新 UI 頁面是 leaf。** 修改 design system 或共用元件庫才是 core。
- **一個新報表是 leaf。** 修改 BI 基礎設施 / 資料倉儲 schema 才是 core。
- **一個 bug fix 是「它所在的那個檔案是什麼」就是什麼。** 別自動把 bug fix 歸為低風險;如果 bug 在 core,就把修正當成 core 處理。

## 停止條件

當使用者拿到清楚的分類 + 對應的建議做法時,結束這支 skill。把分類帶進 `plan-feature` 或 `pr-prepare`,在那裡設定 review 嚴謹度——這支 skill 是那些工作流應該遵循的 leaf/core 定義的**權威擁有者**。

## 本關完成後

<!-- DELIBERATE DELTA(vs upstream appleboy/skills):回呼 sdlc 改為「進度檔存在才接」的條件式,單獨用一支不被逼進完整流程。刻意為之——別還原成無條件 REQUIRED 版。 -->

**在跑完整 agent-sdlc 生命週期?** 若有 `docs/sdlc/<feature>-sdlc-progress.md`(或你刻意啟動整條 SOP 鏈),呼叫 `/agent-sdlc:sdlc`——它勾掉本關並回報確切的 ⏭ 下一步。只導航,不替你跑下一關。

**單獨用這支 skill?** 那你已完成——**不要**呼叫 `/agent-sdlc:sdlc`(沒有進度檔可更新)。若想繼續,通常接的下一步是 **gate 4 寫 code,然後內建 `/simplify` → `/security-review` → `/code-review max -fix`**。
