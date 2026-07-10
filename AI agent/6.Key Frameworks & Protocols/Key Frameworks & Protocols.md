# 關鍵框架與協定 (Key Frameworks & Protocols)

> [!ABSTRACT]
> 整理 AI 時代的核心協定（MCP、LSP）與三大主流多智能體開發框架（LangChain/LangGraph、CrewAI、AutoGen）的實戰優勢與底層直覺。

---

## 一、 核心協定 (Protocols)：標準的鐵軌規格

在沒有協定之前，每家公司自己做工具、每個模型自己定義 [[Function Calling timing|Function Calling]] 格式，串接起來亂七八糟。**協定就是為了「天下一統」而誕生的。**

### MCP ([[MCP Client & MCP Server|Model Context Protocol]] / 模型上下文協定)

- **它是什麼**：由 Anthropic（Claude 的母公司）在 2024 年底推出並迅速成為開源標準的跨時代協定。
    
- **解決什麼痛點**：我們之前提過，它是 **「隨插即用的萬用插座」**。它把工具、資料庫、文件系統全部標準化。不論你用什麼模型（OpenAI, Gemini, Llama），只要兩端都符合 MCP 協定，大腦就能一秒插上並調用這些工具，完全不需要為不同模型重寫 API 對接代碼。
    

### LSP for AI (Language Server Protocol 變體)

- **它是什麼**：原本是微軟用來讓 VS Code 可以支援各種程式語言（語法檢查、自動補完）的協定。在 AI 時代，它被延伸用來規範 **AI Coding Agents** 與你的編輯器（如 Cursor、VS Code）底層檔案系統、終端機互動的標準通訊語言。
    

## 二、 🏗️ 核心框架 (Frameworks)：幫你造車的工具箱

如果你想建構一個多智能體（Multi-Agent）團隊，你不需要從頭用 `while` 迴圈去刻 [[1.ReAct|ReAct]]。這些框架已經把「大腦、記憶、通訊、攔截器」全部封裝好了。目前市面上有三大主力流派：

### LangChain / [[Architectural Patterns|LangGraph]] (微觀控制與狀態機流派)

- **特點**：目前工業級最穩定的基礎建設。
    
- **你的底層直覺**：它引入了「圖（Graph）與狀態機（State Machine）」的概念。
    
- **實戰優勢**：當你在寫 **[[Code as Executor|Code as Executor]]（程式碼當老大）** 的嚴謹管線時，LangGraph 是最完美的選擇。它允許你用程式碼死死控管每一步的資料流向，[[生成式 AI、LLM 概論|LLM]] 只能在固定的節點（Nodes）裡做 Function Calling。它具備完美的狀態同步（State）與長週期記憶維護。
    

### CrewAI (角色扮演與團隊協作流派)

- **特點**：對人類最友善、最直覺的框架。
    
- **你的底層直覺**：這就是 **[[LLM as Orchestrator|LLM as Orchestrator]]（團隊分流模式）** 的教科書等級實作。
    
- **實戰優勢**：在 CrewAI 裡，你直接定義 `Agent(role='Senior Analyst')`、`Agent(role='Writer')`，然後給他們分配各自的 `Task` 與 `Tools`。CrewAI 背後的內建 Orchestrator 會自動打點好小弟們之間要怎麼用 ReAct 互相傳話（Context Isolation），你只需要寫幾行宣告型程式碼，AI 團隊就組建完成了。
    

### Microsoft AutoGen (對話驅動流派)

- **特點**：微軟開源的殿堂級多智能體框架。
    
- **你的底層直覺**：它的核心思想是「把萬事萬物都當成對話群組裡的成員」。
    
- **實戰優勢**：它擅長處理高自由度的「群聊模式」。你可以讓一個 Code Agent 寫 Code，丟到群組裡；另一個 Executor Agent 自動在背景（Docker 沙箱）執行，執行完把報錯日誌貼回群組；第三個 Critic Agent 在群組裡指責它哪裡寫錯。它們在同一個「對話通道（Event Loop）」裡自動演進、自動完成任務。
