# 代碼模式 (Code Mode)

> [!ABSTRACT]
> 說明代碼模式（集中式腳本執行）相較於傳統離散工具調用的優勢，分析其核心功能與 LLM 的四個黃金觸發時機，並探討其安全風險與防禦機制。

---

## 一、 什麼是代碼模式？

**代碼模式**是指在 Agent 架構中，[[生成式 AI、LLM 概論|LLM]] 不再只是單純、離散地呼叫單一 API 工具，而是「直接編寫一段完整的腳本（通常是 Python），並在隔離的沙箱環境中一次性執行」的運作模式。

### 傳統模式 vs 代碼模式的技術斷層

- **傳統模式（離散工具呼叫）**：模型每調用一個工具，都需要將工具定義與執行結果（Observation）反覆傳回上下文視窗（Context Window）。在處理複雜任務時，會導致「上下文膨脹（Context Bloating）」**與**「多輪對話延遲」。
    
- **代碼模式（集中式腳本執行）**：LLM 轉化為「開發者角色」，將多個步驟、邏輯判斷（If-Else）、迴圈（Loops）直接寫成程式碼，交給後端沙箱。==**模型只看最終執行的 Stdout（標準輸出）結果**。==
    

## 二、 代碼模式的 4 大核心功能與優化機制

### API 隨選式串接（Dynamic Tool Retrieval）

- **傳統痛點**：MCP（模型上下文協定）客戶端通常在一開始就把成百上千個工具定義（JSON Schema）全部塞進 System Prompt，初始 Token 直接爆炸。
    
- **代碼模式作法**：將 MCP 伺服器呈現為文件系統中的 API。Agent 像工程師查閱 GitHub 檔案庫或 `import` 庫一樣，先列出目錄，==**只針對當前任務讀取「必要」的工具定義文件**==。==
    
- **優化效果**：顯著降低初始 Prompt 的體積，讓工具箱具備「無限擴展」的可能。
    

### 沙箱內「數據邊緣計算」（Edge Data Filtering）

- **傳統痛點**：處理一個 10,000 橫列的 CSV 檔或大型 JSON 時，整個資料集都必須流入 LLM 的上下文，成本極高且容易超出 Token 限制。
    
- **代碼模式作法**：LLM 撰寫程式碼（如 `pandas` 腳本），直接在隔離的執行環境（Sandbox）中對數據進行矩陣運算、過濾與聚合。
    
- **優化效果**：海量原始數據在沙箱內部被清洗，**最後僅將提煉後的 5 橫列核心數據回傳給 LLM**。==原始數據完全不經過模型==，達成極致的 Token 節省。
    

### 多步調用合併（Multi-Step Consolidation）

- **傳統痛點**：遵循 [[1.ReAct|ReAct 循環]]（思考 $\rightarrow$ 行動 $\rightarrow$ 觀察）。若要「從 Google Drive 下載 $\rightarrow$ 跑回測 $\rightarrow$ 發送 Slack 通知」，需要 3 輪對話，每一輪都要重複傳遞遞增的歷史紀錄。
    
- **代碼模式作法**：模型直接寫出一段包含錯誤處理（Try-Catch）的完整腳本。
    
- **優化效果**：**「多輪對話，一次解決」**。原本需要 5 到 10 輪的交互，縮短為單次生成、單次執行。不僅降低 Token 總量，也大幅減少首字生成延遲（TTFT，Time-to-First-Token），提升 Pipeline 吞吐量。
    

### 隱私保護與 PII 代標記化（PII De-identification）

- **代碼模式作法**：MCP 客戶端在沙箱內執行代碼時，利用腳本在本地端攔截敏感資訊（如姓名、信用卡、Email），並進行代標記化（Tokenize）或遮蔽（Masking）。
    
- **優化效果**：真實的隱私數據安全地留在沙箱環境內，傳回給雲端 LLM 的只有去識別化後的標記，完美符合資安合規。
    
## 三、 LLM 什麼時候會判斷要進入「代碼模式」
LLM 決定使用代碼模式，核心判斷點在於：**「這個任務如果用大腦（語意機率）盲猜，或用一般的 API 工具一個一個串聯，效率與準確度會極低。」**

### 時機一：涉及「多步驟、有前後因果關係」的複雜運算

- **人類指令**：_「幫我找出過去 30 天，BTC 價格大於 50,000 且當天交易量高於前一天 20% 的所有日期。」_
    
- **LLM 的判斷**：如果用一般工具，我得先呼叫 `get_30days_kline` 拿到一坨巨大的 JSON 數據，然後我（LLM）得自己用眼睛一筆一筆看、一筆一筆算。這不僅會讓上下文（Context）爆炸，我還一定會算錯。
    
- **決策**：**「直接寫段 Python 用 Pandas 做矩陣篩選最快！」**
    

### 時機二：遇到需要「精確計時或死邏輯」的迴圈與條件判斷（Loops & Control Flow）

- **人類指令**：_「去查 A 帳戶的餘額，如果小於 1000，就從 B 帳戶轉 500 進去；如果轉帳失敗，連續重試 3 次，每次間隔 1 秒，成功的話傳 Slack 通知我。」_
    
- **LLM 的判斷**：這涉及了 `if-else`、`while` 迴圈、`time.sleep(1)` 以及異常處理（`try-except`）。如果用傳統的 ReAct 工具模式，只要其中一步出錯，我就要跟 Agent 架構來回對話 10 次，延遲太高。
    
- **決策**：**「我直接把這個自動化邏輯寫成一個完整的 `.py` 腳本一次執行完！」**
    

### 時機三：需要處理「無法直接塞入 Prompt」的大型數據檔案

- **人類指令**：_「幫我分析這個 10MB 的 Excel 交易報表，統計總損益並畫出趨勢圖。」_
    
- **LLM 的判斷**：10MB 的文字我根本讀不完（超出 Token 限制），而且我也沒辦法直接在文字視窗裡「畫圖」。
    
- **決策**：**「調用代碼模式，寫 `openpyxl` 讀取檔案，並用 `matplotlib` 畫圖，把圖片存到硬碟裡。」**
    

## 四、 Coding 腳本也是輸出成 JSON 格式給 Agent 架構的嗎？

**是！它依然是透過結構化的格式（如 JSON 或特定標記）傳遞給 Agent 架構。**LLM 無論多聰明，它的唯一輸出渠道就是「文字（Text Stream）」。為了讓 Agent 架構（客戶端）知道「**這段文字不是要對人類說的話，而是要拿去沙箱執行的程式碼**」，業界目前有兩種主流的通訊實作方式：

### 方式 A：封裝在標準的 [[Function Calling timing|Function Calling]] (JSON) 裡（最主流、最嚴謹）

在這種架構下，開發者會給 LLM 一個萬用工具，名字就叫 `execute_python_code`，這個工具只接收一個參數叫 `code`（字串型態）。

當 LLM 決定用代碼模式時，它吐出的底層 JSON 會長這樣：

JSON

```
{
  "tool_calls": {
    "name": "execute_python_code",
    "arguments": {
      "code": "import pandas as pd\ndf = pd.read_csv('wallet.csv')\nprint(df['balance'].sum())"
    }
  }
}
```

**Agent 架構（如 n8n, [[Key Frameworks & Protocols|CrewAI]], [[MCP Client & MCP Server|MCP 客戶端]]）收到這個 JSON 後的動作：**

1. 解析出參數 `arguments.code` 的純文字內容。
    
2. 把這段純文字丟進 Docker 沙箱，建立一個臨時的 `script.py`。
    
3. 在沙箱內執行 `python script.py`。
    
4. 捕獲系統輸出的 Stdout，再打包回傳給 LLM。
    

### 方式 B：使用 XML/Markdown 標記攔截（如 Claude Code / Anthropic 模式）

某些先進的模型（如 Claude 3.5 Sonnet）更傾向於直接在對話中輸出特定的 Markdown 區塊或自訂標記：

XML

```
思維：我需要計算總和，使用 Python 處理。
<call:execute_python_script>
import os
print(os.listdir('.'))
</call:execute_python_script>
```

或者最常見的：

Python

```
print(2.431 * 89.22 / 1.5)
```

此時，Agent 架構的 Token 攔截器（Parser）在讀取模型輸出的文字流（Streaming）時，==一旦偵測到 `<call:execute_python_script>` 或 ` ```python `，就會立刻中斷對人類的顯示，把裡面的程式碼截取出來送進沙箱執行。==
## 五、 代碼模式的「隱藏風險與副作用」

在架構設計上，引入代碼模式並非百利而無一害，必須注意以下工程風險：

1. **沙箱逃逸與安全漏洞（Sandbox Escape）**
    
    - **風險**：==若 LLM 被惡意 Prompt 注入（Prompt Injection），寫出 `os.system("rm -rf /")` 或嘗試掃描內部網路。==
        
    - **防禦**：執行環境必須是**絕對隔離且無狀態（Stateless）的沙箱**（如 Docker、gVisor、Wasm），並限制 CPU/記憶體上限與網路存取權限，執行完畢即銷毀。
        
2. **程式碼「幽靈死循環」（Infinite Loop）**
    
    - **風險**：LLM 寫的 `while` 迴圈可能存在邏輯漏洞，導致沙箱卡死，耗盡伺服器資源。
        
    - **防禦**：必須在後端執行器設定硬性的==**超時中斷機制==（Timeout，例如超過 5 秒自動終止）**。
        
3. **錯誤除錯循環（Debugging Loop Cost）**
    
    - **風險**：LLM 寫的程式碼可能會有 Bug（如 `NameError`）。沙箱執行失敗會回傳報錯日誌，LLM 看到後會嘗試「自我修復」重新寫一段，這可能導致 Agent 卡在「報錯 $\rightarrow$ 改 code $\rightarrow$ 再報錯」的無窮死循環中，瞬間燒光 API 費用。
        
    - **防禦**：必須==限制 Agent 的**最大重試次數==（Max Attempts $\le$ 3 次）**，失敗則強制降級回傳給人類。
        

## 六、 觸發條件與職責邊界

代碼模式是「開發者設定前提，模型自主決策」的協作過程：

### A. 開發者職責（預先配置環境）

1. **開啟權限**：在框架中顯式允許（如 CrewAI 的 `allow_code_execution=True`，或自建的執行開關）。
2. **配置安全沙箱**：提供具備運行環境（如預裝好 `pandas`, `numpy`, `requests` 等必備庫）的隔離容器。
3. **宣告系統資源**：在 System Prompt 中注入完整的環境說明書（API 宣告），透過 MCP 協定將本地文件或資料庫路徑暴露給模型。

### B. LLM 職責（自主決策 4 大黃金時機）

當大腦判定任務涉及「語意機率盲猜或一般 API 串聯效率極低」時，會自主決定進入代碼模式並編寫腳本：

1. **多步驟複雜運算**：需要跨數據多重條件篩選時（如使用 Pandas 進行矩陣計算與數據過濾）。
2. **死邏輯控制流**：涉及 `if-else`、`while` 迴圈、`try-except` 異常重試機制時。
3. **大數據超載**：處理超出 Context 視窗限制的大型檔案（如 10MB 報表）時。
4. **非文字產出**：需要動態繪製圖表（如用 `matplotlib` 畫 K 線趨勢圖）並保存至磁碟時。
