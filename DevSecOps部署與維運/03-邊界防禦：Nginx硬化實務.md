# 03 · 邊界防禦：反向代理選型與硬化實務

> **學習目標**：理解反向代理在 DevSecOps 架構中的安全定位，掌握主流方案的選型邏輯，並以 Nginx 為例學習具體硬化配置。

---

## 一、反向代理的角色與流量全貌

### 1.1 什麼是反向代理？與正向代理的差異

**代理（Proxy）** 就是「中間人」，差別在於它替誰隱藏：

| | 正向代理（Forward Proxy） | 反向代理（Reverse Proxy） |
|--|--------------------------|--------------------------|
| **代理對象** | 客戶端（Client） | 伺服器（Server） |
| **誰不知道真實身份** | 伺服器不知道真實 Client IP | Client 不知道真實後端架構 |
| **典型用途** | VPN、企業內網上網管控 | Nginx、CDN、API Gateway |
| **Client 是否感知** | 需手動設定代理 | 完全透明 |

```
【正向代理】                         【反向代理】
Client → [Proxy] → Internet          Internet → [Proxy] → Backend A
                                                         → Backend B
                                                         → Backend C
```

反向代理坐在**公網入口**，對外是系統的唯一門面，對內可連接任意數量後端服務。

---

### 1.2 流量路徑全貌

```
瀏覽器 / API Client
      │ HTTPS (TLS 1.3)
      ▼
┌──────────────────────────────────────────┐
│              反向代理（邊界層）            │
│                                          │
│  1. TLS 握手 → 解密請求                   │
│  2. 速率限制檢查 → 超額 → 429            │
│  3. IP 黑名單    → 命中 → 403            │
│  4. 請求大小檢查 → 超限 → 413            │
│  5. Header 合法性驗證                     │
│  6. 路由規則 → 決定轉發到哪個 upstream    │
│  7. 注入 X-Real-IP / X-Forwarded-For    │
│  8. 轉發（HTTP，內網明文）→ 後端          │
│  9. 收到後端 Response                    │
│  10. 注入安全標頭（CSP / HSTS...）        │
│  11. TLS 加密 → 回傳 Client              │
└──────────────────────────────────────────┘
      │ HTTP（受信任的內部網路）
      ▼
   Backend Service（Node.js / Python / Go...）
```

**關鍵洞察**：反向代理是**唯一對公網暴露的攻擊面**。後端服務永遠不應對公網直接開放 Port。

---

### 1.3 反向代理的七大安全職責

| 安全職責       | 機制                   | 防禦的攻擊類型                  |
| ---------- | -------------------- | ------------------------ |
| **TLS 終端** | 憑證集中管理、協定版本控制        | 降級攻擊、竊聽、MITM             |
| **拓撲隱藏**   | 隱藏後端 IP/Port/框架版本    | 偵察、CVE 定向攻擊              |
| **流量過濾**   | IP 封鎖、Method 限制、路徑規則 | 路徑掃描、惡意爬蟲                |
| **DoS 緩解** | 速率限制、連線數限制           | 暴力破解、洪水攻擊                |
| **緩衝區控制**  | 請求大小限制、逾時設定          | Slowloris、大型 Payload 攻擊  |
| **安全標頭注入** | 統一注入 CSP、HSTS 等標頭    | XSS、Clickjacking、MIME 探測 |
| **高可用保障**  | 後端健康檢查、自動摘除故障節點      | DDoS 導致全服務中斷             |

> **Nginx 不是萬能的**：邏輯漏洞（SQL Injection、IDOR、業務邏輯缺陷）反向代理無法攔截，那是應用層的責任。

---

### 1.4 TLS 協定：HTTPS 的加密基礎

理解反向代理的 TLS 終端之前，先搞清楚 TLS 本身是什麼。

#### 1.4.1 什麼是 TLS？

**TLS（Transport Layer Security）** 是在傳輸層上提供加密、身份驗證、完整性校驗的協定。HTTPS = HTTP + TLS。

```
HTTP（明文）：  你的密碼 → [網路] → 伺服器        ← 任何人都能看到
HTTPS（加密）：  你的密碼 → [TLS加密] → [網路] → [TLS解密] → 伺服器
```

TLS 同時解決了三個問題：

| 問題 | TLS 解法 | 防禦的攻擊 |
|------|---------|----------|
| **竊聽**（Eavesdropping） | 對稱加密傳輸內容 | MITM 竊取密碼、Cookie |
| **身份冒充**（Impersonation） | 數位憑證 + CA 驗證伺服器身份 | 釣魚網站、偽造伺服器 |
| **內容篡改**（Tampering） | MAC / AEAD 完整性校驗 | 注入惡意腳本、修改封包 |

---

#### 1.4.2 SSL → TLS 版本演進

「SSL」和「TLS」常被混用，但 SSL 是舊名，現在實際使用的都是 TLS：

| 版本 | 年份 | 現況 | 原因 |
|------|------|------|------|
| SSL 2.0 | 1995 | ❌ 廢棄 | 根本性設計缺陷 |
| SSL 3.0 | 1996 | ❌ 廢棄 | POODLE 攻擊（2014），RFC 7568 |
| TLS 1.0 | 1999 | ❌ 廢棄 | BEAST 攻擊，RFC 8996（2021） |
| TLS 1.1 | 2006 | ❌ 廢棄 | 同受 POODLE 影響，RFC 8996 |
| TLS 1.2 | 2008 | ✅ 仍廣泛使用 | 配合安全 Cipher 仍足夠 |
| TLS 1.3 | 2018 | ✅ 推薦 | 大幅簡化，移除所有已知弱演算法 |

> **POODLE（2014）**：攻擊者強迫瀏覽器降級到 SSL 3.0，再利用 CBC Padding Oracle 逐 byte 解密 Session Cookie。  
> **BEAST（2011）**：TLS 1.0 的 CBC 模式 IV 可預測，攻擊者可解密特定已知明文的 Session 資料。

---

#### 1.4.3 對稱加密 vs 非對稱加密

TLS 結合了兩種加密方式，各自承擔不同職責：

| | 非對稱加密 | 對稱加密 |
|--|-----------|---------|
| **金鑰** | 公鑰（加密）+ 私鑰（解密） | 同一把金鑰加解密 |
| **速度** | 慢（數學運算複雜） | 快（適合大量資料） |
| **用途** | 握手期間安全交換金鑰 | 握手後加密實際傳輸內容 |
| **演算法例子** | RSA、ECDHE | AES-GCM、ChaCha20 |

**TLS 的設計邏輯**：
1. 用**非對稱加密**完成握手，安全地協商出一個共享的**對話金鑰（Session Key）**
2. 後續所有資料傳輸改用**對稱加密**（速度快），使用剛才協商好的 Session Key

---

#### 1.4.4 TLS 握手過程（以 TLS 1.3 為例）

```
Client                                  Server
  │                                        │
  │── ClientHello ──────────────────────▶  │
  │   (支援的協定版本、Cipher 清單、        │
  │    ECDHE 公鑰)                          │
  │                                        │
  │  ◀──────────────────── ServerHello ───  │
  │   (選定協定版本、選定 Cipher、           │
  │    ECDHE 公鑰、SSL 憑證)                │
  │                                        │
  │  [雙方各自用對方公鑰 + 自己私鑰          │
  │   透過 ECDHE 推導出相同的 Session Key]  │
  │                                        │
  │  ◀────────────────── {加密的 Finished}  │
  │── {加密的 Finished} ─────────────────▶  │
  │                                        │
  │═══════════════ 後續全程加密傳輸 ════════│
```

**TLS 1.3 vs TLS 1.2 握手差異**：

| | TLS 1.2 | TLS 1.3 |
|--|---------|---------|
| **握手來回次數** | 2-RTT | **1-RTT**（快一倍） |
| **0-RTT 恢復** | ❌ | ✅（Session 復用可達 0-RTT） |
| **允許的 Cipher** | 包含 RSA、RC4 等弱套件 | **只允許 AEAD 套件**（AES-GCM、ChaCha20） |
| **前向保密** | 可選 | **強制**（只允許 ECDHE） |

---

#### 1.4.5 數位憑證與 CA 信任鏈

TLS 能防身份冒充，靠的是**數位憑證（Digital Certificate）**。

```
你的瀏覽器信任的根 CA（Root CA）
        │ 簽發
        ▼
   中間 CA（Intermediate CA）
        │ 簽發
        ▼
   網站憑證（example.com 的 SSL 憑證）
        │ 証明
        ▼
   這個伺服器確實是 example.com
```

憑證裡包含：
- 網域名稱（Subject）
- 公鑰
- 有效期限
- CA 的數位簽章

瀏覽器驗證邏輯：
1. 伺服器送出憑證
2. 瀏覽器檢查憑證的 CA 簽章 → 是否在信任的 Root CA 清單中？
3. 檢查網域名稱是否符合
4. 檢查憑證是否在有效期內、是否被吊銷（OCSP）
5. 全部通過 → 🔒 顯示安全鎖頭

**憑證類型**：

| 類型 | 驗證程度 | 適合對象 |
|------|---------|---------|
| **DV（Domain Validated）** | 只驗證你控制該域名 | 個人網站、API 服務 |
| **OV（Organization Validated）** | 驗證企業真實存在 | 企業官網 |
| **EV（Extended Validation）** | 最嚴格的企業審查 | 銀行、金融機構 |
| **Wildcard（`*.example.com`）** | 覆蓋所有子域名 | 多子域服務 |

> **Let's Encrypt** 提供免費的 DV 憑證，90 天自動續期，是目前最常見的選擇。

---

### 1.5 TLS 終端：核心設計的取捨

```
Client ──HTTPS──▶ [反向代理] ──HTTP──▶ Backend
                   (加解密在此)
```

**好處**：憑證集中管理、後端 CPU 減負、TLS 版本升級只改一處。  
**風險**：代理到後端這段是明文，若攻擊者進入內網可竊聽。

**緩解方式**：
- 確保代理與後端在同一個受信任私有網路（VPC / K8s 內部）
- 高安全要求場景改用 **End-to-End TLS**（代理→後端也走 HTTPS）

---

## 二、主流反向代理方案比較

### 2.1 選型的四個維度

在看具體工具前，先釐清自己的情境：

| 維度 | 要問的問題 |
|------|----------|
| **部署環境** | 裸機 / Docker / K8s / 純雲端托管？ |
| **主要用途** | Web 伺服器 / API Gateway / Service Mesh / CDN？ |
| **運維能力** | 團隊能接受多高的配置複雜度？ |
| **預算** | 能接受付費方案或完全要免費？ |

---

### 2.2 七大方案逐一解析

#### 🔷 Nginx

| 項目 | 說明 |
|------|------|
| **定位** | 高性能 HTTP 伺服器 + 反向代理 + 負載均衡器 |
| **適合環境** | 裸機、VM、Docker、K8s（Ingress Controller） |
| **Layer** | L7（HTTP），搭配 stream 模組可做 L4 |
| **配置語言** | 宣告式 `nginx.conf`（非常穩定但靜態，改完要 reload） |
| **自動 HTTPS** | ❌ 需手動配合 Certbot |
| **服務發現** | ❌ 靜態 upstream，需搭配 Consul 或 K8s 自動化 |
| **費用** | ✅ 開源免費 / Nginx Plus 約 $2,500–$3,500/年/實例 |
| **學習曲線** | 中等（配置語法直觀，但 `if` 指令有坑） |
| **最大優點** | 極度穩定、社群成熟、文件豐富、資源占用極低 |
| **最大缺點** | 靜態配置，動態場景（微服務自動發現）需額外工具 |

**最適場景**：傳統 Web 應用、需要精細控制配置的中小型系統、K8s Ingress。

---

#### 🔷 HAProxy

| 項目 | 說明 |
|------|------|
| **定位** | 業界最高性能的 L4/L7 負載均衡器 |
| **適合環境** | 裸機、VM，對 K8s 支援不如 Traefik 原生 |
| **Layer** | **L4（TCP）+ L7（HTTP）** 雙模式，這是 Nginx 沒有的 |
| **配置語言** | `haproxy.cfg`（語法獨特，需學習成本） |
| **自動 HTTPS** | ❌ 需手動配合 Certbot |
| **服務發現** | 部分支援（Runtime API 可動態更新 backend） |
| **費用** | ✅ 開源免費 / HAProxy Enterprise 約 $2,500+/年 |
| **學習曲線** | 較高（L4/L7 概念混合，ACL 語法複雜） |
| **最大優點** | 極致性能、L4 TCP 代理能力、Health Check 最完整 |
| **最大缺點** | 不做 Web 伺服器（無法直接 serve 靜態檔案）、配置語法學習成本高 |

**最適場景**：資料庫代理（MySQL、PostgreSQL TCP 代理）、金融級高流量 L4 負載均衡、需要最細粒度健康檢查的場景。

> **Nginx vs HAProxy 選哪個？**  
> - 只需要 HTTP/HTTPS → **Nginx**  
> - 需要同時代理 TCP（資料庫、MQ）→ **HAProxy**  
> - 追求最極致 TPS 性能 → **HAProxy**

---

#### 🔷 Traefik

| 項目 | 說明 |
|------|------|
| **定位** | 雲原生 Edge Router，為 Docker / K8s 而生 |
| **適合環境** | Docker Compose、K8s（首選 Ingress Controller 之一） |
| **Layer** | L7（HTTP），支援 TCP/UDP |
| **配置語言** | YAML / TOML，或完全**自動發現**（Label / Annotation 驅動） |
| **自動 HTTPS** | ✅ **內建 Let's Encrypt，自動申請與續期** |
| **服務發現** | ✅ **自動掃描 Docker Labels / K8s Annotations，服務上線即生效** |
| **費用** | ✅ 開源免費 / Traefik Enterprise 付費 |
| **學習曲線** | 低（Docker 環境幾乎零配置）|
| **最大優點** | 動態配置不需 reload、Auto HTTPS、容器化首選 |
| **最大缺點** | 性能不如 Nginx/HAProxy、靜態場景配置反而更繁瑣 |

**最適場景**：Docker Compose 微服務、K8s Ingress（與 cert-manager 配合）、快速原型開發。

```yaml
# Docker Compose 範例：只需加 Label，Traefik 自動配置路由與 HTTPS
services:
  myapp:
    image: my-api:latest
    labels:
      - "traefik.http.routers.myapp.rule=Host(`api.example.com`)"
      - "traefik.http.routers.myapp.tls.certresolver=le"  # 自動 Let's Encrypt
```

---

#### 🔷 Caddy

| 項目 | 說明 |
|------|------|
| **定位** | 現代化 Web 伺服器，以「自動 HTTPS」為核心設計 |
| **適合環境** | 裸機、VM、Docker，個人 / 小型團隊 |
| **Layer** | L7（HTTP） |
| **配置語言** | `Caddyfile`（極簡，可讀性最高） |
| **自動 HTTPS** | ✅ **最完整：自動申請、自動續期、自動 OCSP Stapling** |
| **服務發現** | ❌（靜態，但有 API 可動態調整） |
| **費用** | ✅ 完全免費開源（Apache 2.0） |
| **學習曲線** | **最低**（Caddyfile 可讀性遠超 nginx.conf） |
| **最大優點** | Auto HTTPS 開箱即用、配置極簡、安全預設值最好 |
| **最大缺點** | 生態系與社群成熟度不如 Nginx、性能略遜 |

**最適場景**：個人專案、側邊服務、快速部署需要 HTTPS 的小服務、不想手動管憑證的場景。

```
# Caddyfile 範例：5 行搞定 HTTPS 反向代理（Nginx 要 40+ 行）
api.example.com {
    reverse_proxy localhost:8080
}
# Caddy 自動處理：Let's Encrypt 申請、HTTP→HTTPS 跳轉、HSTS 標頭
```

---

#### 🔷 Envoy

| 項目 | 說明 |
|------|------|
| **定位** | CNCF 專案，Service Mesh 的資料平面（Data Plane） |
| **適合環境** | K8s，通常作為 Sidecar（Istio / Linkerd 的底層） |
| **Layer** | L3/L4/L7 全支援，gRPC 原生支援 |
| **配置語言** | xDS API（JSON/YAML，通常由 Control Plane 生成，不手寫） |
| **自動 HTTPS** | ✅（由 Istio 等 Control Plane 管理 mTLS） |
| **服務發現** | ✅ **動態 xDS，天生為微服務設計** |
| **費用** | ✅ 完全免費開源 |
| **學習曲線** | **最高**（幾乎不直接操作，透過 Istio 使用） |
| **最大優點** | 服務間 mTLS 全自動、可觀測性最強（分散式追蹤）、gRPC 原生 |
| **最大缺點** | 極度複雜，學習曲線陡峭，單獨使用幾乎不可行 |

**最適場景**：K8s Service Mesh（搭配 Istio）、需要服務間 mTLS、分散式追蹤的微服務架構。

---

#### 🔷 雲端托管：AWS ALB / GCP Load Balancer

| 項目 | 說明 |
|------|------|
| **定位** | 雲端平台原生的 L7 負載均衡器 |
| **適合環境** | 全雲端（AWS / GCP / Azure） |
| **配置方式** | 雲端 Console / Terraform / CDK |
| **自動 HTTPS** | ✅ 配合 ACM（AWS）/ Google-managed 憑證自動管理 |
| **服務發現** | ✅ 自動整合雲端服務（ECS、EKS、GKE） |
| **費用** | **依流量計費**：AWS ALB 約 $16–100+/月（依流量與 LCU 數量） |
| **最大優點** | 零運維（全托管）、自動擴展、原生整合雲端 IAM / WAF |
| **最大缺點** | 費用依流量浮動、供應商鎖定（難遷移）、自訂能力有限 |

**費用估算（AWS ALB）**：
```
基本費：$0.008/LCU-hour
固定費：~$16/月
中型服務（1000 萬請求/月）：約 $30–60/月
高流量服務（10 億請求/月）：可能超過 $500/月
```

**最適場景**：已在雲端的服務、不想自管基礎設施、有 Auto Scaling 需求。

---

#### 🔷 Cloudflare（CDN + 反向代理）

| 項目 | 說明 |
|------|------|
| **定位** | 全球 CDN + 反向代理 + DDoS 防護 + WAF |
| **部署方式** | 只需把 DNS 指向 Cloudflare，完全不改後端 |
| **自動 HTTPS** | ✅ 全自動，且有免費 Wildcard 憑證 |
| **DDoS 防護** | ✅ **無限 DDoS 緩解，這是其他方案無法比的** |
| **費用** | Free / Pro $20/月 / Business $200/月 / Enterprise 洽談 |
| **最大優點** | 設置最簡單（只改 DNS）、全球 CDN 加速、DDoS 防護強悍 |
| **最大缺點** | 流量過 Cloudflare（需信任第三方）、免費版功能有限、企業版貴 |

**免費版 vs 付費版差異**：

| 功能 | Free | Pro ($20/月) | Business ($200/月) |
|------|------|-------------|------------------|
| DDoS 防護 | ✅ 無限 | ✅ 無限 | ✅ 無限 |
| WAF 規則 | 5 條 | 20 條 | 100 條 |
| Bot 管理 | 基本 | 進階 | 企業級 |
| 自訂錯誤頁 | ❌ | ✅ | ✅ |
| 自訂快取規則 | 基本 | 進階 | 完整 |

**最適場景**：公開 Web 服務、需要 DDoS 防護、個人或小型團隊（Free 方案極具性價比）。

---

### 2.3 方案比較總表

| 方案 | 性能 | 自動 HTTPS | 動態配置 | K8s 友好 | 費用 | 複雜度 |
|------|------|-----------|---------|---------|------|-------|
| **Nginx** | ⭐⭐⭐⭐⭐ | ❌ 手動 | ❌ 靜態 | ⭐⭐⭐ | 免費 | 中 |
| **HAProxy** | ⭐⭐⭐⭐⭐ | ❌ 手動 | 部分 | ⭐⭐ | 免費 | 高 |
| **Traefik** | ⭐⭐⭐⭐ | ✅ 內建 | ✅ 自動發現 | ⭐⭐⭐⭐⭐ | 免費 | 低（K8s）|
| **Caddy** | ⭐⭐⭐ | ✅ 最完整 | 部分 | ⭐⭐ | 免費 | **最低** |
| **Envoy** | ⭐⭐⭐⭐⭐ | ✅（mTLS） | ✅ xDS | ⭐⭐⭐⭐⭐ | 免費 | **最高** |
| **AWS ALB** | ⭐⭐⭐⭐ | ✅ ACM | ✅ 雲端整合 | ⭐⭐⭐⭐ | $16–500+/月 | 低 |
| **Cloudflare** | ⭐⭐⭐⭐⭐ | ✅ 全自動 | ✅ | ⭐⭐ | 免費～$200+/月 | **最低** |

---

### 2.4 選型決策流程

```
需要代理 TCP（資料庫、MQ）？
├── YES → HAProxy
└── NO ↓

已在 AWS / GCP / Azure，且不想自管？
├── YES → 雲端 ALB / Load Balancer
└── NO ↓

需要全球 CDN + DDoS 防護（且能接受第三方）？
├── YES → Cloudflare
└── NO ↓

部署在 K8s，需要自動服務發現？
├── YES → Traefik（或 Nginx Ingress + cert-manager）
└── NO ↓

個人 / 小型服務，最重視省事？
├── YES → Caddy
└── NO ↓

高流量、精細控制、團隊有 Nginx 經驗？
└── Nginx
```

---

### 2.5 成本比較（月費估算，中型服務）

| 方案 | 月費 | 備註 |
|------|------|------|
| Nginx | $0（軟體費）+ VM 費用 | 1 core / 1GB RAM 足以處理千萬請求 |
| HAProxy | $0（軟體費）+ VM 費用 | 同上 |
| Traefik | $0（軟體費）+ VM/K8s 費用 | K8s cluster 成本另計 |
| Caddy | $0 | 完全免費 |
| AWS ALB | $30–100 | 視 LCU 數量浮動 |
| GCP Load Balancer | $20–80 | 視規則與流量 |
| Cloudflare Free | $0 | 免費版功能已相當完整 |
| Cloudflare Pro | $20 | 推薦大多數小型商業服務 |

> **ponytail：** 大多數情況下，Cloudflare Free + 一台跑 Nginx/Caddy 的 VM，是成本效益最高的搭配。Cloudflare 擋 DDoS 和提供 CDN，Nginx 在後端精細控制，雙層防禦但費用幾乎為零。

---

## 三、Nginx 硬化配置參考

> 以下以 Nginx 為例，展示反向代理的硬化實踐。其他工具的安全配置邏輯相同，語法不同。

### 3.1 核心硬化項目

#### 版本資訊隱藏
```nginx
server_tokens off;  # Server 標頭只回傳 "nginx"，不含版本號
proxy_hide_header X-Powered-By;  # 隱藏後端框架標頭
```

#### 緩衝區防禦（防 DoS / Slowloris）
```nginx
client_body_buffer_size      16k;
client_max_body_size          1m;   # 超出回 413
client_header_buffer_size     1k;
large_client_header_buffers  4 8k;

client_body_timeout   12s;
client_header_timeout 12s;
keepalive_timeout     15s;
send_timeout          10s;
```

#### TLS 協定硬化
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;
```

| 協定 | 狀態 | 原因 |
|------|------|------|
| TLS 1.0 | ❌ 禁用 | BEAST 攻擊、POODLE 降級 |
| TLS 1.1 | ❌ 禁用 | RFC 8996 宣告棄用 |
| TLS 1.2 | ✅ 允許 | 廣泛支援，配合安全 Cipher 足夠 |
| TLS 1.3 | ✅ 允許 | 移除舊式危險 Cipher，握手更快 |

**BEAST（2011）**：TLS 1.0 CBC 模式 IV 可預測，可解密 Session Cookie  
**POODLE（2014）**：強制降級到 SSLv3/TLS 1.0 後利用 Padding Oracle 解密

#### 安全標頭注入
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Frame-Options           "SAMEORIGIN"                          always;
add_header X-Content-Type-Options    "nosniff"                             always;
add_header Referrer-Policy           "strict-origin-when-cross-origin"     always;
add_header Content-Security-Policy   "default-src 'self'; object-src 'none';" always;
```

| 標頭 | 防禦目標 |
|------|---------|
| `Strict-Transport-Security` | HTTP 降級攻擊（HSTS） |
| `X-Frame-Options` | 點擊劫持（Clickjacking） |
| `X-Content-Type-Options` | MIME 型別探測 |
| `Referrer-Policy` | Referrer URL 資訊洩露 |
| `Content-Security-Policy` | XSS、資料注入 |

#### 速率限制
```nginx
# http 區塊
limit_req_zone $binary_remote_addr zone=login_zone:10m rate=5r/m;
limit_req_zone $binary_remote_addr zone=api_zone:10m   rate=30r/s;

# location 區塊
location /api/auth/login {
    limit_req zone=login_zone burst=3 nodelay;
    limit_req_status 429;
}
```

#### SSL Session 優化
```nginx
ssl_session_cache   shared:SSL:10m;  # 10MB ≈ 4 萬條 Session，跨 Worker 共享
ssl_session_timeout 1d;              # 1 天內重連可跳過完整握手
```

---

### 3.2 完整硬化配置範本

```nginx
# /etc/nginx/nginx.conf
user  nginx;
worker_processes auto;

events { worker_connections 1024; }

http {
    server_tokens off;

    # 緩衝區防禦
    client_body_buffer_size      16k;
    client_max_body_size          1m;
    client_header_buffer_size     1k;
    large_client_header_buffers  4 8k;

    # 逾時（防 Slowloris）
    client_body_timeout   12s;
    client_header_timeout 12s;
    keepalive_timeout     15s;
    send_timeout          10s;

    # 速率限制定義
    limit_req_zone $binary_remote_addr zone=login_zone:10m rate=5r/m;
    limit_req_zone $binary_remote_addr zone=api_zone:10m   rate=30r/s;

    server {
        listen 443 ssl http2;
        server_name example.com;

        ssl_certificate     /etc/ssl/certs/example.com.crt;
        ssl_certificate_key /etc/ssl/private/example.com.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
        ssl_prefer_server_ciphers off;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 1d;

        proxy_hide_header X-Powered-By;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options           "SAMEORIGIN"                          always;
        add_header X-Content-Type-Options    "nosniff"                             always;
        add_header Referrer-Policy           "strict-origin-when-cross-origin"     always;
        add_header Content-Security-Policy   "default-src 'self'; object-src 'none';" always;

        location /api/auth/login {
            limit_req zone=login_zone burst=3 nodelay;
            limit_req_status 429;
            proxy_pass http://backend;
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /api/ {
            limit_req zone=api_zone burst=50 nodelay;
            limit_req_status 429;
            proxy_pass http://backend;
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name example.com;
        return 301 https://$host$request_uri;
    }
}
```

---

### 3.3 驗證指令

```bash
# 確認 Server 標頭已隱藏版本號
curl -sI https://example.com | grep -i server

# 確認舊 TLS 協定已拒絕（應連線失敗）
openssl s_client -connect example.com:443 -tls1
openssl s_client -connect example.com:443 -tls1_1

# 全面 TLS 掃描（含 BEAST/POODLE 檢測）
bash testssl.sh https://example.com

# 確認安全標頭存在
curl -sI https://example.com | grep -iE "strict-transport|x-frame|content-security"

# 驗證速率限制（超過應回 429）
for i in {1..10}; do curl -s -o /dev/null -w "%{http_code}\n" https://example.com/api/auth/login; done
```

---

## 四、重點概念速查

- **縱深防禦（Defense in Depth）**：反向代理 + WAF + 應用層 + DB 層各自設防，攻擊者必須突破多層
- **完美前向保密（PFS）**：每次 Session 使用臨時 DH 金鑰，金鑰不保存，事後私鑰外洩也無法解密歷史流量
- **HSTS**：瀏覽器記住「此域名只用 HTTPS」，防止首次連線被降級後遭 MITM
- **TLS 終端 vs End-to-End TLS**：前者在代理解密，後者代理→後端也加密，高安全場景選後者
- **L4 vs L7 代理**：L4 看 IP+Port，不解析 HTTP 內容（適合 TCP 代理）；L7 解析 HTTP，能做路徑路由和 Header 操作

---

## 五、延伸閱讀

- [ ] [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) — 自動產生 Nginx / HAProxy / Caddy 的安全配置
- [ ] [testssl.sh](https://testssl.sh/) — 本地端 TLS 掃描工具
- [ ] [securityheaders.com](https://securityheaders.com/) — 線上掃描 Response 標頭評分
- [ ] [Traefik 官方文件](https://doc.traefik.io/traefik/) — K8s 動態路由參考
- [ ] RFC 8996 — TLS 1.0 / 1.1 正式棄用說明
- [ ] [Cloudflare Learning Center](https://www.cloudflare.com/learning/) — 反向代理與 DDoS 防護概念
