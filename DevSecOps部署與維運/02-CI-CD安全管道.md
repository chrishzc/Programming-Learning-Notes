# 02 · CI/CD 安全管道

> **學習目標**：理解 CI/CD 管道的核心概念，掌握如何在管道各階段嵌入安全工具，實現「安全左移」的自動化落地。
>
> **定位**：CI/CD 是 DevSecOps 的骨幹，所有安全工具（SAST、SCA、簽章、DAST）最終都需透過管道自動執行才有意義。

---

## 學習框架

### 一、概念釐清：CI / CD / CD

現代軟體交付流程由三個層次組成，縮寫相似但職責不同：

| 縮寫     | 全名                           | 核心目標                                                | 觸發時機      |
| ------ | ---------------------------- | --------------------------------------------------- | --------- |
| **CI** | Continuous Integration（持續整合） | 每次代碼提交後，自動執行建置與測試，確保主線代碼始終可用==(PR前檢查)==             | Push / PR |
| **CD** | Continuous Delivery（持續交付）    | CI 通過後自動將產物推進至「可隨時一鍵部署」的狀態，但**發布需人工確認**==(手動更新版本)== | CI 成功後    |
| **CD** | Continuous Deployment（持續部署）  | 完全自動化，CI/CD 通過後**無需人工**直接部署至生產==(偵測有新版本後自動更新)==     | CD 成功後    |

**三者的邊界**：

```
代碼提交
   │
   ▼
【CI 範圍】─── 建置 → 測試 → 安全掃描 → 產物打包
   │
   ▼
【Continuous Delivery】─── 部署至 Staging → 整合測試 → 等待人工放行
   │
   ▼
【Continuous Deployment】─── 自動部署至 Production（高信任度管道才適用）
```

> 💡 **實務建議**：多數企業採用 Continuous Delivery（而非全自動 Deployment），保留生產部署的人工 Checkpoint，風險可控。

---

### 二、Pipeline as Code

#### 2.1 為什麼要把 Pipeline 寫成代碼？

傳統 CI 系統（如老版 Jenkins）用 GUI 點選設定管道步驟，帶來以下問題：

- **無版本歷史**：改了什麼、誰改的，完全不可追溯
- **環境不一致**：Dev、Staging、Prod 的 Pipeline 各自獨立維護，悄悄漂移
- **無法 Review**：安全設定的變更不走 PR，等於繞過代碼審查

**Pipeline as Code 的核心價值**：

| 優點   | 說明                                          |
| ---- | ------------------------------------------- |
| 版本控制 | Pipeline 定義與應用代碼放同一個 Repo，`git log` 可追溯所有變更 |
| 可審計  | 所有 Pipeline 變更都需走 PR，讓人工 Review 安全設定成為常態    |
| 環境一致 | 同一份 YAML，在任何 Runner 上執行結果相同                 |
| 可測試  | Pipeline 本身可以有 Lint 檢查（如 `actionlint`）      |

#### 2.2 主流工具比較

| 工具                     | 託管方式                | 語法格式                 | 最適場景                          |
| ---------------------- | ------------------- | -------------------- | ----------------------------- |
| ==**GitHub Actions**== | SaaS                | YAML                 | 代碼托管在 GitHub、需要大量社群 Action 生態 |
| **GitLab CI/CD**       | Self-hosted / SaaS  | YAML                 | 企業自建 GitLab、需要完整 DevOps 平台    |
| **Jenkins**            | Self-hosted         | Groovy (Jenkinsfile) | 老牌企業環境、高度自定義需求                |
| **Tekton**             | K8s Native          | YAML (CRD)           | 已深度使用 K8s、需要雲原生管道             |
| **ArgoCD**             | K8s Native (GitOps) | YAML                 | CD 端 GitOps 模式（代碼即部署狀態）       |

#### 2.3 GitHub Actions 核心結構解析

```yaml
# .github/workflows/ci-security.yml
name: CI Security Pipeline          # 管道顯示名稱

on:                                  # 觸發條件
  push:
    branches: [main]                 # 主線 push 觸發
  pull_request:
    branches: [main]                 # PR 到主線時觸發（最重要的安全門）

jobs:
  security-scan:                     # Job 名稱（可平行執行多個 Job）
    runs-on: ubuntu-latest           # Runner 環境

    permissions:                     # 最小化 GITHUB_TOKEN 權限（安全最佳實踐）
      contents: read
      security-events: write         # 允許上傳 SARIF 報告到 Security tab

    steps:
      - name: Checkout code
        uses: actions/checkout@v4    # uses = 複用社群/官方 Action
        with:
          fetch-depth: 0             # 取完整 git history（gitleaks 需要）

      - name: Run SAST
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/owasp-top-ten

      # 更多 Step 見第三節
```

**核心欄位說明**：

| 欄位 | 說明 |
|------|------|
| `on` | 觸發條件，支援 push、pull_request、schedule、workflow_dispatch 等 |
| `jobs` | 一個 Workflow 可有多個 Job，預設**平行執行**，用 `needs` 設定依賴關係 |
| `steps` | Job 內的每個動作，**串行執行**，任一 Step 失敗則 Job 失敗 |
| `uses` | 引用外部 Action，格式為 `owner/repo@version`，版本應釘定 commit hash（安全考量）|
| `env` | 環境變數，明文即可 |
| `secrets` | 機敏資訊，從 GitHub Secrets 注入，不會出現在 log 中 |
| `permissions` | 限制 `GITHUB_TOKEN` 的最小權限，預設值過於寬鬆 |

> ⚠️ **安全注意**：`uses` 應釘定完整 commit hash 而非 tag（如 `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`），防止 tag 被惡意移動指向有毒版本（Supply Chain Attack）。

##### 補充

| **環境名稱**    | **全名**                                 | **主要使用者**             | **核心目的**                                | **資料來源**           |
| ----------- | -------------------------------------- | --------------------- | --------------------------------------- | ------------------ |
| **Dev**     | Development<br>開發環境                    | 軟體工程師 (Developers)    | **撰寫與實驗程式碼**<br>進行初步的功能測試與單元測試。         | 假的測試資料 (Mock Data) |
| **Staging** | Staging / Pre-Production<br>預發布 / 階段環境 | QA 測試員、產品經理 (PM)、客戶   | **模擬真實環境測試**<br>在上線前的最後驗收（UAT），驗證功能與效能。 | 脫敏（去標識化）後的真實資料複製版  |
| **Prod**    | Production<br><br>正式 / 營運環境            | **真實使用者 (End Users)** | **提供服務**<br>確保高可用性、穩定度與安全性。             | 最真實的商業資料           |

---

### 三、安全工具嵌入地圖（Security Gates）

這是整篇筆記的核心：**在管道哪個節點嵌入哪個工具、設定什麼阻斷條件**。

```
開發者本機
   │
   ▼
[pre-commit hooks]  ← 本機即時反饋，秒級
   ├── 機敏資訊掃描（gitleaks）      阻斷：任何命中
   ├── 代碼格式 Lint
   └── 基礎語法檢查

   │  git push
   ▼
[PR / Merge Request 觸發]  ← 主要安全門禁
   ├── SAST 靜態分析（Semgrep/CodeQL）  阻斷：High/Critical
   ├── SCA 依賴掃描（Trivy/Snyk）       阻斷：CVSS ≥ 9.0（可調整）
   └── 機敏資訊掃描（gitleaks）         阻斷：任何命中

   │  PR Merged → main
   ▼
[Build 構建]
   ├── 編譯 / 打包應用程式
   ├── 建置 Container Image
   ├── 容器鏡像掃描（Trivy）             阻斷：Critical CVE
   ├── SBOM 自動生成（Syft）            → 推送至 Artifact Store
   └── Cosign 數位簽章                  → 簽章上傳至 Registry

   │
   ▼
[部署至 Staging 環境]
   └── DAST 動態測試（OWASP ZAP）       阻斷：High 漏洞

   │
   ▼
[人工 Checkpoint]  ← 生產部署前的最後審查

   │  核准
   ▼
[部署至 Production]
   └── Cosign 驗證簽章                  阻斷：簽章無效則拒絕部署
```

**為什麼按這個順序？**

越靠左的掃描，**反饋越快、修復成本越低**，但能覆蓋的問題類型有限。越靠右的掃描（如 DAST）覆蓋更真實的攻擊面，但需要服務運行，只能在後期執行。

---

### 四、各安全工具詳解

#### 4.1 SAST — 靜態應用安全測試

**原理**：不執行程式，直接分析原始碼的語法樹（AST）或控制流程圖，比對已知漏洞模式（如不安全的函數呼叫、SQL 字串拼接等）。

**優點**：
- 在代碼合併前即可發現問題，修復成本最低
- 可掃描所有代碼路徑，包括很少執行的分支
- 無需部署運行環境

**限制**：
- **誤報率（False Positive）高**：約 30–50% 的警報是誤報，需人工確認
- 無法發現需要「運行時環境」才能觸發的漏洞（如設定檔錯誤、業務邏輯缺陷）
- 對加密、序列化等複雜操作的分析準確度有限

| 工具 | 支援語言 | 授權 | 特色 |
|------|---------|------|------|
| **Semgrep** | 多語言（30+）| 開源核心 + 商業規則 | 規則可自定義，語法簡潔；支援 OWASP Top 10 規則集 |
| **CodeQL** | 多語言 | GitHub 免費（公開 Repo）| 查詢語言強大，深度語義分析；GitHub Advanced Security 整合 |
| **Bandit** | Python | 開源（Apache 2.0）| 輕量，專為 Python 設計，適合快速導入 |
| **ESLint Security Plugin** | JavaScript/TypeScript | 開源 | 整合進現有 ESLint 工作流，零額外工具成本 |

**GitHub Actions 整合範例（Semgrep）**：

```yaml
- name: Run Semgrep SAST
  uses: returntocorp/semgrep-action@v1
  with:
    config: >
      p/owasp-top-ten
      p/secrets
  env:
    SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

#### 4.2 SCA — 軟體組成分析

**原理**：解析專案的依賴清單檔案（`package.json`、`requirements.txt`、`go.mod`、`pom.xml` 等），將所有直接與間接依賴與 CVE 資料庫（NVD、GitHub Advisory Database 等）比對，找出已知漏洞的第三方套件。

**與 SAST 的關鍵差異**：

| | SAST | SCA |
|--|------|-----|
| 掃描對象 | **自己寫的**原始碼 | **引入的**第三方套件 |
| 找什麼 | 代碼中的漏洞模式 | 已知 CVE 的依賴版本 |
| 修復方式 | 修改自己的代碼 | 升級/替換套件版本 |

> 💡 現代應用有 70–90% 的代碼來自第三方依賴，SCA 的覆蓋範圍往往比 SAST 更廣。

| 工具 | 特色 |
|------|------|
| **Dependabot** | GitHub 內建，免配置，自動開 PR 更新有漏洞的依賴 |
| **Trivy**（SCA 模式）| 同時支援 Container 掃描與依賴掃描，一個工具多用途 |
| **Snyk** | 商業工具，提供「可利用性分析」，可過濾實際不可利用的漏洞 |
| **OWASP Dependency-Check** | 老牌開源工具，支援多語言，報告格式豐富 |

**GitHub Actions 整合範例（Trivy SCA）**：

```yaml
- name: Run Trivy vulnerability scanner (SCA)
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'          # 掃描檔案系統（依賴清單）
    scan-ref: '.'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'           # 發現漏洞時讓 Job 失敗（阻斷 Pipeline）
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload Trivy scan results to GitHub Security tab
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

#### 4.3 機敏資訊掃描（Secret Scanning）

機敏資訊洩漏是最常見且最嚴重的安全事故類型之一。一旦 API Key、資料庫密碼進入 git history，即使後來刪除，由於 git 的不可變特性，仍可從歷史記錄中還原——**必須視為已洩露，立即輪換金鑰**。

**常見洩漏場景**：

- API Key、Access Token 硬編碼在代碼中（最常見）
- `.env` 檔案未加入 `.gitignore` 被意外 commit
- 在測試代碼中使用真實憑證
- `git stash` 後切換分支，stash 中的機敏資訊被推送

| 工具 | 特色 |
|------|------|
| **gitleaks** | 掃描完整 git history，支援自定義正則規則，pre-commit 與 CI 皆可用 |
| **truffleHog** | 同時使用高熵值分析（隨機字符串）與正則匹配，誤報率較低 |
| **GitHub Secret Scanning** | 平台內建，推送時即時掃描，自動通知並可自動撤銷部分 Token（如 GitHub PAT）|

**GitHub Actions 整合範例（gitleaks）**：

```yaml
- name: Run gitleaks secret scan
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # 商業版才需要
```

> ⚠️ **重要**：掃描時要帶 `fetch-depth: 0`（checkout 時取完整歷史），否則只掃最新一個 commit，無法發現歷史中的機敏資訊。

#### 4.4 Container 鏡像掃描

Container Image 由多個層（Layer）疊加而成，每一層都可能引入漏洞：

| 掃描層次 | 內容 | 典型問題 |
|---------|------|---------|
| **Base Image OS 套件** | `apt`/`apk` 安裝的系統套件 | 過時的 OpenSSL、libc 等 |
| **應用層依賴** | 應用程式安裝的語言套件 | 有 CVE 的 npm/pip 套件 |
| **檔案系統** | 鏡像內的任意檔案 | 意外打包進去的 `.env`、私鑰 |
| **設定** | Dockerfile 指令 | 以 root 執行、暴露不必要端口 |

| 工具 | 特色 |
|------|------|
| **Trivy** | 最廣泛使用，支援 OS 套件 + 應用依賴 + 機敏資訊，速度快 |
| **Grype** | Anchore 出品，可與 Syft 搭配使用（SBOM → 掃描）|
| **Snyk Container** | 商業工具，提供可利用性過濾與基礎鏡像升級建議 |

**GitHub Actions 整合範例（Trivy 鏡像掃描）**：

```yaml
- name: Build Docker image
  run: docker build -t myapp:${{ github.sha }} .

- name: Run Trivy image scan
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'image'
    image-ref: 'myapp:${{ github.sha }}'
    severity: 'CRITICAL'
    exit-code: '1'
```

**最佳化 Base Image 選擇**（減少攻擊面）：

| 類型 | 說明 | 推薦場景 |
|------|------|---------|
| `ubuntu:24.04` | 完整系統，套件多 | 不推薦直接用於生產 |
| `alpine:3.x` | 極輕量（~5MB），套件少 | 推薦，攻擊面最小 |
| `distroless` | Google 出品，無 shell、無套件管理器 | 最安全，難以除錯 |
| `chainguard` | Wolfi 基礎，每日重建，CVE 最少 | 企業級推薦 |

#### 4.5 DAST — 動態應用安全測試

**原理**：對**正在運行**的服務發送各種惡意 HTTP 請求（SQL 注入 payload、XSS payload、路徑遍歷等），根據伺服器的**回應行為**判斷是否存在漏洞。

**與 SAST 的互補關係**：

| | SAST | DAST |
|--|------|------|
| 需要服務運行 | ❌ 不需要 | ✅ 需要 |
| 能發現業務邏輯漏洞 | ❌ 難以發現 | ✅ 可以（觀察行為）|
| 能發現設定錯誤 | ❌ 難以發現 | ✅ 可以（如安全標頭缺失）|
| 誤報率 | 高 | 低（基於真實回應）|
| 執行位置 | CI 早期 | Staging 部署後 |

| 工具 | 特色 |
|------|------|
| **OWASP ZAP** | 開源旗艦工具，有 Baseline Scan（快速）和 Full Scan（深度）模式 |
| **Burp Suite Enterprise** | 商業工具，功能最強，適合高安全要求場景 |
| **Nuclei** | 基於模板，速度極快，適合針對特定 CVE 的驗證掃描 |

**GitHub Actions 整合範例（OWASP ZAP Baseline）**：

```yaml
- name: OWASP ZAP Baseline Scan
  uses: zaproxy/action-baseline@v0.12.0
  with:
    target: 'https://staging.myapp.com'
    rules_file_name: '.zap/rules.tsv'   # 自定義忽略規則
    cmd_options: '-a'                    # 包含 Ajax Spider
```

#### 4.6 SBOM — 軟體物料清單

**什麼是 SBOM？**

Software Bill of Materials — 軟體的「成分表」，==完整列出一個軟體產品所使用的所有直接與間接依賴，包含名稱、版本、授權條款、來源等資訊。==

**為什麼重要（Log4Shell 案例）**：

2021 年 Log4Shell（CVE-2021-44228）爆發時，大量企業花費數週才確認自己哪些系統使用了 Log4j。若當時有 SBOM，只需幾分鐘的 `grep` 即可定位所有受影響服務，大幅縮短響應時間。

**主要格式**：

| 格式 | 維護組織 | 特色 |
|------|---------|------|
| **SPDX** | Linux Foundation | 較老牌，ISO 標準（ISO/IEC 5962:2021）|
| **CycloneDX** | OWASP | 較現代，專為安全設計，工具生態豐富 |

**生成與使用流程**：

```yaml
# CI 中自動生成 SBOM
- name: Generate SBOM with Syft
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: cyclonedx-json          # 或 spdx-json
    output-file: sbom.json

- name: Upload SBOM as artifact
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.json
```

#### 4.7 Cosign 數位簽章

**解決的問題**：如何確保部署到生產環境的鏡像，是且僅是通過 CI 管道構建的那個版本，而非被惡意替換的版本？

**Cosign** 是 Sigstore 專案的一部分，使用公鑰/私鑰對 Container Image 進行數位簽章，並將簽章存儲在 OCI Registry 中（與鏡像同一位置）。

**完整流程**：

```bash
# Step 1：產生金鑰對（私鑰妥善保管，公鑰可公開）
cosign generate-key-pair

# Step 2：CI 構建鏡像後，用私鑰簽章
# 注意：簽章針對 digest（不可變），而非 tag（可變）
cosign sign --key cosign.key myregistry/myapp@sha256:<digest>

# Step 3：部署前驗證簽章（可整合至 Kyverno 准入控制）
cosign verify --key cosign.pub myregistry/myapp@sha256:<digest>
```

**GitHub Actions 整合範例**：

```yaml
- name: Sign container image
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
  run: |
    cosign sign --key env://COSIGN_PRIVATE_KEY \
      ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
```

> 💡 **Keyless 簽章（進階）**：Cosign 也支援無金鑰模式，利用 OIDC（如 GitHub Actions 的 identity token）和 Rekor 透明日誌進行簽章，不需管理私鑰。

---

### 五、SLSA 供應鏈安全框架

**SLSA（Supply chain Levels for Software Artifacts，發音 "salsa"）** 是由 Google 提出、OpenSSF 維護的供應鏈安全等級框架，定義了從「有基本流程」到「高度可信的密封構建」的四個等級。

| 等級 | 核心要求 | 典型實作方式 |
|------|---------|------------|
| **SLSA 1** | 構建流程有文件，能生成 Provenance（來源證明）| 任何 CI 系統 + Provenance 生成 Action |
| **SLSA 2** | 使用版本控制的 CI 系統，Provenance 由 CI 自動生成且不可偽造 | GitHub Actions + `slsa-framework/slsa-github-generator` |
| **SLSA 3** | 構建環境對開發者隔離，Provenance 可被第三方驗證 | 強化的 Runner 環境 + Sigstore 驗證 |
| **SLSA 4** | 雙人 Review、完全密封的構建環境、重現性構建（Reproducible Build）| 最高要求，適合底層基礎設施軟體 |

**Provenance（來源證明）**：

一份機器可讀的元數據文件，記錄：
- 這個產物是從**哪個代碼 commit** 構建的？
- 是在**哪個 CI 系統、哪台機器**上構建的？
- 使用了**哪些構建工具、哪些版本**？
- **構建過程**是否可重現？

```yaml
# 使用 SLSA GitHub Generator 自動生成 SLSA 2/3 等級的 Provenance
- name: Generate SLSA Provenance
  uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2
  with:
    image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
    digest: ${{ steps.build.outputs.digest }}
```

---

### 六、Security Gate 設定原則

**核心理念**：Gate 設定太嚴 → 開發者繞過或關閉掃描；太鬆 → 形同虛設。目標是找到**讓團隊願意配合的最高安全標準**。

**阻斷 vs 警告的決策矩陣**：

| 漏洞等級 | 可利用性 | 建議行為 | 修復 SLA |
|---------|---------|---------|---------|
| Critical | 任何 | ❌ 直接阻斷 Pipeline | 24 小時內 |
| High | 可利用 | ❌ 阻斷 Pipeline | 72 小時內 |
| High | 不可利用 | ⚠️ 警告，不阻斷 | 30 天內 |
| Medium | 任何 | ⚠️ 警告，記錄 Issue | 90 天內 |
| Low | 任何 | 📝 報告，可暫時忽略 | 視情況 |
| 誤報確認 | — | ✅ 加白名單（需審批記錄）| — |

**避免「警報疲勞」的關鍵做法**：

1. **設定精確的阻斷條件**：只阻斷真正需要立即處理的級別，不把所有 Medium 都設成阻斷
2. **白名單機制要有門禁**：誤報加白名單必須走 PR Review，留下審批紀錄，防止惡意放行
3. **定期清理白名單**：白名單隨時間增長會失去意義，應每季複審
4. **差異化掃描**：PR 掃描只掃「新增/變更的代碼」，而非全量掃描，縮短反饋時間
5. **可利用性過濾**：在技術成熟度允許時，引入 Reachability Analysis 降低誤報（參考 [07 筆記](./07-漏洞分流ASPM與度量指標.md)）

---

## 重點概念速查

- **CI（持續整合）**：每次代碼提交後自動執行建置與測試，確保主線代碼始終整合無誤
- **Continuous Delivery vs Deployment**：前者人工確認發布，後者全自動；多數企業選前者以保留生產閘控
- **Pipeline as Code**：將管道定義寫成 YAML/Groovy，與業務代碼一起版控、審查、測試
- **SAST**：靜態分析自己的原始碼，不需執行程式，早期發現漏洞模式，但誤報率高
- **DAST**：對運行中的服務發送惡意請求，發現 SAST 難以覆蓋的業務邏輯與設定漏洞
- **SCA**：掃描第三方依賴的已知 CVE，現代應用的主要攻擊面
- **Secret Scanning**：防止 API Key 等機敏資訊進入 git history，進入即視為洩露
- **SBOM**：軟體成分表，使組織能快速定位受新 CVE 影響的系統（Log4Shell 教訓）
- **Cosign**：對 Container Image 進行數位簽章，確保部署的鏡像來源可信
- **SLSA**：供應鏈安全等級框架，定義從基礎到高可信構建的四個等級
- **Security Gate**：管道中的安全阻斷條件，設計原則是找到讓團隊配合的最高標準
- **Provenance**：構建來源證明，記錄產物從哪個 commit、在哪個環境構建
