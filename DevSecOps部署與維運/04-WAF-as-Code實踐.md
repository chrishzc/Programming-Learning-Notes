# 04 · WAF-as-Code 實踐

> **學習目標**：理解 WAF 在 L7 的防禦角色，並掌握以 Terraform 聲明式管理 WAF 規則的工程化方法。

---

## 學習框架

### 一、WAF vs 反向代理：防禦層次差異

| 面向 | 反向代理（Nginx）| WAF |
|------|----------------|-----|
| 運作層 | L4 傳輸 + 部分 L7 | L7 應用層語意 |
| 主要防禦 | 待補充 | 待補充 |
| 典型工具 | Nginx、HAProxy | AWS WAFv2、ModSecurity |

---

### 二、WAF 的核心防禦能力

> 待補充

- SQL 注入（SQLi）
- 跨站腳本（XSS）
- 機器人惡意爬取
- 零日漏洞攻擊（虛擬補丁）

---

### 三、傳統 WAF 維運的問題

> 待補充
>
> 提示：手動微調 → 配置漂移 → 環境不一致 → 部署延遲 → 設定後即遺忘

---

### 四、WAF-as-Code 架構

#### 4.1 核心理念

> 待補充
>
> 提示：WAF 規則 = 代碼 → 版本控制 → 可測試 → 可回滾 → CI/CD 自動部署

#### 4.2 工具選擇

| 工具 | 適用場景 |
|------|---------|
| Terraform + AWS WAFv2 | 待補充 |
| Terraform + GCP Cloud Armor | 待補充 |
| Terraform + Azure WAF | 待補充 |

---

### 五、AWS WAFv2 Web ACL 實作解析

#### 5.1 基本結構

```hcl
# 待補充：Web ACL 基本 Terraform 骨架解析
```

#### 5.2 託管規則集（Managed Rule Groups）

```hcl
# 待補充：引用 AWSManagedRulesCommonRuleSet
```

**常用 AWS 託管規則**：

| 規則集 | 防禦目標 |
|-------|---------|
| AWSManagedRulesCommonRuleSet | 待補充（OWASP 通用）|
| AWSManagedRulesSQLiRuleSet | 待補充 |
| AWSManagedRulesKnownBadInputsRuleSet | 待補充 |

#### 5.3 自定義速率限制規則

```hcl
# 待補充：針對 /api/v1/auth/login 的速率限制規則
```

**關鍵參數說明**：

- `limit`：待補充（計算週期為 5 分鐘）
- `aggregate_key_type`：待補充
- `scope_down_statement`：待補充

#### 5.4 `scope` 的選擇

| 值 | 適用場景 |
|----|---------|
| `REGIONAL` | 待補充（ALB / API Gateway）|
| `CLOUDFRONT` | 待補充 |

---

### 六、WAF 日誌與 SIEM 整合

> 待補充
>
> 提示：WAF 日誌 → Splunk / Elastic Stack → 即時告警 → 虛擬補丁生命週期對齊

---

### 七、虛擬補丁（Virtual Patching）概念

> 待補充
>
> 提示：新 CVE 發布 → WAF 規則阻斷 → 爭取修補代碼的時間視窗

---

## 重點概念速查

- **Web ACL（Web Access Control List）**：待補充
- **配置漂移（Configuration Drift）**：待補充
- **虛擬補丁**：待補充

---

## 延伸閱讀 / 參考資源

- [ ] AWS WAFv2 官方文件
- [ ] ModSecurity + OWASP CRS（開源方案）
- [ ] Terraform AWS WAF Provider 文件
