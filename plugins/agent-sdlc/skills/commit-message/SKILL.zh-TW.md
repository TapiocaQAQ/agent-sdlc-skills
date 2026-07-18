---
name: commit-message
description: 分析 staged 的 git 變更,生成一則 conventional commit 訊息。當使用者想從目前 staged 的 diff 建立、撰寫或生成 git commit 訊息時使用。觸發詞包括「commit」「commit message」「commit msg」「寫 commit」「提交訊息」「generate commit」,或使用者剛改完程式碼想 commit 時。使用者執行 /agent-sdlc:commit-message 也觸發。
---

# 生成 Commit 訊息(中文鏡像)

> 📌 本檔是 `SKILL.md`(英文權威版)的中文鏡像,給中文語系維護者/讀者閱讀用。**Claude 實際載入/觸發的是英文 `SKILL.md`;本檔不會被當 skill 觸發。** 兩版內容一一對應,**每次改動兩版必須一起更新**。
>
> ⚠️ **與上游 appleboy/skills 的刻意偏離**(兩處,非抄漏):
> 1. **Step 6 加了「人審閘」**:appleboy 原作是「`git commit` 直接跑、不問確認」(呼叫即授權);本 pack 刻意改成 **commit 前先把完整訊息給使用者看、批准才送出**。這是把關卡**加嚴**,不是簡化。
> 2. **強制 `Co-Authored-By: Claude <noreply@anthropic.com>` 尾行**:appleboy 原作的 commit 格式沒有這行;本 pack 依全域協作規範,強制每則 commit 末端都掛此 trailer(用 model-agnostic 的純 `Claude` 形,不寫死型號,避免長壽 skill 過期)。

依結構化的 prompt 流水線,從 staged 的 git 變更生成一則 conventional commit 訊息。

## 步驟

### 1. Stage 變更並取得 diff

只 stage 本次工作階段實際改動的檔案。**不要盲目跑 `git add -A` 或 `git add .`**——挑跟任務相關的特定檔案。若不確定該 stage 哪些,把 `git status` 丟給使用者,讓他決定。

接著同時取得總覽與完整 diff:

```bash
git diff --staged --stat
git diff --staged
```

若這之後 diff 是空的,告知使用者沒有 staged 變更並停止。

若 diff 非常大(例如超過 2000 行),聚焦在 `--stat` 輸出與最重要的 hunk。對於變更過於龐大的檔案,註明那些 diff 已略過,並根據 stat 總覽與可得脈絡來寫摘要。

若 staged 變更橫跨多個不相關的模組、或跨不同關注點超過 10 個檔案,在往下做之前先建議拆成多個聚焦的 commit。

### 2. 檢查既有 commit 風格

快速掃一下近期 commit,對齊 repo 的慣例:

```bash
git log --oneline -20
```

若 repo 已遵循一致風格(例如特定的 scope 命名、前綴偏好、或語言),把生成的訊息調整成相符。本 skill 裡的慣例只是**預設值**——當 repo 既有樣式不同時,以 repo 為準。

### 3. 分析 diff

產出變更的條列式摘要。遵守這些規則:

- 以 `+` 開頭的行代表新增,`-` 代表刪除。兩者皆非的是 context。
- 每則摘要註解都寫成 `-` 開頭的條列。
- 註解裡不要包含檔名。
- 不要使用 `[` 或 `]` 字元。
- 不要放從程式碼複製來的註解。
- 只寫最重要的註解。拿不定主意時,寧可寫少一點。
- 可讀性最優先。

參考用的摘要註解範例(不要逐字照抄):

```
- Increase the number of returned recordings from 10 to 100
- Correct a typo in the GitHub Action name
- Relocate the octokit initialization to a separate file
- Implement an OpenAI API endpoint for completions
```

### 4. 生成 commit 標題

從摘要寫出單行的 commit 標題:

- 用祈使句,遵循 kernel git commit 風格指南。
- 寫一個抓住「單一具體主題」的高層次標題。
- 不要重複檔案摘要或逐項列出個別變更。
- 標題精簡——目標 50 字以內(含前綴與 scope 的完整 header 應維持在 72 字以內)。
- 首字小寫。
- 去掉結尾的句點。

### 5. 決定 prefix 與 scope

**Prefix** —— 根據摘要,恰好選一個標籤:

- `build`:影響建置系統或外部依賴的變更(範例 scope:gulp、broccoli、npm)
- `chore`:更新函式庫、著作權、或其他 repo 設定,包含更新依賴
- `ci`:對 CI 設定檔與腳本的變更(範例 scope:Travis、Circle、GitHub Actions)
- `docs`:非程式碼的變更,例如修錯字或新增文件
- `feat`:為 codebase 引入一個新功能
- `fix`:修補 codebase 裡的一個 bug
- `perf`:改善效能的程式碼變更
- `refactor`:既不修 bug 也不加功能的程式碼變更
- `style`:不影響程式碼意義的變更(空白、格式等)
- `test`:補上缺少的測試或修正既有測試

**破壞性變更** —— 若 diff 移除或改名了公開 API、以破壞相容的方式改了函式簽章、或做了任何其他向後不相容的變更,在 prefix 之後(有 scope 則在 scope 之後)加 `!`。例如:`feat!:` 或 `feat(api)!:`。此外,在 commit body 加一行 `BREAKING CHANGE:`,描述改了什麼、如何遷移。

**Scope** —— 從改動的檔案判斷模組或套件的 scope:

- 看 diff 裡的檔案路徑,判斷影響到哪個模組/套件/元件。
- 若所有變更都在單一模組/套件/目錄內,就用它當 scope(例如 `model`、`git`、`prompt`、`cmd`、`provider`)。
- 用「最具體的共同目錄或套件名」。例如只在 `provider/openai/` 的變更用 `openai`,而不是 `provider`。
- 若變更橫跨多個模組,挑對變更目的最核心的那個。
- scope 要短——單一小寫字。
- scope **建議但可省**——若判不出清楚的 scope(例如變更碰到很多不相關的區域),就省略它,改用 `<prefix>: <title>` 格式。

### 6. 先給使用者檢查,批准後才建立 commit

<!-- 與上游 appleboy/skills 的刻意偏離:原作 Step 6 結尾是
     「`git commit` 直接跑、不問確認」(呼叫即授權)。
     本 pack 刻意加一道人審閘:先把完整訊息秀出來,使用者批准後才 commit。
     這是把關卡加嚴,不是抄漏。 -->

把 commit 訊息格式化為:

```
<prefix>(<scope>): <title>

<summary>

Co-Authored-By: Claude <noreply@anthropic.com>
```

或無 scope:

```
<prefix>: <title>

<summary>

Co-Authored-By: Claude <noreply@anthropic.com>
```

若有破壞性變更,把 `BREAKING CHANGE:` footer 放在 trailer 之上:

```
<prefix>(<scope>)!: <title>

<summary>

BREAKING CHANGE: <描述破壞了什麼與遷移路徑>

Co-Authored-By: Claude <noreply@anthropic.com>
```

`Co-Authored-By: Claude <noreply@anthropic.com>` 這個 trailer 是**強制**的——永遠把它當作每則 commit 訊息的**最後一行**(有 `BREAKING CHANGE:` footer 時,放在該 footer 之後)。

**commit 前先審。** 把完整的提議 commit 訊息秀給使用者看——完整的 header、摘要、trailer,一字不差就是即將 commit 的樣子——並等待他批准。若使用者要求修改,改好後再秀一次更新版。**在使用者批准前,不要跑 `git commit`。**

一旦批准,用 HEREDOC 把多行訊息傳給 `git commit`:

```bash
git commit -m "$(cat <<'EOF'
<prefix>(<scope>): <title>

<summary>

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

### 7. 處理使用者的附加請求

若使用者的請求包含 commit 以外的額外動作(例如「commit and push」「commit 然後推上去」「commit and create PR」),在 commit 成功之後才執行那些動作。常見樣式:

- **push**:commit 後跑 `git push`
- **push and PR**:push,然後建立一個 pull request
- **tag**:commit 後建立一個 git tag

只做使用者明確要求的動作。

## 本關完成後

<!-- DELIBERATE DELTA(vs upstream appleboy/skills):回呼 sdlc 改為「進度檔存在才接」的條件式,單獨用一支不被逼進完整流程。刻意為之——別還原成無條件 REQUIRED 版。 -->

**在跑完整 agent-sdlc 生命週期?** 若有 `docs/sdlc/<feature>-sdlc-progress.md`(或你刻意啟動整條 SOP 鏈),呼叫 `/agent-sdlc:sdlc`——它勾掉本關並回報確切的 ⏭ 下一步。只導航,不替你跑下一關。

**單獨用這支 skill?** 那你已完成——**不要**呼叫 `/agent-sdlc:sdlc`(沒有進度檔可更新)。若想繼續,通常接的下一步是 **gate 9 `/agent-sdlc:pr-prepare`**。
