# 外部 API 調用 (External API)

> [!ABSTRACT]
> 介紹透過 JSON 契約與 Pydantic 實現外部 API 工具調用的機制，探討 Strict 與 Lax 轉換模式，並對比直接調用與代碼模式在效能與安全治理上的考量。

---

## 一、 工具調用的技術實現：函數調用與 JSON 契約

代理人與外部 API 的交互並非直接連通，而是透過一套**結構化的數據交換契約**來達成：

1. JSON 契約的核心組件 (The Contract)

當開發者定義一個工具時，必須向模型提供標準化的 Schema，模型會根據這些描述評估是否調用。

- **名稱 (name)**：工具的唯一識別標誌。
- **功能描述 (description)**：這段自然語言會作為模型的提示詞，引導模型判斷該工具的用途。
- **參數結構 (parameters / input_schema)**：定義輸入數據的類型（如 string, number）與屬性約束。
- **必填項 (required)**：列出調用時必須存在的參數。

2. 工具定義範例 (Tool Definition)

| 平台            | 契約定義結構 (以獲取天氣為例)                                                                                                                                                                        |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **OpenAI**    | 使用 `parameters` 鍵值：<br>`{"name": "get_weather", "description": "取得天氣", "parameters": {"type": "object", "properties": {"location": {"type": "string"}}, "required": ["location"]}}`     |
| **Anthropic** | 使用 `input_schema` 鍵值：<br>`{"name": "get_weather", "description": "取得天氣", "input_schema": {"type": "object", "properties": {"location": {"type": "string"}}, "required": ["location"]}}` |

3. 執行期數據流實例 (Execution Data Flow)

以查詢「Olivia Wilde 男朋友的年齡」為例，展示完整的 **[[1.ReAct|ReAct]] (思考-行動-觀察)** 閉環：

**第一步：模型輸出 JSON 指令 (Action)**

模型判斷需要搜尋資訊，輸出如下 JSON 契約載荷：

```
{
  "thought": "我需要先找出 Olivia Wilde 的男朋友是誰。",
  "action": "Search",
  "action_input": "Olivia Wilde boyfriend"
}
```

**第二步：執行器回傳執行結果 (Observation)**

外部腳手架（Agent Scaffolding）攔截上述 JSON，調用搜尋引擎後，將結果反饋回模型：

```
Observation: Olivia Wilde started dating Harry Styles...
```

**第三步：再次生成與連鎖調用**

模型讀取新事實後，發現仍缺 Harry Styles 的年齡，於是發起第二次契約調用：

```
{
  "thought": "搜尋顯示男朋友是 Harry Styles，現在我需要找出他的年齡。",
  "action": "Search",
  "action_input": "Harry Styles age"
}
```

4. 契約實作的進階模式

- ==**Pydantic 轉換**==：現代框架（如 [[Key Frameworks & Protocols|CrewAI]]）通常允許開發者定義 Python 的 `BaseModel`，框架會自動將其轉化為嚴格的 JSON Schema 給模型讀取，確保數據校驗。
- **並行調用 (Parallel Tool Calls)**：如果任務是「對比 A 與 B 的價格」，模型可以在單次生成中輸出包含兩個工具調用的 JSON 陣列，執行器並行處理後再同時回傳結果。
- **結構化輸出 (Structured Outputs)**：==開發者可設定參數（如 `tool_choice: required`），強制模型在該次生成中不輸出自由文本，僅輸出符合契約的 JSON。==


## 二、 Pydantic 自動型態轉換與驗證
當你把數據輸入給 Pydantic 的模型（Model）時，它不只會檢查資料型態是否正確（驗證），還會**自動嘗試將數據轉換為你指定的型態**。這種特性在程式設計中通常被稱為**型態強轉（Type Coercion）**。
### 自動型態轉換（Data Coercion）

Pydantic 的原則是「儘可能理解你的意圖」。如果輸入的資料型態不對，但可以被安全、合理地轉換，Pydantic 就會自動完成轉換，而不會直接報錯。

### 核心範例

```Python
from pydantic import BaseModel
from datetime import datetime

class User(BaseModel) :
    id: int
    username: str
    is_active: bool
    signup_ts: datetime

input_data = {
    "id": "123",                    # 字串 "123"
    "username": "chris",
    "is_active": "true",            # 字串 "true"
    "signup_ts": "2026-07-10 17:30" # 字串時間
}

user = User(**input_data)

print(user.id)         # 輸出: 123 (已成功轉換為 int)
print(user.is_active)  # 輸出: True (已成功轉換為 bool)
print(type(user.signup_ts)) # 輸出: <class 'datetime.datetime'> (已成功轉換為 datetime 物件)
```

### 常見的自動轉換規則：

- **數字轉換**：字串 `"123"` 或浮點數 `123.0` 會自動轉換為整數 `123`（如果該欄位定義為 `int`）。
    
- **布林值轉換**：字串 `"true"`、`"True"`、`"1"`、`"on"`、`"yes"` 都會被聰明地轉換為布林值 `True`；相對地，`"false"`、`"0"`、`"no"` 等會轉換為 `False`。
    
- **日期時間轉換**：符合 ISO 8601 格式的字串（例如 `"2026-07-10T12:00:00"`）或 Unix 時間戳（Timestamp），會自動轉換為 Python 的 `datetime` 物件。
    

### 兩種轉換模式：Strict 與 Lax

Pydantic（特別是 V2 版本）提供了兩種對待數據轉換的態度：

- **Lax 模式（寬鬆模式，預設）**：如上所述，會主動嘗試進行合理的型態轉換。
    
- **Strict 模式（嚴格模式）**：不允許任何自動轉換。如果定義是 `int`，輸入就必須是 `int`，輸入 `"123"` 就會直接拋出 `ValidationError`。
    

> **如何開啟嚴格模式？** 可以在模型設定中開啟：`model_config = {'strict': True}`，或者在驗證時指定：`User.model_validate(data, strict=True)`。

### 自定義轉換（Custom Validators / Field Validators）

除了內建的轉換邏輯，你常常會遇到更複雜的資料需要清洗。這時可以使用 Pydantic 提供的 `@field_validator` 或 `@model_validator` 來寫自己的轉換邏輯。

```Python
from pydantic import BaseModel, field_validator

class Product(BaseModel):
    name: str
    price: float

    @field_validator('name')
    @classmethod
    def clean_name(cls, v: str) -> str:
        # 自定義轉換：自動去除前後空白，並將英文字母大寫
        return v.strip().upper()

product = Product(name="  apple iphone  ", price=999.0)
print(product.name) # 輸出: "APPLE IPHONE"
```

### 輸出轉換（Serialization / Dumping）

Pydantic 的「轉換」不只發生在**輸入**，也發生在**輸出**。當你需要把 Python 物件轉換成可以傳輸的格式（例如 JSON 或 Dict）時，Pydantic 提供了非常方便的方法：

- **`model.model_dump()`**：將模型物件轉換為標準的 Python `dict`。
    
- **`model.model_dump_json()`**：將模型物件轉換為 **JSON 字串**（會自動把 `datetime`、`UUID` 等無法直接 JSON 序列化的物件轉換為字串）。
    

### 為什麼需要 Pydantic 轉換

在開發後端 API（如 FastAPI）或處理外部資料（如資料庫、網路爬蟲、加密貨幣交易所的 API 數據）時，外部輸入的資料通常是一堆純字串或結構鬆散的 JSON。

Pydantic 的轉換功能扮演了**資料清洗與標準化**的角色：

1. **確保資料髒不到你的核心邏輯**：進到程式內部的資料型態絕對符合預期。
    
2. **減少樣板代碼**：不需要手動寫一堆 `int(x)` 或 `datetime.strptime()`。
    
3. **錯誤提示明確**：如果資料真的爛到無法轉換（例如把 `"abc"` 轉成 `int`），它會噴出結構清晰的錯誤報告，告訴你哪一個欄位出錯。

## 三、 效能優化策略：直接調用 vs. 代碼模式

在操作多個 API 或處理大量數據時，傳統的「直接工具調用」會面臨**上下文膨脹（Context Bloating）**的挑戰：

- **上下文壓力**：如果 API 回傳海量數據（如一萬列的試算表），模型會因為處理過多 Token 而變得遲鈍或出錯。
- **代碼模式 ([[code 模式|Code Mode]])**：先進架構（如 Anthropic 的代碼執行模式）引導模型撰寫代碼來操作 API。代碼在受限沙箱中執行 API 調用並進行數據過濾，**僅將精簡後的結果回傳模型**，據稱可節省高達 **98.7%** 的 Token 消耗。

## 四、 安全與治理考量

由於 API 具備操作數據的能力，其在工具調用中的安全性至關重要：

- **物理隔離**：對於涉及敏感 API 的操作，應使用 **Docker 容器**進行安全隔離，防止惡意指令危害宿主機。
- **人類在環 (Human-in-the-Loop)**：對於具備破壞性（如刪除數據）或資金交易能力的 API 調用，必須強制引入人類審批流程。
- **隱私保護**：利用 [[MCP Client & MCP Server|MCP]] 客戶端在 API 數據進入模型前對敏感資訊（PII）進行**代標記化（Tokenization）**，確保真實數據不流出執行環境。
