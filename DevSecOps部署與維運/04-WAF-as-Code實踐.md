# 04 · WAF-as-Code 實踐

> **學習目標**：理解 WAF 在 L7 的防禦角色，並掌握以 Terraform 聲明式管理 WAF 規則的工程化方法。

---

## 一、WAF vs 反向代理：防禦層次差異

==反向代理（如 Nginx）和 WAF 都坐在流量路徑上，但**防禦的東西完全不同**==：

| 面向 | 反向代理（Nginx）| WAF |
|------|----------------|-----|
| **運作層** | L4 傳輸 + L7 協定合規 | L7 **HTTP 語意**深度解析 |
| **主要防禦** | DoS 緩解、速率限制、TLS 終端、緩衝區溢位 | SQLi、XSS、惡意爬蟲、零日漏洞 |
| **看的內容** | 連線數、請求大小、IP、Port | HTTP 路徑、Query String、Body 參數、Cookie 值 |
| **典型工具** | Nginx、HAProxy、Traefik | AWS WAFv2、Cloudflare WAF、ModSecurity |
| **能否阻斷業務邏輯漏洞** | ❌ | ⚠️ 部分（靠行為規則）|

**兩者是互補關係，不是替代關係**：

```
Client
  │
  ▼
反向代理（Nginx）← 過濾：速率超標、緩衝區過大、非法 Method
  │
  ▼
WAF             ← 過濾：SQLi payload、XSS 腳本、已知攻擊特徵
  │
  ▼
應用伺服器      ← 只接收「被兩層過濾後」的乾淨流量
```

---

## 二、WAF 運作原理

### 2.1 WAF 檢查的 HTTP 請求欄位

WAF 不是只看「URL」，它==對整個 HTTP 請求進行**全欄位語意解析**==：

```
HTTP Request
┌─────────────────────────────────────────┐
│ GET /search?q=<script>alert(1)</script> │  ← URI + Query String
│ Host: example.com                       │  ← Header
│ User-Agent: sqlmap/1.7                  │  ← Header（已知攻擊工具特徵）
│ Cookie: session=eyJhbGci...             │  ← Cookie
│ X-Forwarded-For: 192.168.1.1            │  ← Header
│                                         │
│ {"username": "admin' OR 1=1--"}         │  ← Request Body（JSON）
└─────────────────────────────────────────┘
         │
         ▼
       WAF 對每個欄位獨立執行規則比對
```

**WAF 可檢查的欄位**：

| 欄位 | 說明 | 典型攻擊 |
|------|------|---------|
| `URI Path` | 路徑本身 | 路徑遍歷 `../../etc/passwd` |
| `Query String` | URL `?` 後的參數 | XSS、SQLi payload 注入 |
| `Request Body` | POST/PUT 的內容 | JSON/Form 中的 SQLi |
| `Headers`（全部）| 任意 Header 值 | Header Injection |
| `Cookie` | Cookie 值 | Session Fixation payload |
| `HTTP Method` | GET/POST/DELETE... | 非法 Method（如 TRACE）|
| `URI 完整字串` | 含路徑 + 查詢 | 複合型攻擊 |

---

### 2.2 請求處理流水線（Request Processing Pipeline）

==WAF 接收到請求後，**不是直接把原始字串拿去比對規則**。攻擊者會對 payload 進行各種編碼來繞過規則，所以 WAF 必須先「解碼還原」==再比對：

```
原始請求進入 WAF
       │
       ▼
【Step 1：解碼轉換（Text Transformation）】
  URL Decode：  %3Cscript%3E  →  <script>
  HTML Decode： &lt;script&gt; →  <script>
  Base64 Decode（如有）
  小寫轉換：    SELECT        →  select
  壓縮空白：    OR   1=1      →  OR 1=1
       │
       ▼
【Step 2：規則比對（Pattern Matching）】
  正則表達式比對 OR 字串比對
  例：偵測 <script> 特徵
       │
       ▼
【Step 3：動作執行（Action）】
  Block → 回傳 403，請求終止
  Allow → 轉發給後端
  Count → 記錄，繼續轉發
```

**為什麼解碼很重要？**

```
攻擊者的原始 payload：  <script>alert(1)</script>
URL 編碼後送出：       %3Cscript%3Ealert%281%29%3C%2Fscript%3E

沒有解碼的 WAF → 比對不到 <script> 特徵 → 放行（被繞過）
有解碼的 WAF  → 先還原 → 比對到 <script> → 阻斷
```

在 AWS WAFv2 的 Terraform 中，`text_transformation` 欄位就是控制這個步驟：

```hcl
text_transformation {
  priority = 0
  type     = "URL_DECODE"      # 先 URL 解碼
}
text_transformation {
  priority = 1
  type     = "LOWERCASE"       # 再轉小寫，防止大小寫繞過
}
```

---

### 2.3 偵測方法：==三種流派==

#### 方法一：特徵比對（Signature-based）

最傳統也最常見的方法。維護一個龐大的「已知攻擊特徵」資料庫，對每個請求進行比對：

```
規則範例（偽代碼）：
IF request.query_string CONTAINS "' OR 1=1"
  OR request.query_string CONTAINS "UNION SELECT"
  OR request.body CONTAINS "xp_cmdshell"
THEN → Block
```

**優點**：快速、精確（已知攻擊的低誤報率）  
**缺點**：對**未知攻擊**（Zero-day）無效；攻擊者可用混淆技術繞過

---

#### 方法二：異常評分（Anomaly Scoring）

OWASP CRS（ModSecurity 使用的規則集）採用的方法。**不是單條規則觸發即阻斷**，而是累積「可疑分數」：

```
請求進入 WAF
  │
  ├── 包含 SQL 關鍵字？      → +5 分
  ├── User-Agent 是已知工具？ → +3 分
  ├── 路徑含 ../？            → +4 分
  ├── 請求體超大？            → +2 分
  │
  ▼
累計分數 ≥ 閾值（例：10 分）→ Block
累計分數 < 閾值            → Allow
```

**優點**：對「部分符合」的攻擊有效，降低誤報  
**缺點**：需要調整閾值，設定複雜

AWS WAFv2 的 **AWS Managed Rules AWSManagedRulesCommonRuleSet** 部分規則也採用類似的邏輯。

---

#### 方法三：機器學習 / 行為分析（ML-based）

較新的雲端 WAF（Cloudflare、AWS WAF Bot Control）採用：

- 學習**正常流量的行為基線**（請求頻率、路徑分布、時間模式）
- 偵測**偏離基線的異常行為**，即使沒有已知攻擊特徵也能發現

**典型應用**：Bot 識別（判斷是否為真人操作）、帳號填充攻擊（Credential Stuffing）偵測

---

### 2.4 誤報（False Positive）的根源與處理

WAF 的最大日常挑戰不是攻擊，而是==**把正常請求誤判成攻擊**==：

| 場景     | 說明                                            |     |
| ------ | --------------------------------------------- | --- |
| 富文本編輯器 | 用戶在 CMS 貼入 HTML 內容，包含 `<script>` → 被 XSS 規則阻斷 |     |
| 搜尋框    | 用戶搜尋「SELECT 課程」→ 被 SQLi 規則阻斷                  |     |
| 安全研究工具 | 內部滲透測試工具 → 被 Bot 規則阻斷                         |     |
| 特殊字符   | 日文、阿拉伯文中的某些字符觸發正則                             |     |

**處理流程**：

```
發現誤報
   │
   ▼
分析：哪條規則觸發？（從 WAF 日誌找 terminatingRuleId）
   │
   ▼
選擇處理方式：
   ├── 方案 A：Rule Override（排除特定規則對特定路徑的執行）
   ├── 方案 B：調低 Anomaly Score 閾值
   └── 方案 C：新增白名單條件（scope_down 排除合法請求特徵）
   │
   ▼
PR Review → 審批 → Terraform Apply → 驗證修復
```

**AWS WAFv2 中排除特定規則的寫法**：

```hcl
statement {
  managed_rule_group_statement {
    name        = "AWSManagedRulesCommonRuleSet"
    vendor_name = "AWS"

    # 排除某條子規則（例：排除 SizeRestrictions_BODY 以允許大型上傳）
    rule_action_override {
      name = "SizeRestrictions_BODY"
      action_to_use {
        count {}  # 只計數，不阻斷
      }
    }
  }
}
```

---

## 三、WAF 的核心防禦能力

### 3.1 OWASP Top 10 覆蓋

WAF 的規則集通常對應 OWASP Top 10 漏洞：

| 攻擊類型 | WAF 防禦方式 | 範例 payload |
|---------|------------|-------------|
| **SQL 注入（SQLi）** | 比對 SQL 關鍵字特徵 | `' OR 1=1 --`、`UNION SELECT` |
| **跨站腳本（XSS）** | 偵測 HTML/JS 注入特徵 | `<script>alert(1)</script>` |
| **路徑遍歷** | 比對 `../` 序列 | `../../etc/passwd` |
| **命令注入** | 偵測 Shell 命令字符 | `; cat /etc/passwd` |
| **協定異常** | 驗證 HTTP 格式合規性 | 畸形 Header、非標準 Method |

### 3.2 虛擬補丁（Virtual Patching）

當一個新 CVE 被揭露，而應用代碼**尚未修補**時，WAF 可以立即部署規則阻斷已知的攻擊特徵，為開發團隊爭取修補時間窗口：

```
CVE 揭露（例：Log4Shell）
    │
    ▼ 數小時內
WAF 部署阻斷規則（阻擋 ${jndi:...} 特徵）
    │
    ▼ 數天/數週後
開發團隊修補代碼、升級依賴
    │
    ▼
移除 WAF 虛擬補丁（可選，或保留作縱深防禦）
```

> **關鍵概念**：WAF 規則是「即時部署的臨時護盾」，代碼修復才是「根治」。兩者不可互相取代。

### 3.3 Bot 管理與速率限制

| 能力 | 說明 |
|------|------|
| **速率限制（Rate Limiting）** | 限制單一 IP 在時間窗口內的請求數 |
| **Bot 特徵識別** | 偵測自動化工具的 User-Agent、行為模式 |
| **IP 信譽過濾** | 比對已知惡意 IP 資料庫（Threat Intelligence Feed）|
| **地理封鎖（Geo-blocking）** | 只允許特定國家 / 地區的 IP |
| **爬蟲挑戰（Challenge）** | 對可疑請求回傳 CAPTCHA 或 JS Challenge |

---

## 三、傳統 WAF 維運的問題

在沒有導入 WAF-as-Code 之前，WAF 的維護長這樣：

```
安全工程師 → 登入 AWS Console
           → 在 GUI 手動新增 / 修改規則
           → 存檔（沒有審批流程）
           → Staging 沒更新（忘了）
           → 三個月後沒有人記得為什麼有這條規則
```

這帶來四個系統性問題：

| 問題 | 說明 |
|------|------|
| **配置漂移（Configuration Drift）** | Dev / Staging / Prod 的 WAF 規則各自演化，最終不一致 |
| **無審計軌跡** | 誰在什麼時間改了什麼規則，完全不可追溯 |
| **部署延遲** | 新漏洞出現時，手動更新規則需要人工操作，速度慢 |
| **「設定後即遺忘」** | 規則只增不減，累積大量過時規則，噪音增加 |

---

## 四、WAF-as-Code 架構

### 4.1 核心理念

**WAF 規則 = 代碼**，==與應用程式代碼放同一個版本控制系統，並整合進 CI/CD 管道==：

```
WAF 規則修改（.tf 檔）
       │
       ▼
Pull Request → Code Review（安全規則變更需要審查）
       │
       ▼
CI Pipeline → terraform plan（預覽變更影響）
       │
       ▼
人工核准（或自動 merge）
       │
       ▼
CD Pipeline → terraform apply（自動部署到目標環境）
       │
       ▼
Staging 先驗證 → 再推 Prod
```

**WAF-as-Code 帶來的改善**：

| 傳統 GUI 維護 | WAF-as-Code |
|-------------|-------------|
| 無版本歷史 | `git log` 可追溯所有變更 |
| 無審批流程 | PR Review 強制審查 |
| 環境不一致 | 同一份 `.tf`，多環境一致部署 |
| 回滾困難 | `git revert` + `terraform apply` |
| 部署需人工 | CI/CD 自動化 |

### 4.2 工具選擇

| 工具 | 適用場景 | 備註 |
|------|---------|------|
| **Terraform + AWS WAFv2** | AWS 環境（ALB、API Gateway、CloudFront）| 本篇重點 |
| **Terraform + GCP Cloud Armor** | GCP 環境（Cloud Load Balancing）| 語法相似，概念相通 |
| **Terraform + Azure WAF** | Azure 環境（Application Gateway、Front Door）| 同上 |
| **Cloudflare Terraform Provider** | Cloudflare WAF 規則管理 | 適合已用 Cloudflare 的場景 |
| **ModSecurity + OWASP CRS** | 自建 Nginx/Apache（開源方案）| 需自己管規則更新 |

---

## 五、AWS WAFv2 完整實作解析

### 5.1 核心概念架構

```
Web ACL（Web Access Control List）
  │
  ├── Rule 1（priority=1）: AWS 託管規則集
  │     └── Statement: managed_rule_group
  │
  ├── Rule 2（priority=2）: 自定義速率限制
  │     └── Statement: rate_based_statement
  │
  ├── Rule 3（priority=3）: IP 封鎖清單
  │     └── Statement: ip_set_reference
  │
  └── Default Action: allow / block
        （所有規則都不觸發時的預設行為）
```

**優先級（Priority）規則**：數字越小越先評估；第一條觸發的規則決定結果，後面的不再評估。

### 5.2 Web ACL 基本 Terraform 骨架

```hcl
# waf.tf
resource "aws_wafv2_web_acl" "main" {
  name        = "production-web-acl"
  description = "Production WAF - managed rules + custom rate limiting"
  scope       = "REGIONAL"  # 保護 ALB / API Gateway
  # scope = "CLOUDFRONT"    # 改這個保護 CloudFront（且必須部署在 us-east-1）

  # 預設動作：允許（僅規則命中才阻斷）
  # 若要「預設拒絕、白名單放行」改成 block {}
  default_action {
    allow {}
  }

  # 每條 Rule 的 visibility_config 控制 CloudWatch 指標
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ProductionWebAcl"
    sampled_requests_enabled   = true
  }

  # Rules 見下方各節
}
```

### 5.3 AWS 託管規則集（Managed Rule Groups）

AWS 維護並持續更新的規則集，涵蓋主流攻擊特徵，**開箱即用不需自己寫規則**：

```hcl
# 規則一：OWASP 通用規則集（最常用，建議所有環境都加）
rule {
  name     = "AWSManagedRulesCommonRuleSet"
  priority = 1

  override_action {
    none {}  # 使用規則集的預設動作（通常是 Block）
    # count {} # 改成 Count 模式：只記錄不阻斷，上線初期觀察用
  }

  statement {
    managed_rule_group_statement {
      name        = "AWSManagedRulesCommonRuleSet"
      vendor_name = "AWS"
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "CommonRuleSet"
    sampled_requests_enabled   = true
  }
}

# 規則二：SQL 注入防護
rule {
  name     = "AWSManagedRulesSQLiRuleSet"
  priority = 2

  override_action { none {} }

  statement {
    managed_rule_group_statement {
      name        = "AWSManagedRulesSQLiRuleSet"
      vendor_name = "AWS"
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "SQLiRuleSet"
    sampled_requests_enabled   = true
  }
}
```

**常用 AWS 託管規則一覽**：

| 規則集名稱 | 防禦目標 | 適用場景 |
|-----------|---------|---------|
| `AWSManagedRulesCommonRuleSet` | OWASP Top 10 通用攻擊（必加）| 所有 Web 應用 |
| `AWSManagedRulesSQLiRuleSet` | SQL 注入 | 有資料庫操作的應用 |
| `AWSManagedRulesKnownBadInputsRuleSet` | 已知惡意輸入特徵、Log4Shell 等 | 所有應用 |
| `AWSManagedRulesAmazonIpReputationList` | AWS 已知惡意 IP 資料庫 | 所有應用 |
| `AWSManagedRulesAnonymousIpList` | Tor、VPN、代理 IP | 需防匿名訪問的服務 |
| `AWSManagedRulesLinuxRuleSet` | Linux 伺服器攻擊（路徑遍歷、LFI）| Linux 後端 |
| `AWSManagedRulesBotControlRuleSet` | 惡意爬蟲、自動化攻擊工具 | 有爬蟲防護需求 |

### 5.4 自定義速率限制規則

```hcl
# 規則三：登入端點精確速率限制
rule {
  name     = "RateLimitLoginEndpoint"
  priority = 3

  action {
    block {}  # 觸發速率限制時直接阻斷（也可改為 count 先觀察）
  }

  statement {
    rate_based_statement {
      limit              = 100   # 每 5 分鐘同一 IP 最多 100 次請求
      aggregate_key_type = "IP"  # 以來源 IP 作為計數鍵

      # scope_down：只對符合此條件的請求計數
      # 不符合條件的請求不計入這個速率限制
      scope_down_statement {
        byte_match_statement {
          field_to_match {
            uri_path {}          # 比對 URI 路徑
          }
          positional_constraint = "STARTS_WITH"  # 路徑以此開頭
          search_string         = "/api/auth/login"
          text_transformation {
            priority = 0
            type     = "LOWERCASE"  # 先轉小寫再比對，防大小寫繞過
          }
        }
      }
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "RateLimitLogin"
    sampled_requests_enabled   = true
  }
}
```

**關鍵參數解析**：

| 參數 | 說明 |
|------|------|
| `limit` | 計算窗口（固定 **5 分鐘**）內同一 key 的最大請求數 |
| `aggregate_key_type` | 計數鍵類型：`IP`（來源 IP）、`FORWARDED_IP`（含 X-Forwarded-For）、`CUSTOM_KEYS`（自定義）|
| `scope_down_statement` | 縮小計數範圍：只有符合此條件的請求才計入速率 |
| `positional_constraint` | 字串比對方式：`EXACTLY`、`STARTS_WITH`、`ENDS_WITH`、`CONTAINS`、`CONTAINS_WORD` |

### 5.5 IP 封鎖清單

```hcl
# IP 封鎖清單（用於封鎖已知惡意 IP）
resource "aws_wafv2_ip_set" "blocklist" {
  name               = "manual-blocklist"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"

  addresses = [
    "192.0.2.0/24",    # 範例：封鎖整個 /24 段
    "203.0.113.5/32",  # 範例：封鎖單一 IP
  ]
}

# 在 Web ACL 中引用 IP Set
rule {
  name     = "BlockMaliciousIPs"
  priority = 0  # 最高優先級，最先評估

  action { block {} }

  statement {
    ip_set_reference_statement {
      arn = aws_wafv2_ip_set.blocklist.arn
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "BlocklistIPs"
    sampled_requests_enabled   = true
  }
}
```

### 5.6 `scope` 的選擇與部署限制

| `scope` 值 | 保護的 AWS 資源 | 部署 Region |
|-----------|--------------|------------|
| `REGIONAL` | ALB、API Gateway、AppSync | 任意 Region |
| `CLOUDFRONT` | CloudFront Distribution | **必須是 `us-east-1`** |

> ⚠️ **常見陷阱**：如果要保護 CloudFront，Terraform 的 AWS Provider 必須明確設定 `region = "us-east-1"`，否則部署失敗。

```hcl
# 保護 CloudFront 的 WAF 必須在 us-east-1 建立
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_wafv2_web_acl" "cloudfront_waf" {
  provider = aws.us_east_1
  scope    = "CLOUDFRONT"
  # ...
}
```

### 5.7 與 ALB 關聯

WAF Web ACL 建立後，需要關聯到實際的 AWS 資源才會生效：

```hcl
# 將 WAF 關聯到 ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn           # ALB 的 ARN
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

---

## 六、Count 模式 vs Enforce 模式（上線策略）

直接在生產環境啟用 Block 模式，可能誤傷正常流量。建議採用「先觀察、後封鎖」的漸進策略：

```
Phase 1：Count 模式（觀察期，1–2 週）
   override_action { count {} }
   → WAF 只記錄觸發的請求，不阻斷
   → 觀察 CloudWatch Metrics，確認誤報率
   → 分析取樣請求，調整規則

Phase 2：調整規則（針對誤報白名單）
   → 發現正常流量被誤報 → 新增 Rule Exception
   → 例如：排除特定路徑、特定 Header

Phase 3：切換 Enforce 模式（正式防禦）
   override_action { none {} }
   → 開始實際阻斷惡意請求
   → 持續監控 CloudWatch，設定告警
```

**Terraform 切換方式**（只改一行）：

```hcl
# Count 模式（觀察）
override_action { count {} }

# ↓ 改成 ↓

# Enforce 模式（阻斷）
override_action { none {} }
```

---

## 七、WAF 日誌與 SIEM 整合

### 7.1 啟用 WAF 日誌

```hcl
# 將 WAF 日誌輸出到 Kinesis Data Firehose → S3
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  log_destination_configs = [aws_kinesis_firehose_delivery_stream.waf_logs.arn]
  resource_arn            = aws_wafv2_web_acl.main.arn

  # 可選：在日誌中遮蔽敏感欄位（如 Authorization Header）
  redacted_fields {
    single_header {
      name = "authorization"
    }
  }
}
```

### 7.2 WAF 日誌欄位說明

WAF 日誌每條記錄包含：

| 欄位 | 說明 |
|------|------|
| `timestamp` | 請求時間 |
| `action` | `ALLOW` / `BLOCK` / `COUNT` |
| `terminatingRuleId` | 觸發最終動作的規則名稱 |
| `httpRequest.clientIp` | 來源 IP |
| `httpRequest.uri` | 請求路徑 |
| `httpRequest.headers` | 請求標頭 |
| `ruleGroupList` | 所有評估過的規則及各自動作 |

### 7.3 整合到 SIEM

```
WAF 日誌
  │
  ▼ Kinesis Firehose
  │
  ├──▶ S3（長期存檔，合規要求）
  │
  └──▶ Elasticsearch / OpenSearch
            │
            ▼
         Kibana Dashboard（即時視覺化）
            │
            ▼
         告警：當 BLOCK 率突增 → SNS 通知 → Slack / PagerDuty
```

**關鍵監控指標**：

| 指標 | 告警條件 | 意義 |
|------|---------|------|
| `BlockedRequests` | 突然大幅上升 | 可能遭受攻擊 |
| `CountedRequests` | 大量觸發但未阻斷 | Count 模式下的潛在威脅 |
| `AllowedRequests` 驟降 | 正常流量驟降 | 可能誤報大量阻斷正常用戶 |

---

## 八、完整 WAF Terraform 模組結構（最佳實踐）

```
waf/
├── main.tf          # Web ACL 主資源
├── ip_sets.tf       # IP 封鎖 / 白名單清單
├── rule_groups.tf   # 自定義規則群組（可複用）
├── logging.tf       # 日誌配置
├── variables.tf     # 可配置參數（環境、scope 等）
├── outputs.tf       # 輸出 Web ACL ARN 供其他模組引用
└── README.md        # 規則說明文件
```

**多環境部署範例**：

```hcl
# terraform/environments/staging/waf.tf
module "waf" {
  source      = "../../modules/waf"
  environment = "staging"
  scope       = "REGIONAL"

  # Staging 環境：所有規則用 Count 模式
  enforce_mode = false
}

# terraform/environments/prod/waf.tf
module "waf" {
  source      = "../../modules/waf"
  environment = "prod"
  scope       = "REGIONAL"

  # Prod 環境：強制執行阻斷
  enforce_mode = true
}
```

---

## 重點概念速查

- **Web ACL（Web Access Control List）**：AWS WAFv2 的頂層資源，包含多條規則，決定每個 HTTP 請求的命運（Allow / Block / Count）
- **Managed Rule Group**：AWS 維護的預建規則集，涵蓋主流攻擊特徵，開箱即用且持續更新
- **Priority**：規則評估順序，數字越小越先；第一條觸發的規則決定最終動作
- **Rate-Based Rule**：基於請求頻率的規則，計算窗口固定為 5 分鐘
- **scope_down_statement**：縮小速率限制的適用範圍（例如只對特定路徑生效）
- **Count 模式**：只記錄不阻斷，上線初期用於觀察誤報率
- **配置漂移（Configuration Drift）**：多環境 WAF 規則各自演化導致不一致，WAF-as-Code 的主要解決問題
- **虛擬補丁（Virtual Patching）**：新 CVE 出現時，在代碼修復前以 WAF 規則阻斷已知攻擊特徵
- **SIEM（Security Information and Event Management）**：集中收集、分析、告警安全日誌的平台（如 Splunk、Elastic Stack）
