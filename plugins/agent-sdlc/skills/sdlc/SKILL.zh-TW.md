---
name: sdlc
description: 更新目前功能的 SDLC 進度檔並回報下一步。當使用者問「下一步是什麼」「我在 SDLC 哪一關」「更新 SDLC 進度」「我做完某關了」,或其他 agent-sdlc skill 收尾回呼時使用。讀每個 PR 一支的進度檔 + git 狀態,勾選已完成關卡、刷新「目前→下一步」區塊,回報 📍目前/⏭下一步/卡點。只導航——不跑下一關、不核可人審閘、不碰版控。
---

<!-- 中文鏡像:非 active(不被當 skill 觸發),與同目錄 SKILL.md 同步。權威版是英文 SKILL.md。 -->

# SDLC 導航器 — 進度檔更新器

你維護**每個 PR 一支的進度檔**,告訴使用者下一步。你是**導航員,不是駕駛**:讀狀態、更新檔案、報下一步。你不執行下一關、不簽核人審閘、不 commit。

完整 12 步鏈在 pack 的 `SOP.md`。進度檔是**單一 PR** 走這條鏈的單一真相來源。

⛔ **一個 PR 一支,不要一個多 PR 的 feature 共用一支。** 你**每一關**都要讀它,共用的會無上限地長:
實測一支 4-PR 的共用檔到了 **845 行 / 80 KB**,裡面還躺著幾天前就合併的 PR 的逐關細節。
下一個 PR 真正需要的東西(feature 層 gate 0 產出、跨 PR 決策、未關項目)放 **plan**,進度檔指回去。

## 何時執行

- 使用者顯式打 `/agent-sdlc:sdlc`。
- 其他 `agent-sdlc` skill 完成該關後回呼(其結尾「本關完成後」步驟)。
- 使用者說「下一步」「更新進度」「我剛做完 code-review」等。

## 第 1 步 — 定位進度檔

- 在目前開發 repo 找 `docs/sdlc/<feature>-<pr>-sdlc-progress.md`。
- 有多支時,挑與當前工作/branch 相符者;不明確就問使用者是哪個 PR。
- 找不到 → 走下面「初始化」。

## 第 2 步 — 讀目前狀態

- 讀進度檔。
- 讀 git 狀態佐證:目前 branch、staged/unstaged、`git log --oneline -n 5`。
- 留意本次對話剛做了什麼——這是追認**內建關卡**(5 `/simplify`、6 `/security-review`、7 `/code-review max -fix`)的方式;它們是 Claude Code 內建、無法自己回呼。在**它們之後第一個跑到的 pack 關卡**一併勾選 5/6/7——依執行順序 `… → 7 → 9 → 10 → Δ → 8 → 11`,那是第 **9** 關 `pr-prepare`,不是第 8 關。
- ⚠️ **順序看 `SOP.md`,不要看編號。** 第 8 關(commit)排在 9、10、Δ **之後**;卡在第 7 關的 PR,⏭ 是 `/agent-sdlc:pr-prepare`,不是 commit-message。

## 第 3 步 — 更新檔案

- 勾選完成關卡:`- [ ]` → `- [x]`;進行中那關標 🔄。
- 刷新頂部 **📍 目前位置 → ⏭ 下一步 → 卡點**——這是使用者看「下一步」的落腳點。「下一步」要寫成明確動作(如 `` `/simplify` ``、`/agent-sdlc:commit-message`、「人審 core diff」)。
- 補事實到 **紀錄 / 連結**:commit hash、PR URL、external-review 摘要、以及任何**決策/偏移**(如規劃 leaf 但真實 diff 漂成 core——記下升級)。
- **不要編造完成**。git 狀態不佐證的關卡就別勾,並說明。
- **Δ 關(`/agent-sdlc:delta-enumerate`)要有自己的一段,不能只勾一個框**:母體是哪幾行、怎麼導出的、枚舉了什麼、幾條發現、有沒有達門檻(≥1 條會走到危險路徑或推翻已寫下的保證 → 升級成完整的第二次 gate 7)、**Δ 修了什麼與留給人審什麼(兩份分開的清單)**、以及 Δ′ 的結果。第 9/10 關零採納的話 Δ 母體是空的——記「Δ 跳過,母體為空」。⚠️「Δ 跑了但沒發現」和「Δ 沒跑」在檔案上必須長得不一樣。
- **第 5/6 關的結論在第 7 關放行前都是暫定的。** 勾 5 或 6 的時候,把它們的發現記成*待第 7 關確認*——先不要編定案號、先不要寫進 plan 的決策區。實測:第 7 關推翻過一條已回報的 gate-5 結論與一條 gate-6 結論,兩條都是「把一次量測當成規格」。

## 第 4 步 — 回報使用者

回報三行:
- **📍 目前**:功能在哪一關。
- **⏭ 下一步**:確切的下一個動作/指令。
- **卡點**:無,或描述。

若使用者想跳關(如第 10 關 external-ai-review 沒勾就要 merge),明白指出——提醒而非硬擋(pack 不自簡化關卡,但你是導航員不是門衛)。

## 初始化(尚無進度檔)

1. 複製 bundled 模板到開發 repo:從本 plugin 的 `templates/sdlc-progress.template.md`(在 `skills/` 上一層,即 `../../templates/sdlc-progress.template.md`)→ `docs/sdlc/<feature>-<pr>-sdlc-progress.md`。
2. 填頭部:feature 名、開題日、規劃鏈(見下)、base/工作 branch、已知的 leaf/core 裁決。

   **規劃這關要「串起來」,不是二選一。** 這兩支工具在**不同高度**、可以疊,不是替代關係:

   | 階段 | 工具 | 產出 |
   |:--|:--|:--|
   | a. 釐清 + 比方案 | `superpowers:brainstorming`(**有裝才跑**) | 經核可的 design/spec;強制 2–3 個方案比 trade-off、一次只問一題 |
   | b. **決策層(必跑)** | `/agent-sdlc:plan-feature` | 目標、may-modify + **must-not-touch**、驗證策略、Mermaid、風險、**回滾** |
   | c. 實作層 | `superpowers:writing-plans`(**有裝**,且工作 code 量大到需要逐 task 拆步驟才跑) | 逐 task 的 red-then-green 步驟 |

   **b 永遠不可跳過**——只有它產出 must-not-touch 清單、回滾方案、以及 planning-exit checklist 要的東西。
   a 與 c 優雅降級:沒裝 superpowers 就略過,並在進度檔註明。

   記下**實際跑了哪幾段**,例如 `brainstorming + plan-feature`。
3. 確認開發 repo 的 `docs/sdlc/` 已 gitignore(它是工作 scratch,同 mem-tmp 性質——絕不進版控)。若 `.gitignore` 未涵蓋,提示使用者加 `docs/sdlc/`。
4. 然後跑第 2–4 步。

## 結案(這個 PR 已 merge — 第 11 關完成)

- 確認第 11 關(人審合併)完成、PR 已 merge。
- 回報「此 PR 的 SDLC 完成」。
- **先把還活著的東西移進 plan**(未關項目、跨 PR 決策、下一個 PR 的硬輸入),然後**刪除進度檔**。不歸檔——commit 與 PR 就是永久紀錄,沒有歸檔步驟。
- 同 feature 還有後續 PR 的話,下一個 PR **開一支新檔**,不要把這支撿回來用。

## 硬邊界

- ❌ **不**跑下一關(不叫 `/simplify`、不 commit、不開 PR)。你只**說**下一步是什麼。
- ❌ **不**核可任何 🚦 人審閘——核可永遠是人的。
- ❌ **不** `git add`/`git commit` 進度檔,也不 commit 任何東西。進度檔是 gitignored scratch。
- ❌ **不**改其他 skill 的 SDLC 把關邏輯。

## 與 pack 其餘部分的關係

- **7 支** pack skill(第 0/1/3/8/9/10/**Δ** 關)完成各自關卡、**且進度檔存在時**(即正在跑完整生命週期,而非單獨用一支)回呼你——這就是進度檔免使用者操心仍保持最新的原因。
- 內建關卡 5/6/7 在下一個 pack 關卡追認——依執行順序是第 **9** 關 `pr-prepare`,不是第 8 關。進度檔把該事實帶過任何 `/compact`。
- ⚠️ **第 0 關是 feature 層,不是 PR 層**:同一 feature 的第二個以後的 PR 不重跑 `prompt-audit`,進度檔寫「不重跑,見 \<哪裡\>」。不要當成被跳過的關卡回報。
