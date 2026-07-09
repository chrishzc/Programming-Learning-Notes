# GitHub Repository Ruleset (分支保護規則) 設定指引

> [!ABSTRACT]
> 本篇筆記整理了 GitHub Repository Ruleset (分支保護規則集) 的核心功能設定。包含基礎設定、Bypass 特權例外名單、Target Branches 作用範圍、四大類 Branch Rules 規則詳解以及三種常見的合併方式 (Merge Methods) 的對比。

---

## 一、 基礎與 Bypass (特權例外) 設定

### 1. 規則集基礎設定
- **Ruleset Name**：設定規則集的名稱（例如：`main-branch-protection`）。
- **Enforcement Status**：
  - **Active**：開啟並強制執行規則。
  - **Evaluate**：評估模式（僅記錄違反規則的行為，不進行阻斷，適合測試新規則）。
  - **Disabled**：關閉規則。

### 2. Bypass (特權豁免) 對象
您可以設定特定對象繞過分支保護規則的限制，主要分為三種維度：
- **Role (角色 - 認職不認人)**：
  - 例如 `Repository admin`（管理員）或 `Maintainer`（維護者）。只要使用者具備該角色，即自動享有豁免權。
- **Team (團隊 - 協作與火急修復)**：
  - 例如指定的資深工程小組（`@senior-developers`）。當遇到緊急線上 Bug 需要緊急推代碼時，可跳過標準的 PR 審查流程直接 Push。
- **App / Agent (自動化機器人)**：
  - 例如 `github-actions` 或 `dependabot`。允許掃地機器人在半夜自動更新套件、修補安全性漏洞，而不需要人工介入點選 Approve。

### 3. 特權豁免模式 (Bypass Mode)
- **Always allow (總是允許)**：該對象完全不受任何規則限制，可直接 Push 或 Merge。
- **For pull requests only (僅限 PR 時免受限制)**：該對象平時仍需乖乖建立 PR，但在審查 PR 時可以強行點擊合併，跳過「必須有審查核准」或「自動測試通過」的限制。

---

## 二、 Target Branches (規則套用範圍)

定義該 Ruleset 要作用於哪些分支：
- **Default branch**：僅作用於預設分支（通常為 `main` 或 `master`）。
- **All branches**：作用於專案內所有分支。
- **Include by pattern**：使用通配符匹配特定分支。若要套用至除 `main` 之外的其他特定分支（例如 `release/*`），可選擇此項並輸入 Pattern。

---

## 三、 Branch Rules (分支細部規則)

這些規則主要可以細分為以下四大類別：

### 第一類：基礎進出管制 (最嚴格的封鎖)
1. **Restrict creations (限制建立)**：
   - **說明**：限制哪些使用者可以建立與 Pattern 匹配的分支。避免一般開發者隨意建立像 `main` 或 `release-v1` 等關鍵分支。
2. **Restrict updates (限制更新)**：
   - **說明**：限制哪些使用者可以直接推送代碼（Push）到該分支。例如在「封測」或準備發布時，對該分支進行鎖定，僅允許管理員推 Bugfix。
3. **Restrict deletions (限制刪除)**：
   - **說明**：強制禁止刪除此分支。防範成員手滑或意外刪除重要的主分支（`main`）。

### 第二類：檔案品質與規格檢查 (代碼健康)
4. **Require linear history (需要線性歷史)**：
   - **說明**：禁止在主分支中產生交錯的分支合併地鐵圖。強迫合併時必須使用 `Squash` 或 `Rebase`，讓主軸線的歷史紀錄保持一條乾淨的直線。
5. **Require deployments to succeed (需要部署成功)**：
   - **說明**：程式碼合進 `main` 之前，必須先在測試環境（如 Vercel, AWS）成功部署。如果部署失敗（如畫面打不開），將直接攔截合併。
6. **Require signed commits (需要簽名提交)**：
   - **說明**：強制所有 Commit 都必須附有個人的數位簽章（如 GPG 金鑰）。防範駭客偽造他人 Email 來推送惡意代碼。

### 第三類：進階審查機制 (PR 與 Code Review 細部設定)
開啟 **Require a pull request before merging (合併前需要 PR)** 後的延伸設定：
7. **Dismiss stale pull request approvals when new commits are pushed (新推送時撤銷舊核准)**：
   - **說明**：若有人通過了你的 PR 審查，但你隨後又補推了新的 Commit，系統會自動撤銷先前的 Approve，強迫審查者重新過濾新修改的內容。
8. **Require review from specific teams (需要特定團隊審查)**：
   - **說明**：如果程式碼動到了特定核心功能，強迫指派特定團隊（如 `@security-team`）的成員審查通過才可合併。
9. **Require review from Code Owners (需要 Code Owners 審查)**：
   - **說明**：透過專案內設定的 `CODEOWNERS` 檔案，當修改特定資料夾（如 `/db/`）時，系統會自動強制標記資料庫負責人來審查。
10. **Require approval of the most recent reviewable push (最新推送核准)**：
    - **說明**：確保最後一次修改/推送程式碼的人，不能與最後按 Approve 的人是同一個人，落實第三方雙重把關。
11. **Require conversation resolution before merging (合併前需解決所有對話)**：
    - **說明**：PR 中的所有 Code Review 留言討論（如指出語法錯誤或潛在 Bug），必須全部被點選「Resolve conversation」標記解決後，合併按鈕才會解鎖。

### 第四類：自動化檢測與工具把關 (CI/CD 整合)
12. **Require status checks to pass (需要狀態檢查通過)**：
    - **說明**：結合 GitHub Actions (CI)。當提出 PR 時，系統自動執行單元測試與語法檢查，測試失敗則無法合併。
13. **Block force pushes (阻止強制推送)**：
    - **說明**：阻止使用 `git push -f` 暴力指令覆蓋遠端歷史紀錄，防範多人協作時的代碼覆蓋悲劇。
14. **Require code scanning results (需要程式碼安全掃描)**：
    - **說明**：結合 CodeQL 等工具。如果掃描出代碼中硬編碼了密碼/金鑰或有明顯漏洞，將直接拉響警報並攔截。
15. **Require code quality results / Restrict code coverage (品質與覆蓋率要求)**：
    - **說明**：限制程式碼複雜度，且測試覆蓋率必須達到設定門檻（例如 80%）才允許合入。
16. **Automatically request Copilot code review (自動 Copilot 審查)**：
    - **說明**：自動請求 AI 助理（Copilot）針對 PR 提供初步的代碼審查與挑錯建議。

---

## 四、 允許的合併方式對比 (Allowed Merge Methods)

當我們將開發分支（`feature`，含提交 A, B, C）合併到主分支（`main`，目前狀態為 D ➔ E）時，GitHub 支援以下三種合併方式：

| 合併方式 | 合併後的歷史圖表 | 特點與適用場景 |
| :--- | :--- | :--- |
| **建立合併提交<br>(Merge Commit)** | `main` 軸線：**D ➔ E ➔ M** (M 指向 E 與 C)<br>呈現非線性（類似地鐵分岔圖） | - 保留所有細節（A、B、C 提交會完整保留）。<br>- 歷史會出現一個專用的合併提交（M）。<br>- **適用**：需要追蹤完整開發脈絡與合併歷史時。 |
| **壓扁並合併<br>(Squash and Merge)** | `main` 軸線：**D ➔ E ➔ S**<br>呈現乾淨的一直線 | - 將 A、B、C 合併為一個單一提交（S）。<br>- 原始細碎的 commit 歷史會被隱藏。<br>- **適用**：開發過程有許多雜亂或未完工的 WIP commit，希望主分支保持極簡乾淨時。 |
| **變基並合併<br>(Rebase and Merge)** | `main` 軸線：**D ➔ E ➔ A' ➔ B' ➔ C'**<br>呈現線性但保留每個步驟 | - 複製 A, B, C 並重新排列在主分支頂端。<br>- 每個 commit 的 SHA 值會改變。<br>- **適用**：希望保留完整的開發歷史，又想維持線性歷史紀錄時。 |