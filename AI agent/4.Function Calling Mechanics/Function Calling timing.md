# 函數呼叫時機與機制 (Function Calling Timing & Mechanics)

> [!ABSTRACT]
> 剖析 Function Calling 的核心機制與語意空間匹配原理，並針對工具過載與描述衝突等判斷失敗的常見場景提出前置分類等優化方案。

---

## 一、 核心機制：Function Calling（函數呼叫）

這是目前最主流、也是 OpenAI、Gemini 等大廠在 API 底層直接支援的機制。

### 告訴 [[生成式 AI、LLM 概論|LLM]] 你有哪些工具（宣告）

在把使用者的問題丟給 LLM 之前，程式（例如你的 Python Agent 框架）會先定義好一組工具清單，並用 JSON Schema 格式詳細描述這些工具。

JSON

```
// 程式傳給 LLM 的工具清單範例
[
  {
    "name": "get_crypto_price",
    "description": "當使用者想查詢加密貨幣（如 BTC, ETH）的即時價格時使用。",
    "parameters": {
      "type": "object",
      "properties": {
        "ticker": {"type": "string", "description": "貨幣代碼，例如 btc"}
      },
      "required": ["ticker"]
    }
  }
]
```

### LLM 的語意理解與決策

當使用者輸入：_「幫我看看現在比特幣多少錢？」_

LLM 接收到「使用者問題」+「工具清單」後，它會發揮強大的語意理解能力。它發現使用者的意圖（比特幣多少錢）與 `get_crypto_price` 的 `description` 高度吻合。


### LLM 決定不回答文字，而是「吐出參數」

這時候 LLM **不會**直接回答：「我知道了，我去幫你查。」

它會判定「現在必須調用工具」，並輸出一個特殊的 JSON 結構：

JSON

```
{
  "tool_calls": {
    "name": "get_crypto_price",
    "arguments": {"ticker": "btc"}
  }
}
```

隨後，後端的 Agent 框架（如 [[Key Frameworks & Protocols|LangChain]] 或你寫的 Python 程式）看到這個 JSON，就會真的去觸發對應的 API，拿到價格後，再餵回給 LLM 讓它組織成人類看得懂的回答。

## 二、 LLM 是怎麼精準判斷「要不要用」與「用哪一個」？

這背後主要依賴 LLM 的兩種能力：

- **語意空間的匹配（Semantic Match）**：LLM 透過注意力機制（Attention Mechanism），將使用者的 Prompt 與工具的描述（Description）進行向量比對。因此，==**工具的描述寫得好不好，直接決定了 LLM 判斷的準確度**==。如果描述含糊不清，LLM 就容易「幻覺」或選錯工具。
    
- **少樣本學習（Few-Shot Prompting）**：在複雜的 Agent 系統中，開發者會在 Prompt 中塞入幾個範例（例如：「如果使用者說 A，你就應該調用工具 B」）。LLM 模仿能力極強，看過範例後就能大幅提升判斷時機的精準度。在開發 Agent 時，我們如果是用 **n8n、LangChain 或自寫 Python 腳本**，本質上都是在後端默默地把這些範例當成「背景設定」，在每一次呼叫 API 時重複餵給 LLM，營造出它「很有經驗、判斷很準」的專業效果。
    

## 三、 什麼時候會判斷失敗？（常見的坑）

儘管 LLM 很聰明，但在以下情況下，它常會出現判斷失誤：

1. **工具太多（Tool Overloading）**：當你一口氣塞給 LLM 30 個工具時，它的上下文注意力和推理能力會下降，開始出現「選擇障礙」或亂選工具。
    
2. **描述衝突（Ambiguous Descriptions）**：如果你有兩個工具分別叫 `get_crypto_info` 和 `get_crypto_price`，且描述很接近，LLM 就很容易在該查價格時去查了基本面資訊。
    
3. **過度依賴（Over-reliance）**：有時使用者只是想跟 AI 聊天（例如：「你覺得比特幣是未來的趨勢嗎？」），但 LLM 因為太過急切，誤判成需要調用 `get_crypto_price`，這就屬於調用時機誤判。

==通常會在 LLM 前面再加一層路由（Router）或分類器（甚至用較小的 local 模型如 Llama 先做[[Intent Filtering|意圖過濾]]），確保只有在真正需要時，才把複雜的工具清單暴露給主 LLM。==
