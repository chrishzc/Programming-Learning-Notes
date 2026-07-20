# 08 · 實戰 Lab：三個微型專案

> **學習目標**：在本地虛擬化環境中，透過完整動手實作，驗證並鞏固前六篇筆記的所有核心安全機制。
>
> **前置條件**：建議先閱讀完 01–06 篇後再進行實作。

---

## Lab 概覽

| # | 專案名稱 | 核心技術 | 難度 | 狀態 |
|---|---------|---------|------|------|
| Lab 1 | 邊界安全與反向代理硬化 | Nginx、OpenSSL | 🟡 中 | 🔲 未開始 |
| Lab 2 | 黃金 CI 管道與供應鏈簽章 | GitHub Actions、Cosign、SAST/SCA | 🔴 高 | 🔲 未開始 |
| Lab 3 | K8s 集群零信任微隔離與准入 | Minikube、Calico/Cilium、Kyverno | 🔴 高 | 🔲 未開始 |

---

## Lab 1：邊界安全與反向代理硬化

**目標**：構建符合工業級安全標準的前置流量防禦節點，防範掃描與基礎 L7 攻擊

### 1.1 環境準備

> 待補充（需要的軟體、Docker、測試用 Web 服務）

### 1.2 實施步驟

- [ ] 部署一個簡單 Web 服務
- [ ] 在 Nginx 後方代理該服務
- [ ] 硬化 `/etc/nginx/nginx.conf`（參考 [02](./02-邊界防禦：Nginx硬化實務.md)）
- [ ] 強制啟用 TLS 1.3，禁用舊版協定
- [ ] 關閉 `server_tokens`，設定安全回應標頭

### 1.3 驗證指標

```bash
# 測試 1：舊版 TLS 是否被拒絕
openssl s_client -connect localhost:443 -tls1_1
# 預期結果：待補充

# 測試 2：Server 標頭是否隱藏
curl -I https://localhost
# 預期結果：待補充

# 測試 3：安全標頭是否正確傳遞
curl -I https://localhost | grep -E "X-Frame|X-Content|CSP"
# 預期結果：待補充
```

### 1.4 遇到的問題與解法（學習記錄）

> 待補充（邊做邊填）

---

## Lab 2：黃金 CI 管道與供應鏈簽章

**目標**：建置完整的安全左移自動化流程，不讓任何未授權、不安全的代碼流向生產

### 2.1 環境準備

> 待補充（GitHub Repo、Container Registry 設定）

### 2.2 實施步驟

- [ ] 建立 GitHub Actions CI 管道
- [ ] 整合 SAST 掃描（工具待選：待補充）
- [ ] 整合 SCA 掃描（工具待選：待補充）
- [ ] 整合機敏資訊掃描（如 truffleHog / gitleaks）
- [ ] 整合 SBOM 自動生成（如 Syft）
- [ ] 使用 Cosign 對構建完成的鏡像執行私鑰簽章
- [ ] 推送帶有簽章的鏡像至 Registry

### 2.3 GitHub Actions Pipeline 骨架

```yaml
# 待補充：.github/workflows/ci-security.yml 骨架
```

### 2.4 Cosign 簽章流程

> 待補充

```bash
# 生成金鑰對
# cosign generate-key-pair ...（待補充）

# 簽署鏡像
# cosign sign ...（待補充）

# 驗證簽章
# cosign verify ...（待補充）
```

### 2.5 驗證指標

- [ ] 故意提交含明文 API Key 的代碼 → 確認 CI 成功阻斷
- [ ] 檢索生成的 SBOM 完整性
- [ ] 確認 Registry 中的鏡像包含有效的 `.sig` 簽名

### 2.6 遇到的問題與解法（學習記錄）

> 待補充

---

## Lab 3：K8s 集群零信任微隔離與准入

**目標**：實踐容器內部的多層次安全，防範越權掛載、未授權鏡像部署與內部橫向滲透

### 3.1 環境準備

> 待補充

```bash
# 安裝 Minikube
# 安裝 Calico 或 Cilium CNI
# 安裝 Kyverno
# （待補充各工具安裝指令）
```

### 3.2 實施步驟

- [ ] 在 Minikube 部署集群並加裝 Calico/Cilium CNI
- [ ] 安裝 Kyverno
- [ ] 部署 Kyverno `enforce-workload-standards` 策略（參考 [04](./04-K8s集群硬化與准入控制.md)）
- [ ] 部署 `default-deny-all` NetworkPolicy（參考 [05](./05-K8s零信任網路策略.md)）
- [ ] 增設 DNS 流出放行規則（放行 kube-dns UDP/TCP 53）

### 3.3 驗證指標

```bash
# 測試 1：Kyverno 准入攔截
kubectl apply -f pod-no-limits.yaml
# 預期結果：待補充（應被 Kyverno 拒絕）

# 測試 2：NetworkPolicy 橫向移動阻斷
kubectl exec -it <pod-A> -- ping <pod-B-IP>
# 預期結果：待補充（應被阻斷）

# 測試 3：DNS 解析是否正常
kubectl exec -it <pod> -- nslookup kubernetes.default
# 預期結果：待補充（DNS 放行後應成功解析）
```

### 3.4 遇到的問題與解法（學習記錄）

> 待補充

---

## 三個 Lab 對應的筆記篇章

| Lab | 對應筆記 |
|-----|---------|
| Lab 1 | [02 Nginx 硬化](./02-邊界防禦：Nginx硬化實務.md) |
| Lab 2 | 05 CI/CD（待建立）、06 供應鏈安全（待建立）|
| Lab 3 | [04 K8s 硬化](./04-K8s集群硬化與准入控制.md) · [05 NetworkPolicy](./05-K8s零信任網路策略.md) |

---

## 延伸閱讀 / 參考資源

- [ ] Cosign 官方文件：https://docs.sigstore.dev
- [ ] Syft（SBOM 生成工具）
- [ ] GitHub Actions 官方文件
- [ ] Minikube 官方文件
