# Nebula CA 证书查找指南

## 问题

在 MinIO `robot-nebula-cert` 存储桶中，只找到了设备证书（`.crt`）和私钥（`.key`），但没有找到 CA 根证书（`ca.crt`）。

---

## 🔍 CA 证书的可能位置

### 位置 1: MinIO 存储桶中的独立目录

可能在以下位置：

```bash
# 1. CA 专用目录
mc ls myminio/robot-nebula-cert/CA/
mc ls myminio/robot-nebula-cert/ROOT/
mc ls myminio/robot-nebula-cert/NEBULA_CA/

# 2. 根目录下的 CA 文件
mc ls myminio/robot-nebula-cert/ | grep -i "ca\|root"

# 3. 搜索所有可能的 CA 文件
mc find myminio/robot-nebula-cert --name "*ca.crt"
mc find myminio/robot-nebula-cert --name "*ca.pem"
mc find myminio/robot-nebula-cert --name "*root*"
```

---

### 位置 2: Kubernetes Secrets

CA 证书可能存储在 Kubernetes Secret 中：

```bash
# 1. 查找 Nebula 相关的 Secret
kubectl get secrets -n spf | grep -i nebula
kubectl get secrets -n infrastructure | grep -i nebula

# 2. 查找 CA 相关的 Secret
kubectl get secrets -n spf | grep -i ca
kubectl get secrets -n infrastructure | grep -i ca

# 3. 查看 salt-remote-call 使用的 Secret
kubectl describe deployment salt-remote-call -n spf | grep -A 5 "Mounts:"
```

**从配置文件中看到的线索**：

`apps/spf_yaml/salt-remote-call.yaml:53-56`:
```yaml
- name: NEBULA_CA_KEY_PATH
  value: /nebula-cert/tls.key
- name: NEBULA_CA_CERT_PATH
  value: /nebula-cert/tls.crt
```

这表明 CA 证书可能被命名为 `tls.crt` 而不是 `ca.crt`。

---

### 位置 3: Kubernetes ConfigMap

CA 证书（公钥）可能存储在 ConfigMap 中：

```bash
# 1. 查找可能的 ConfigMap
kubectl get configmaps -n spf | grep -i "nebula\|ca\|aurotek"
kubectl get configmaps -n infrastructure | grep -i "nebula\|ca\|aurotek"

# 2. 查看内容
kubectl get configmap <configmap-name> -n spf -o yaml
```

---

### 位置 4: Station 设备本地

CA 证书可能只存在于 Station 设备上：

```bash
# SSH 到 Station
ssh station-admin@station5-067

# 查找 Nebula 配置目录
ls -la /etc/nebula/
ls -la /opt/nebula/
ls -la /opt/station/nebula/

# 常见的 CA 证书位置
cat /etc/nebula/ca.crt
cat /etc/nebula/config.yml | grep ca
```

---

### 位置 5: 嵌入在 Nebula 证书中

Nebula 使用的可能是**证书链**，CA 信息嵌入在设备证书中。

**验证方法**：

```bash
# 1. 下载设备证书
mc cp myminio/robot-nebula-cert/STATION/station5-067/station5-067.crt ./

# 2. 查看证书的 Issuer（签发者）
openssl x509 -in station5-067.crt -noout -issuer

# 3. 查看证书的 Subject（主题）
openssl x509 -in station5-067.crt -noout -subject

# 4. 查看完整证书信息
openssl x509 -in station5-067.crt -noout -text

# 5. 检查是否是自签名证书
openssl verify -CAfile station5-067.crt station5-067.crt
```

---

## 🎯 最可能的情况

### 情况 1: Nebula 不需要传统的 CA 证书

**Nebula VPN 的特点**：

- Nebula 使用自己的 PKI 系统
- 不一定需要传统的 X.509 CA 证书链
- CA 公钥可能直接嵌入在 Nebula 配置文件中

**Nebula 证书特点**：
- 使用 Ed25519 算法（这解释了 127B 的私钥大小）
- 证书格式可能不是标准的 X.509
- 使用 `nebula-cert` 工具生成和管理

**查看 Nebula 配置**：
```yaml
# /etc/nebula/config.yml
pki:
  ca: /etc/nebula/ca.crt          # CA 证书路径
  cert: /etc/nebula/host.crt      # 设备证书路径
  key: /etc/nebula/host.key       # 设备私钥路径
```

---

### 情况 2: CA 证书在 MinIO 的根目录

```bash
# 检查根目录
mc ls myminio/robot-nebula-cert/

# 可能的文件名
# - ca.crt
# - ca.pem
# - nebula-ca.crt
# - aurotek-ca.crt
# - root.crt
```

---

### 情况 3: CA 证书与设备证书同名

根据 `salt-remote-call.yaml` 的配置：
```yaml
NEBULA_CA_CERT_PATH: /nebula-cert/tls.crt
```

CA 证书可能就是 MinIO 中某个设备的 `tls.crt`，可能是：
- 一个特殊的 "CA" 设备证书
- 或者 STATION/ROBOT 中的某个特殊证书

---

## 🔬 深度分析步骤

### Step 1: 分析设备证书格式

```bash
# 下载证书
mc cp myminio/robot-nebula-cert/STATION/station5-067/station5-067.crt ./

# 查看证书格式
file station5-067.crt

# 查看证书内容（前几行）
head -20 station5-067.crt

# 如果是标准 X.509 证书
openssl x509 -in station5-067.crt -noout -text

# 如果是 Nebula 格式证书
nebula-cert print -path station5-067.crt
```

---

### Step 2: 检查证书大小的含义

**观察到的证书大小**：
- `.crt`: 312-324 bytes
- `.key`: 127 bytes（固定）

**分析**：
- 127B 的私钥 → 可能是 Ed25519 (32 bytes key + Base64 编码 ≈ 127B)
- 316B 的证书 → 可能是 Nebula 格式而非 X.509

**Nebula 证书格式**：
```
Nebula Certificate v1
  Name: station5-067
  Ip: 10.147.0.67/16
  Subnets: []
  Groups: ["station"]
  Not before: 2025-08-04 00:00:00 +0000 UTC
  Not After: 2026-08-04 00:00:00 +0000 UTC
  Public key: ...
  IsCA: false
  Issuer: ...
  Signature: ...
```

---

### Step 3: 查找 Nebula CA 工具

```bash
# 检查 Kubernetes 中是否有 nebula-cert 工具
kubectl exec -it deployment/salt-remote-call -n spf -- which nebula-cert

# 或者在本地安装
# https://github.com/slackhq/nebula/releases
wget https://github.com/slackhq/nebula/releases/download/v1.8.2/nebula-linux-amd64.tar.gz
tar -xzf nebula-linux-amd64.tar.gz
./nebula-cert print -path station5-067.crt
```

---

## 🛠️ 解决方案

### 方案 1: 使用自动查找脚本

```bash
chmod +x find_nebula_ca.sh
./find_nebula_ca.sh
```

---

### 方案 2: 从 Kubernetes 中提取

```bash
# 1. 找到使用 Nebula 证书的 Pod
kubectl get pods -n spf | grep salt-remote-call

# 2. 进入 Pod 查看
kubectl exec -it <pod-name> -n spf -- sh

# 3. 查找 CA 证书
ls -la /nebula-cert/
cat /etc/nebula/ca.crt
env | grep NEBULA

# 4. 复制出来
kubectl cp <pod-name>:/nebula-cert/ca.crt ./ca.crt -n spf
```

---

### 方案 3: 从 Station 设备获取

```bash
# 1. SSH 到 Station
ssh station-admin@station5-067

# 2. 查找 CA 证书
find /opt /etc -name "ca.crt" -o -name "ca.pem" 2>/dev/null

# 3. 复制回本地
scp station-admin@station5-067:/etc/nebula/ca.crt ./
```

---

### 方案 4: 联系运维团队

如果以上方法都无法找到，需要询问运维团队：

**关键问题**：
1. Nebula CA 证书存储在哪里？
2. 是使用标准 X.509 证书还是 Nebula 专有格式？
3. CA 证书是否需要分发给所有设备？
4. 如何获取 CA 证书用于验证设备证书？

---

## 📋 检查清单

在宣布"找不到 CA 证书"之前，请确认：

- [ ] 检查了 MinIO 所有顶级目录
- [ ] 搜索了 MinIO 中所有包含 "ca" 的文件
- [ ] 检查了 Kubernetes Secrets (spf 和 infrastructure namespace)
- [ ] 检查了 Kubernetes ConfigMaps
- [ ] 查看了 salt-remote-call 的 volume 挂载
- [ ] 分析了设备证书的格式和 Issuer
- [ ] 检查了证书是否是自签名
- [ ] 查看了 Nebula 相关的环境变量配置
- [ ] 尝试访问 Station 设备
- [ ] 联系了运维团队

---

## 🎓 Nebula 证书系统简介

### Nebula 与传统 PKI 的区别

| 特性 | 传统 X.509 PKI | Nebula PKI |
|------|---------------|-----------|
| 证书格式 | X.509 (PEM/DER) | Nebula 专有格式 |
| 签名算法 | RSA/ECDSA | Ed25519 |
| 证书链 | 支持多级 CA | 单级 CA |
| 用途 | 通用 TLS/SSL | 专为 Nebula VPN |
| 工具 | openssl | nebula-cert |

### Nebula 证书命令

```bash
# 生成 CA
nebula-cert ca -name "Aurotek"

# 签发设备证书
nebula-cert sign -name "station5-067" -ip "10.147.0.67/16"

# 查看证书
nebula-cert print -path station5-067.crt

# 验证证书
nebula-cert verify -ca ca.crt -crt station5-067.crt
```

---

## 🔗 相关资源

- [Nebula 官方文档](https://nebula.defined.net/)
- [Nebula GitHub](https://github.com/slackhq/nebula)
- [Nebula 证书管理](https://nebula.defined.net/docs/config/certificate-authority/)

---

## 💡 总结

**最可能的情况**：

1. **CA 证书在 Kubernetes Secret 中**，可能命名为 `nebula-ca` 或类似
2. **CA 证书在 MinIO 根目录**，但没有放在子目录中
3. **Nebula 使用自己的证书格式**，不是标准 X.509，需要用 `nebula-cert` 查看
4. **CA 公钥嵌入在配置**中，不是独立文件

**建议的行动**：

1. 运行 `./find_nebula_ca.sh` 脚本
2. 检查 Kubernetes 中的 salt-remote-call Pod
3. 分析设备证书确定是否需要独立的 CA
4. 必要时联系运维团队

---

**文档版本**: 1.0
**创建时间**: 2025-11-18
**最后更新**: 2025-11-18
