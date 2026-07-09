# Git 常用指令與實務指引

> [!ABSTRACT]
> 本章整理版本控制工具 Git 的核心概念與常用命令速查表，涵蓋 SSH 金鑰設定、環境初始化、本地版控工作流、分支管理、遠端協作以及版本回退等常用實務操作。

---

## 零、 SSH 金鑰設定（只需做一次）

SSH 金鑰讓你的電腦與 GitHub 之間能進行加密認證，設定好後 Push / Pull 就不需要每次輸入密碼。

### 1. 生成 SSH 金鑰
```bash
# 生成 Ed25519 金鑰（推薦）；-C 後面填你的 GitHub 帳號信箱
ssh-keygen -t ed25519 -C "your_email@example.com"

# 一路按 Enter 採用預設路徑與無密碼保護即可
# 金鑰檔案會產生在 ~/.ssh/id_ed25519（私鑰）與 ~/.ssh/id_ed25519.pub（公鑰）
```

### 2. 將公鑰加入 GitHub
```bash
# 複製公鑰內容（Windows）
type %USERPROFILE%\.ssh\id_ed25519.pub | clip

# 複製公鑰內容（Mac / Linux）
cat ~/.ssh/id_ed25519.pub | pbcopy
```
接著進入 GitHub → 右上角頭像 → **Settings** → **SSH and GPG keys** → **New SSH key**，貼上公鑰內容並儲存。

### 3. 驗證連線是否成功
```bash
ssh -T git@github.com
# 看到 "Hi username! You've successfully authenticated..." 即代表設定成功
```

> [!IMPORTANT]
> Clone 時請選擇 **SSH** 格式的網址（`git@github.com:user/repo.git`）而非 HTTPS，才能使用金鑰認證。

---

## 一、 Git 初始化與基礎設定

在開始使用 Git 進行版本控制前，需進行基本的使用者身分設定。

### 1. 使用者身分設定
```bash
# 設定全域使用者名稱
git config --global user.name "YourName"

# 設定全域電子郵件
git config --global user.email "your_email@example.com"

# 查看當前所有 Git 設定
git config --list
```

### 2. 建立與複製倉庫
```bash
# 在當前目錄初始化一個新的本地 Git 倉庫（建立 .git 資料夾）
git init

# 複製一個遠端的 Git 倉庫到本地
git clone <遠端倉庫URL>
```

---

## 二、 本地版控基礎工作流

本地版控流程通常在：**工作區 (Working Directory)** ➔ **暫存區 (Staging Area)** ➔ **本地倉庫 (Local Repository)** 之間進行。

```mermaid
flowchart LR
    WD["工作區<br>(Working Directory)"]
    SA["暫存區<br>(Staging Area)"]
    LR["本地倉庫<br>(Local Repository)"]
    RR["遠端倉庫<br>(Remote Repository)"]

    %% Git Commands Flow
    WD ==>|"git add"| SA
    SA ==>|"git commit"| LR
    LR ==>|"git push"| RR

    %% Git Undo Flow
    SA -.->|"git restore --staged"| WD
    LR -.->|"git reset --soft"| SA
    LR -.->|"git reset --hard"| WD
    WD -.->|"git restore (放棄修改)"| WD

    %% Git Sync Flow
    RR ==>|"git pull / git clone"| WD
```

| 指令                          | 說明                               |
| :---------------------------- | :--------------------------------- |
| `git status`                  | 檢查當前工作區與暫存區的檔案狀態（新增、修改、刪除）       |
| `git add <file>`              | 將指定檔案的變更加入暫存區                    |
| `git add .`                   | ==將當前目錄下所有修改與新增的檔案一次性加入暫存區==     |
| `git commit -m "msg"`         | ==將暫存區的變更提交到本地倉庫，並附上提交訊息==       |
| `git commit -am "msg"`        | 直接將所有「已追蹤並修改」的檔案加入暫存並提交（不適用於新檔案） |
| `git diff`                    | 查看工作區與暫存區之間的代碼差異                 |
| `git log`                     | 查看詳細歷史提交紀錄                       |
| `git log --oneline --graph`   | 以單行與圖形化線條簡潔呈現分支歷史紀錄              |

### 1. git commit 指令進階細節

> [!TIP]
> **直接輸入 `git commit`（未帶 `-m` 參數）的行為與操作**
>
> 若在終端機中直接執行 `git commit`，Git 會自動開啟預設的終端機文字編輯器（在 Linux/Mac 通常為 Vim 或 Nano，Windows 通常為 Vim）要求您輸入 Commit 訊息：
>
> 1. **編輯 Commit 訊息**：
>    - **Vim 操作**：進入時為「命令模式」，請按鍵盤上的 **`i`** 進入「編輯模式（INSERT）」，此時可開始輸入文字。建議格式為：第一行寫簡短標題，空一行，第三行起寫詳細的修改內容說明。
>    - **儲存並離開**：輸入完畢後，按 **`Esc`** 退出編輯模式回到命令模式，接著輸入 **`:wq`** 並按下 **`Enter`** 即可完成存檔並提交。
>    - **取消本次提交**：若不想提交，按 **`Esc`** 回到命令模式，輸入 **`:q!`** 並按下 **`Enter`**（或是不輸入任何內容直接存檔離開），Git 會顯示 `Aborting commit due to empty commit message` 並取消本次提交。
> 2. **更換預設編輯器**（例如改用 VS Code）：
>    ```bash
>    git config --global core.editor "code --wait"
>    ```

---

## 三、 分支管理與切換 (Branching)

分支是 Git 的核心功能，讓開發者能平行開發不同功能而互不干擾。

| 指令                       | 說明                                         |
| :------------------------- | :------------------------------------------- |
| `git branch`               | 列出本地所有分支，並標註當前所在分支                         |
| `git branch -a`            | 列出本地與遠端的所有分支                               |
| `git branch <name>`        | ==建立一個名為 `<name>` 的新分支（但不切換）==             |
| `git checkout <name>`      | ==切換至指定分支（新版本推薦使用 `git switch <name>`）==   |
| `git checkout -b <name>`   | 建立新分支並立即切換過去（新版本推薦 `git switch -c <name>`） |
| `git merge <name>`         | 將指定的 `<name>` 分支代碼合併到「當前所在分支」              |
| `git branch -d <name>`     | 刪除已合併的本地分支                                 |
| `git branch -D <name>`     | 強制刪除未合併的本地分支                               |

---

## 四、 遠端倉庫協作 (Remote)

用於本地倉庫與 GitHub / GitLab 等遠端伺服器的同步協作。

| 指令                            | 說明                                             |
| :------------------------------ | :----------------------------------------------- |
| `git remote -v`                 | 查看當前連結的遠端倉庫地址與名稱（通常預設為 `origin`）               |
| `git remote add origin <URL>`   | 將本地倉庫連結至指定的遠端倉庫地址                              |
| `git push -u origin <branch>`   | 將本地分支推送到遠端，並設定預設追蹤（後續只需打 `git push`）           |
| `git push`                      | ==將本地提交推送到已綁定追蹤的遠端分支==                         |
| `git fetch`                     | ==從遠端拉取最新的變更紀錄，但不自動合併到本地工作分支==                 |
| `git pull`                      | ==從遠端拉取最新代碼並「自動合併」至當前本地分支（相當於 fetch + merge）== |

### 1. git push 參數與推動規則

當我們想將本地變更推送至遠端伺服器時，主要有以下三種常用指令形式，其運作規則如下：

#### 💡 `git push origin <分支名稱>` (例如 `git push origin main`)
- **行為**：明確指定要將本地變更推送到遠端倉庫（`origin`）的特定分支（`main`）。
- **特點**：這是一次性的推動指令，**不會**在本地與遠端分支之間建立長期的追蹤關係。如果下次直接輸入 `git push`，Git 依然不知道預設要推送到哪裡。

#### 💡 `git push -u origin <分支名稱>` (例如 `git push -u origin main`)
- **行為**：推送的同時，使用 `-u`（或 `--set-upstream`）在本地分支與遠端的 `origin/main` 之間**建立預設追蹤（Upstream）連結**。
- **適用場景**：**第一次將新分支推送到遠端時使用**。設定好後，未來在此分支下只需輸入極簡的 `git push` 或 `git pull`，Git 就會自動對應到遠端的 `origin/main`，無須再輸入完整路徑。

#### 💡 `git push` (無任何參數)
- **行為**：將當前分支的變更推送至其對應的「已追蹤遠端分支」。
- **限制**：只有在當前分支已使用 `-u` 建立過追蹤連結時才有效。若未設定追蹤即直接執行 `git push`，Git 會報錯並提示需要指定遠端與分支，或是使用 `--set-upstream`。

---

## 五、 復原與衝突解決 (Undo & Conflict)

當發生錯誤或程式碼衝突時，使用以下指令進行回退或修正。

### 1. 修改撤銷與復原

| 指令                             | 說明                                              |
| :------------------------------- | :------------------------------------------------ |
| `git restore <file>`             | 放棄工作區中未暫存的修改（還原到最近一次 commit 或暫存狀態）              |
| `git restore --staged <file>`    | 將檔案從暫存區移回工作區（等同於取消 `git add`）                   |
| `git commit --amend`             | 修改最後一次的 commit 訊息或追加檔案至最近一次 commit              |
| `git reset --soft HEAD~1`        | 撤銷最近一次 commit，但保留工作區與暫存區的修改代碼（安全回退）             |
| `git reset --hard HEAD~1`        | 撤銷最近一次 commit，且**徹底丟棄**該次 commit 以後的所有代碼修改      |
| `git reset --hard <commit_id>`   | 將本地倉庫強制回退至指定的歷史 commit 版本                       |
| `git revert <commit_id>`         | 以新增一個 commit 的方式來撤銷指定 commit 的修改（適用於已 push 的歷史） |

### 2. 衝突（Conflict）是怎麼發生的？

當 Git 無法自動判斷「要保留哪一方的修改」時就會產生衝突。常見的三種觸發情境：

| 情境 | 觸發原因 |
| :--- | :--- |
| **`git merge`** | 兩個分支各自修改了同一個檔案的同一行 |
| **`git pull`** | 遠端有新 commit，本地也有新 commit，且兩者修改了相同位置 |
| **`git push` 被拒絕** | 本地分支落後遠端（`push` 本身不產生衝突，但 Git 會拒絕推送，須先 `pull` 整合後再 `push`） |

---

### 3. 情境一：`git merge` 產生衝突

```bash
git checkout main
git merge feat/login   # ← 若兩分支修改了同一行，此時觸發衝突
```

Git 會在衝突的檔案中插入標記：

```text
<<<<<<< HEAD           ← 當前分支（main）的版本
目前 main 的程式碼
=======                ← 分隔線
feat/login 的程式碼
>>>>>>> feat/login     ← 要被合併分支的版本
```

**解決步驟：**
1. 打開所有衝突的檔案，手動決定保留哪一方（或兩者都保留），並刪除 `<<<<<<<`、`=======`、`>>>>>>>` 這三行標記。
2. 存檔後，將解決完的檔案加入暫存並提交：
   ```bash
   git add <衝突的檔案>
   git commit -m "fix: resolve merge conflict in <檔名>"
   ```
3. 若要放棄此次合併、回到 merge 前的狀態：
   ```bash
   git merge --abort
   ```

---

### 4. 情境二：`git pull` 產生衝突

`git pull` 等同於 `git fetch` + `git merge`，所以若遠端與本地都有新 commit，就可能在 merge 階段觸發衝突。

```bash
git pull    # ← 若遠端 main 與本地 main 修改了相同位置，觸發衝突
```

衝突標記與解決方式和情境一的 `git merge` 完全相同。

> [!TIP]
> **用 `git pull --rebase` 取代 `git pull` 可保持線性歷史**
>
> `git pull --rebase` 改用 Rebase 策略，可以避免多餘的「合併 commit」讓 log 更整潔。
> 發生衝突時解決方式略有不同：
> ```bash
> # 解決完衝突後（不需要 git commit）
> git add <衝突的檔案>
> git rebase --continue
>
> # 若要放棄此次 rebase
> git rebase --abort
> ```

---

### 5. 情境三：`git push` 被遠端拒絕

這不是衝突，但是新手最常碰到的問題。當遠端有別人推送的新 commit，而你本地還沒 pull 時，Git 會拒絕你的 push：

```
 ! [rejected]        main -> main (fetch first)
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. Integrate the remote changes before pushing again.
```

**解決步驟：**
```bash
# 1. 先把遠端的最新變更拉下來（若有衝突在此解決）
git pull

# 2. 衝突解決完後，再推送
git push
```

> [!CAUTION]
> **永遠不要用 `git push --force`（或 `git push -f`）來強行解決這個問題。**
> 強制推送會直接覆蓋遠端歷史，把其他人的 commit 永久抹除。
> 若確實有改寫歷史的需求，使用 `git push --force-with-lease`，它會先確認遠端狀態，若遠端有你沒有的新 commit 就會阻止強推。

---

### 6. 衝突排查快速流程

```
git pull / merge / push 後出現問題
          │
          ├─ 出現「rejected」且提示「fetch first」？
          │   └─ 先 git pull，解決衝突後再 git push
          │
          └─ 出現「CONFLICT」訊息？
              ├─ 用 git status 查看哪些檔案衝突（標記為 both modified）
              ├─ 手動編輯衝突檔案（刪除 <<<<<<<, =======, >>>>>>> 標記）
              ├─ git add <解決完的檔案>
              │
              ├─ 若是 merge / pull 引起 → git commit -m "fix: resolve conflict"
              └─ 若是 pull --rebase 引起 → git rebase --continue
```

---

### 7. 衝突相關輔助指令

| 指令 | 說明 |
| :--- | :--- |
| `git status` | 列出所有尚未解決的衝突檔案（標記為 `both modified`）|
| `git diff` | 查看工作區差異，確認衝突標記是否都已刪除 |
| `git log --merge` | 顯示造成此次衝突的兩邊 commit，方便理解衝突來源 |
| `git checkout --ours <file>` | 直接採用「自己這一方」的版本，放棄對方的修改 |
| `git checkout --theirs <file>` | 直接採用「對方」的版本，放棄自己的修改 |
| `git merge --abort` | 放棄整次 merge，回到 merge 前的狀態 |
| `git rebase --abort` | 放棄整次 rebase，回到 rebase 前的狀態 |

---

## 六、 新專案建立完整流程 (From Zero)

這是新手最常需要但最容易卡住的完整步驟，從頭開始建立一個與 GitHub 連動的新專案。

### 方式 A：GitHub 先建，本地 clone（推薦新手）
```bash
# 1. 在 GitHub 上建立新 Repo（勾選 Add README），複製 SSH 網址後

# 2. 在本地 clone 下來（已自動設定好 remote origin）
git clone git@github.com:your-username/your-repo-name.git

# 3. 進入專案資料夾，正常開發後 push
cd your-repo-name
# ... 撰寫程式碼 ...
git add .
git commit -m "feat: initial setup"
git push
```

### 方式 B：本地先建，再推上 GitHub
```bash
# 1. 在本地初始化
git init
git add .
git commit -m "feat: initial commit"

# 2. 在 GitHub 上建立空的新 Repo（不要勾選 README）

# 3. 連結遠端並推送
git remote add origin git@github.com:your-username/your-repo-name.git
git branch -M main        # 確保主分支名稱為 main
git push -u origin main   # 第一次推送，-u 建立追蹤連結
```

> [!NOTE]
> 完成一次 `git push -u origin main` 後，之後的日常開發只需要三個指令：
> ```bash
> git add .
> git commit -m "描述這次的修改"
> git push
> ```
