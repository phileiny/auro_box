# SSL 憑證更新指南

## 概述

本文檔說明如何更新雲平台的 SSL 憑證（`default-com-tls` Secret）。

## 憑證檔案說明

從憑證供應商（如 Sectigo）取得的憑證通常包含以下檔案：

| 檔案 | 說明 |
|------|------|
| `XXXXXXXX.crt` | 主憑證（伺服器憑證） |
| `KEY.txt` 或 `*.key` | 私鑰 |
| `SectigoPublicServerAuthenticationCADVR36.crt` | 中繼 CA 憑證 |
| `SectigoPublicServerAuthenticationRootR46_USERTrust.crt` | 根/中繼 CA 憑證 |
| `USERTrustRSACertificationAuthority.crt` | 根 CA 憑證（可選） |

## 更新步驟

### 步驟 1：上傳憑證到雲平台

將憑證檔案上傳到雲平台伺服器，例如：
```bash
/certs/cert_2026/
```

### 步驟 2：合併憑證鏈

SSL 憑證需要包含完整的信任鏈，順序為：主憑證 → 中繼 CA → 根 CA

```bash
cd /certs/cert_2026

# 合併憑證鏈（依實際檔名調整）
cat 2708611629.crt \
    SectigoPublicServerAuthenticationCADVR36.crt \
    SectigoPublicServerAuthenticationRootR46_USERTrust.crt \
    > tls.crt

# 複製私鑰
cp KEY.txt tls.key
```

### 步驟 3：驗證憑證與私鑰匹配

```bash
# 檢查憑證資訊
openssl x509 -in tls.crt -noout -subject -dates -issuer

# 驗證憑證與私鑰是否匹配（兩個輸出應相同）
openssl x509 -noout -modulus -in tls.crt | md5sum
openssl rsa -noout -modulus -in tls.key | md5sum
```

### 步驟 4：更新 Kubernetes Secret

```bash
kubectl create secret tls default-com-tls \
  --cert=tls.crt \
  --key=tls.key \
  --dry-run=client -o yaml | kubectl apply -f - -n infrastructure
```

> **注意**：如果出現 `missing the kubectl.kubernetes.io/last-applied-configuration annotation` 警告，這是正常的，不影響更新結果。

### 步驟 5：驗證更新結果

```bash
# 檢查憑證有效期
kubectl get secret default-com-tls -n infrastructure -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

預期輸出：
```
notBefore=Dec 16 00:00:00 2025 GMT
notAfter=Jan 16 23:59:59 2027 GMT
```

### 步驟 6：驗證外部訪問

使用瀏覽器訪問以下網址，點擊地址欄「鎖」圖標查看憑證有效期：

- https://u-ops.aurotek.com/login
- https://u-deploy.aurotek.com/login

## 相關資源

- Secret 名稱：`default-com-tls`
- Namespace：`infrastructure`
- 憑證類型：`kubernetes.io/tls`

## 更新紀錄

| 日期 | 有效期 | 備註 |
|------|--------|------|
| 2025-12-23 | 2025-12-16 ~ 2027-01-16 | 更新至 Sectigo 憑證 |
