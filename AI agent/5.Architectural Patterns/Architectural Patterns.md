# 架構模式 (Architectural Patterns)

> [!ABSTRACT]
> 分析 Tools (確定性組件) 與 Subagents (專科專家) 的特徵，探討 Tools 的超時重試機制以及 Subagents 的上下文隔離、狀態同步與結構化輸出等工程實踐。

---

### Deterministic Execution (確定性執行)

- **直覺**：**「輸入相同，輸出保證絕對相同。」**
    
- **一話看穿**：例如一個寫死數學公式的 `calculate_rsi()` 函數，只要傳入過去 14 天的 K 線，它算出來的數字 100% 固定。這裡**沒有幻覺風險、沒有機率、容錯率極高**。這也是 **[[Code as Executor|Code as Executor]]** 最喜歡掌控的硬核心臟。
    

### Shared Context (共享上下文)

- **卸妝直覺**：**「工具沒有自己的小祕密，執行結果會直接攤在主大腦的對話歷史裡。」**
    
- **一話看穿**：工具執行完後的 Stdout 輸出（Observation），會被 Agent 框架直接注入到主 [[生成式 AI、LLM 概論|LLM]] 的 Context 視窗中。主大腦能毫無阻礙地看到工具回傳的每一個細節欄位。
    

### Low Latency / Cost (低延遲與低成本)

- **直覺**：**「跑原生的 Python 程式或 SQL 幾毫秒就收工，而且完全不花任何 Token 費用。」**
    
- **一話看穿**：因為工具背後是一台「無情的執行機器（計算機）」，不需要經過神經網路的文字機率預測，所以速度快到飛起（低延遲），成本趨近於零。

## 一、 Tools 的「超時與重試機制」（Engineering Guardrails for Tools）

心智圖上寫 Tools 具備 **Low Latency（低延遲）** 與 **Deterministic（確定性）**。但別忘了，在真實的區塊鏈世界或網路環境中，交易所的 API 是會斷線、卡住或限制頻率（Rate Limit）的。

- **工程陷阱**：如果你的 Tools 寫得很裸露（例如直接用 `requests.get()` 沒設超時），一旦某次 API 沒回應，整個主 Agent 就會停在背景死等，直到系統主程式卡死。
    
- **正確做法**：==在設計 Tools 的 Interface 時，必須強制包裝 `timeout` 攔截，並結合**指數退避重試==（Exponential Backoff）**。


當總指揮官發現「單靠寫死的 Python 函數頂不住，任務需要獨立的語意理解與思考」時，它就會派發 **[[AI Agent 概論|Subagents]]**。Subagent 的背後是**另一個帶有專屬 System Prompt（甚至不同模型）的 LLM**。它具備三大架構優勢：

### Bounded Reasoning (有邊界/受限的推理)

- **直覺**：**「各司其職，小弟不需要知道公司的上市計畫，只要專心把程式寫好就好。」**
    
- **一話看穿**：如果讓一個主 LLM 負責全盤大局，大腦很容易過載。我們把大任務切碎，分給帶有「專門提示詞」的 Subagent（例如：一個專職洗資料的 Code Agent、一個專職分析情緒的 Text Agent）。這群小專家在各自被限定的狹窄領域（Bounded）裡進行高精度的推理。
    

### Context Isolation (上下文隔離 / 防止 Token 爆炸)

- **直覺**：**「小弟跟客戶扯皮、Debug 了 50 輪的髒對話，不應該塞進總經理（主大腦）的辦公桌上。」**
    
- **一話看穿**：這是節省 Token 最強大的防禦！Subagent 自己在背景用 **[[1.ReAct|ReAct]] Loop** 跟 Docker 沙箱反覆死磕、改了 5 次 Bug。這 5 次噴出來的幾萬字報錯日誌，完全被**隔離**在 Subagent 自己的視窗裡。當它搞定後，它只會回傳一句話給主 Orchestrator：「報告總經理，資料洗好了，這是最終的 5 列數據。」主大腦的 Context 乾淨無比。
    

### Parallel Processing (非同步/並行處理)

- **直覺**：**「多核心同步開工，不用排隊。」**
    
- **一話看穿**：當使用者說：「幫我分析 BTC、ETH、XRP 三個幣種的情緒。」Orchestrator 不需要老實地一隻一隻查。它直接分身出三個獨立的 Subagents，用非同步（`async`）的方式**同時分頭去狂奔**。三個小模型在背景並行運算，最後由 Orchestrator 一次收割。

## 二、 Subagents 的「狀態同步問題」（State Synchronization & Loss）

下半部的心智圖提到 Subagents 具備 **Context Isolation（上下文隔離）**，這能完美防止主大腦的 Token 爆炸。但這會帶來一個副作用——**==「不同模型之間的資訊斷層」==**。

- **工程陷阱**：如果 Subagent A（負責洗資料）洗完資料後，把結果存在它自己的 Context 裡，接著 Orchestrator 叫 Subagent B（負責回測）去跑程式。Subagent B 會因為看不到 A 的上下文，導致完全不知道資料放在哪裡。
    
- **正確做法**：這就是為什麼現代進階架構（如 **LangGraph**）不再只依賴模型的 Context 傳話，而是強制引入一個全局的 **「State（狀態黑板）」**。
    
    - 所有的 Subagents 共同看著同一個在記憶體裡的 Python 字典（Dict）或資料庫。
        
    - Subagent A 洗完資料，把檔案路徑寫進 `State['cleaned_data_path'] = '/tmp/btc.csv'`。
        
    - Subagent B 啟動時，直接去讀取 `State`。這才是優雅處理 Context Isolation 卻又能完美協作的解法。

## 三、 Subagents 專用的「結構化輸出」（Structured Outputs for Subagents）

當 Orchestrator 呼叫 Subagent 時，Subagent 回傳的結果不能是一堆混亂的、給人類看的發揮文字。因為 Orchestrator 需要用這些結果去決定下一步。

- **工程陷阱**：你讓 Subagent 去分析市場情緒，它回傳了：「嗯，看來今天 BTC 社群情緒蠻激進的，大家都在做多...。」主 Orchestrator 看到這堆字，會很難用 `if-else` 去做邏輯判斷。
    
- **正確做法**：即便在 Subagents 之間互傳訊息，也要==強迫使用 **[[External API|Pydantic]] 轉換為 JSON Schema Interface**。==
