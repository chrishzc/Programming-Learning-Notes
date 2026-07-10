# 代碼作為執行器 (Code as Executor)

> [!ABSTRACT]
> 對比 LLM as Orchestrator 與 Code as Executor 兩種編排模式，分析後者在零幻覺、低延遲與可測試性上的優勢，並提供專案選型建議。

---

## 一、 核心概念：為什麼需要 Code as Executor？

在真正的工業級 Data Pipeline（資料管線）或高頻自動化交易系統中，**完全讓 [[生成式 AI、LLM 概論|LLM]] 當主導者（Orchestrator）是非常危險且低效的**。

因為 LLM 具有「隨機性（Stochastic）」，它今天心情好 [[1.ReAct|ReAct]] 跑了 3 步就交件，明天可能突然犯傻在第 2 步卡死，或者輸出格式崩壞，導致你珍貴的交易管線直接中斷。

為了追求 **100% 的穩定度與極致的效能**，工程師會採用 **Code as Executor**：

==系統的主架構是一段用 Python 寫死的、非常嚴謹的程式碼==（例如使用 Airflow 或 Prefect 控管的 DAG 流程）。這段程式碼負責掌控全局、處理資料流、管理狀態、對接資料庫。當程式碼遇到「傳統演算法解決不了的模糊語意問題」時，程式碼才會發動 API 去呼叫 LLM 來幫忙。

## 二、 兩種模式的底層對比：


### 模式 A：[[LLM as Orchestrator|LLM as Orchestrator]]（LLM 是老大）

```
[使用者指令] ──> 【主 LLM 大腦】
                     │
                     ├──> (Function Calling) ──> 呼叫 K 線 API
                     ├──> (代碼模式) ──────────> 丟給 Docker 計算
                     └──> (Function Calling) ──> 決定下單
```

- **特點**：自由度極高。但如果 LLM 突然算錯或 JSON 格式出錯，整個下單管線就崩潰了。
    

### 模式 B：Code as Executor（Python 程式碼是老大）

```
[定時排程/Webhook] ──> 【Python 主程式 (Executor)】
                            │
                            ├──> (100% 穩定) ──> 自動去 SQL 撈歷史數據
                            ├──> (100% 穩定) ──> 用 Pandas 執行 N 型突破數學計算
                            │
                            v (遇到模糊語意任務：如分析市場情緒、或將混亂的外部公告轉成結構化資料)
                    【呼叫 LLM (當作高階過濾器工具)】
                            │
                            v (拿到 LLM 的結構化答案後)
                    【Python 主程式】──> (安全檢查) ──> 執行下單
```

- **特點**：極度穩定。核心的資料處理與交易邏輯完全由原生的 Python 程式碼硬核掌控（Executor），==LLM 被降級為一個專門用來處理「語意理解」的**插件/工具（Tool）**==。
    

## 三、 Code as Executor 在實戰中的 3 大核心優勢

1. **零幻覺風險（Deterministic Safety）**：
    
    核心的商務邏輯（如資金風控、平倉線、下單數量計算）是用真正的 Python 程式碼硬寫下來的。不論 LLM 怎麼胡言亂語，它都無法突破 Python 程式碼設定的 `if balance < min_required:` 安全防線。
    
2. **極致的效能與低延遲（Latency Optimization）**：
    
    撈資料、矩陣運算、資料庫讀寫，本來就是程式碼的強項。如果讓 LLM 用 ReAct 一步一步去呼叫，會耗費好幾秒的網路延遲與 Token 費；交給 Python 程式碼當 Executor，幾毫秒就收工。
    
3. **容易測試與除錯（Debuggability）**：
    
    因為程式碼結構是固定的，你可以為 Executor 寫標準的**單元測試（Unit Test）**。你可以確信「資料處理管線」本身是 100% 沒 Bug 的，唯一的變數只有 LLM 回傳的語意品質。
    

## 四、 總結：你該怎麼在專案中抉擇？

- 如果你的任務**邊界非常模糊、千變萬化**（例如做一個全自動的 AI 特助，要幫你安排行程、回信、到處找資料） $\rightarrow$ 適合 **LLM as Orchestrator**，讓 LLM 發揮 ReAct 的動態編排能力。
    
- 如果你的任務**邏輯非常嚴謹、數據量極大、容錯率極低**（例如企業級的 ETL 資料清洗管線） $\rightarrow$ 強烈建議採用 **Code as Executor**。
