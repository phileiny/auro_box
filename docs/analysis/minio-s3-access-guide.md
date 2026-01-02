# MinIO/S3 访问指南

## 概述

本文档提供 MinIO/S3 对象存储的访问凭证和操作指南，用于从云平台下载 Station 证书备份和其他文件。

---

## 凭证信息

### 基本访问凭证

| 项目 | 值 |
|------|-----|
| **Endpoint** | `https://robot-s3.aurotek.com` |
| **Access Key ID** | `i9khdUiVt4ygn46CvX8pNmhgNNw6Vd35` |
| **Secret Access Key** | `YHyNE3o4SoV9cjHSzAsoLVAaG5JwBZXx` |
| **协议** | S3 兼容 API |
| **SSL 验证** | 禁用 (使用自签名证书) |

**来源**: 这些凭证在多个 Kubernetes Deployment 配置中使用，包括:
- `apps/spf_yaml/jarvis-iot-deploy.yaml:126-129`
- `apps/spf_yaml/jarvis-alarm.yaml:85-88`
- `apps/spf_yaml/jarvis-diagnostics.yaml:60-62`

---

## 存储桶列表

系统中使用的主要存储桶：

| 存储桶名称 | 用途 | 使用服务 | 配置文件 |
|-----------|------|---------|---------|
| `robot-local-data` | 本地数据、截图、临时文件 | jarvis-alarm, jarvis-iot | jarvis-alarm.yaml:90 |
| `robot-jarvis-dobby` | Dobby 相关文件 | jarvis-iot-deploy | jarvis-iot-deploy.yaml:133 |
| `robot-mapscan` | 地图扫描数据 | jarvis-iot-deploy | jarvis-iot-deploy.yaml:141 |
| `robot-jarvis-site` | 站点文件、地图 | jarvis-site | jarvis-site.yaml |
| `robot-nebula-cert` | **Nebula 网络证书（所有设备）** | 证书管理 | ✓ 已确认 |
| `robot-software-release` | 软件发布包 | 软件部署 | - |
| `robot-c-app` | C 应用程序文件 | 应用部署 | - |
| `robot-erp` | ERP 相关文件 | ERP 系统 | - |
| `robot-jarvis-watson` | Watson 服务文件 | jarvis-watson | - |

---

## 快速开始（推荐）

### 使用自动化脚本下载证书

项目提供了两个自动化脚本，可以快速下载 Station 证书：

#### Bash 脚本（推荐）

```bash
# 赋予执行权限
chmod +x download_station_cert.sh

# 下载 station5-067 的证书
./download_station_cert.sh station5-067

# 下载其他 Station 的证书
./download_station_cert.sh station5-066
```

#### Python 脚本

```bash
# 安装依赖
pip install boto3

# 列出所有可用的 Station
python download_station_cert.py --list

# 下载指定 Station 的证书
python download_station_cert.py station5-067
```

**脚本功能：**
- ✓ 自动检查证书是否存在
- ✓ 下载证书和私钥
- ✓ 设置正确的文件权限 (644/600)
- ✓ 验证证书和私钥匹配
- ✓ 显示证书详细信息

---

## 手动访问方式

### 方法 1: MinIO Client (mc) - 推荐

#### 安装 MinIO Client

```bash
# Windows
# 下载: https://dl.min.io/client/mc/release/windows-amd64/mc.exe
# 添加到 PATH 或直接使用

# Linux
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# macOS
brew install minio/stable/mc
```

#### 配置和使用

```bash
# 1. 配置别名
mc alias set robot-cloud https://robot-s3.aurotek.com i9khdUiVt4ygn46CvX8pNmhgNNw6Vd35 YHyNE3o4SoV9cjHSzAsoLVAaG5JwBZXx --insecure

# 2. 列出所有存储桶
mc ls robot-cloud/

# 3. 查看特定存储桶内容
mc ls robot-cloud/robot-local-data/
mc ls robot-cloud/certs/

# 4. 搜索证书相关文件
mc find robot-cloud/certs --name "*station*"
mc find robot-cloud/certs --name "*5040001*"
mc find robot-cloud/certs --name "*station5-067*"

# 5. 下载整个目录
mc cp robot-cloud/certs/STATION/station5-067/ ./certs/ --recursive

# 6. 下载单个文件
mc cp robot-cloud/certs/STATION/station5-067/tls.crt ./certs/station5-067.crt
mc cp robot-cloud/certs/STATION/station5-067/tls.key ./certs/station5-067.key
mc cp robot-cloud/certs/STATION/station5-067/ca.crt ./certs/ca.crt

# 7. 查看对象信息
mc stat robot-cloud/certs/STATION/station5-067/tls.crt

# 8. 设置文件权限
chmod 600 ./certs/*.key
chmod 644 ./certs/*.crt
```

---

### 方法 2: AWS CLI (S3 兼容)

#### 安装 AWS CLI

```bash
# Windows
# 下载安装: https://awscli.amazonaws.com/AWSCLIV2.msi

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# macOS
brew install awscli
```

#### 配置和使用

```bash
# 1. 设置环境变量
export AWS_ACCESS_KEY_ID=i9khdUiVt4ygn46CvX8pNmhgNNw6Vd35
export AWS_SECRET_ACCESS_KEY=YHyNE3o4SoV9cjHSzAsoLVAaG5JwBZXx
export AWS_ENDPOINT_URL=https://robot-s3.aurotek.com

# 2. 列出所有存储桶
aws s3 ls --endpoint-url $AWS_ENDPOINT_URL --no-verify-ssl

# 3. 列出存储桶内容
aws s3 ls s3://certs/ --endpoint-url $AWS_ENDPOINT_URL --recursive --no-verify-ssl

# 4. 下载文件
aws s3 cp s3://certs/STATION/station5-067/tls.crt ./certs/station5-067.crt \
    --endpoint-url $AWS_ENDPOINT_URL --no-verify-ssl

# 5. 下载整个目录
aws s3 cp s3://certs/STATION/station5-067/ ./certs/ \
    --recursive --endpoint-url $AWS_ENDPOINT_URL --no-verify-ssl

# 6. 同步目录
aws s3 sync s3://certs/STATION/ ./certs/ \
    --endpoint-url $AWS_ENDPOINT_URL --no-verify-ssl
```

---

### 方法 3: 通过 Kubernetes Port-Forward 访问 MinIO Console

```bash
# 1. 查看 MinIO 服务
kubectl get svc -n infrastructure | grep minio

# 2. 转发 MinIO API 端口
kubectl port-forward svc/minio -n infrastructure 9000:9000

# 3. 转发 MinIO Console 端口 (如果存在)
kubectl port-forward svc/console -n infrastructure 9090:9090

# 4. 在浏览器访问 MinIO Web UI
# 访问 Console: http://localhost:9090
# 或访问 API: http://localhost:9000

# 登录凭证:
# Username: i9khdUiVt4ygn46CvX8pNmhgNNw6Vd35
# Password: YHyNE3o4SoV9cjHSzAsoLVAaG5JwBZXx
```

#### Web UI 操作

1. 登录后，点击左侧 "Buckets"
2. 找到 `certs` 或相关存储桶
3. 浏览目录结构: `STATION/` → `station5-067/`
4. 选择文件，点击 "Download" 下载

---

### 方法 4: 使用 Python boto3 库

#### 安装依赖

```bash
pip install boto3
```

#### 示例代码

```python
#!/usr/bin/env python3
"""
MinIO S3 证书下载脚本
"""

import boto3
from botocore.client import Config
from botocore.exceptions import ClientError
import os
import urllib3

# 禁用 SSL 警告
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# 配置
ENDPOINT = 'https://robot-s3.aurotek.com'
ACCESS_KEY = 'i9khdUiVt4ygn46CvX8pNmhgNNw6Vd35'
SECRET_KEY = 'YHyNE3o4SoV9cjHSzAsoLVAaG5JwBZXx'

# 创建 S3 客户端
s3_client = boto3.client(
    's3',
    endpoint_url=ENDPOINT,
    aws_access_key_id=ACCESS_KEY,
    aws_secret_access_key=SECRET_KEY,
    config=Config(signature_version='s3v4'),
    verify=False  # 跳过 SSL 验证
)

def list_buckets():
    """列出所有存储桶"""
    try:
        response = s3_client.list_buckets()
        print("可用的存储桶:")
        for bucket in response['Buckets']:
            print(f"  - {bucket['Name']}")
        return [bucket['Name'] for bucket in response['Buckets']]
    except ClientError as e:
        print(f"错误: {e}")
        return []

def list_objects(bucket_name, prefix=''):
    """列出存储桶中的对象"""
    try:
        response = s3_client.list_objects_v2(
            Bucket=bucket_name,
            Prefix=prefix
        )

        if 'Contents' not in response:
            print(f"在 {bucket_name}/{prefix} 中没有找到对象")
            return []

        objects = []
        for obj in response['Contents']:
            print(f"  - {obj['Key']} ({obj['Size']} bytes)")
            objects.append(obj['Key'])

        return objects
    except ClientError as e:
        print(f"错误: {e}")
        return []

def download_file(bucket_name, object_key, local_path):
    """下载文件"""
    try:
        # 确保目录存在
        os.makedirs(os.path.dirname(local_path), exist_ok=True)

        # 下载文件
        s3_client.download_file(bucket_name, object_key, local_path)
        print(f"✓ 下载成功: {object_key} -> {local_path}")

        # 设置权限 (如果是密钥文件)
        if local_path.endswith('.key') or 'key' in local_path:
            os.chmod(local_path, 0o600)
        elif local_path.endswith('.crt') or 'crt' in local_path:
            os.chmod(local_path, 0o644)

        return True
    except ClientError as e:
        print(f"✗ 下载失败: {e}")
        return False

def download_station_certs(station_name, station_uid=None):
    """下载 Station 证书"""
    print(f"\n正在查找 {station_name} 的证书...")

    # 可能的存储桶和路径
    possible_locations = [
        ('certs', f'STATION/{station_name}/'),
        ('certs', f'STATION/{station_uid}/'),
        ('nebula-certs', f'{station_name}/'),
        ('station-certs', f'{station_name}/'),
        ('robot-local-data', f'certs/{station_name}/'),
    ]

    found = False
    for bucket, prefix in possible_locations:
        print(f"\n检查: {bucket}/{prefix}")
        objects = list_objects(bucket, prefix)

        if objects:
            found = True
            print(f"\n在 {bucket}/{prefix} 找到文件，开始下载...")

            for obj_key in objects:
                filename = os.path.basename(obj_key)
                local_path = f'./certs/{filename}'
                download_file(bucket, obj_key, local_path)

            break

    if not found:
        print(f"\n⚠ 未找到 {station_name} 的证书")
        print("建议:")
        print("  1. 检查所有存储桶列表")
        print("  2. 手动浏览存储桶内容")
        print("  3. 联系运维团队确认证书存储位置")

# 主程序
if __name__ == '__main__':
    print("=" * 60)
    print("MinIO S3 证书下载工具")
    print("=" * 60)

    # 1. 列出所有存储桶
    buckets = list_buckets()

    # 2. 下载特定 Station 的证书
    download_station_certs('station5-067', '5040001')

    print("\n" + "=" * 60)
    print("完成")
    print("=" * 60)
```

#### 运行脚本

```bash
# 保存脚本为 download_certs.py
python download_certs.py
```

---

## 存储桶路径结构（已确认）

### robot-nebula-cert 存储桶结构

证书存储桶按**设备类型**分类，每个设备有独立的证书和私钥：

```
robot-nebula-cert/
├── STATION/                    # Station 设备证书
│   ├── station5-066/
│   │   ├── station5-066.crt   # 证书 (316B)
│   │   └── station5-066.key   # 私钥 (127B)
│   ├── station5-067/          # ✓ 目标 Station
│   │   ├── station5-067.crt   # 证书 (316B)
│   │   └── station5-067.key   # 私钥 (127B)
│   └── ...
├── ROBOT/                      # 机器人设备证书
│   ├── kago5-1190-arma/
│   │   ├── kago5-1190-arma.crt
│   │   └── kago5-1190-arma.key
│   ├── kago5-1190-armb/
│   │   ├── kago5-1190-armb.crt
│   │   └── kago5-1190-armb.key
│   └── ...
├── ANGO/                       # Ango Box 设备证书
│   ├── ango5-box-029/
│   │   ├── ango5-box-029.crt
│   │   └── ango5-box-029.key
│   └── ...
├── LASMA_DISINFECTION/         # 消毒设备证书
│   ├── air-018/
│   │   ├── air-018.crt
│   │   └── air-018.key
│   └── ...
└── VENDOR/                     # 供应商设备证书
    ├── vendor3-306/
    │   ├── vendor3-306.crt
    │   └── vendor3-306.key
    └── ...
```

**证书文件特征：**
- 证书文件 `.crt`: 约 312-324 字节
- 私钥文件 `.key`: 固定 127 字节
- 命名规则: `{设备名称}.crt` / `{设备名称}.key`
- 更新时间: 2025-08-04 至 2025-08-08

### 其他存储桶结构

```
robot-local-data/
├── things/
│   └── {unit_uid}/
│       └── sys_a/
│           └── screen/
│               └── screenshot/
│                   └── file/
│                       └── {timestamp}

robot-jarvis-dobby/
├── dobby-files/
└── ...

robot-mapscan/
├── maps/
└── scans/

robot-jarvis-site/
├── sites/
└── site-files/
```

---

## 查找证书的步骤

### Step 1: 列出所有存储桶

```bash
mc ls robot-cloud/
```

### Step 2: 搜索证书相关的存储桶

查找名称包含以下关键字的存储桶:
- `cert`
- `station`
- `nebula`
- `tls`
- `ssl`

### Step 3: 探索证书存储桶

```bash
# 证书存储在 robot-nebula-cert 桶中
mc ls robot-cloud/robot-nebula-cert/

# 查看 STATION 类型的证书
mc ls robot-cloud/robot-nebula-cert/STATION/

# 列出所有 Station 证书
mc ls robot-cloud/robot-nebula-cert/STATION/ --recursive
```

### Step 4: 按 Station 信息搜索

```bash
# 按 Station 名称搜索
mc find robot-cloud/robot-nebula-cert --name "*station5-067*"

# 搜索所有 STATION 类型设备
mc find robot-cloud/robot-nebula-cert/STATION --name "*.crt"

# 列出特定 Station 的证书
mc ls robot-cloud/robot-nebula-cert/STATION/station5-067/
```

### Step 5: 下载证书

```bash
# 方法 1: 下载整个证书目录
mc cp robot-cloud/robot-nebula-cert/STATION/station5-067/ ./certs/ --recursive

# 方法 2: 分别下载证书和私钥
mc cp robot-cloud/robot-nebula-cert/STATION/station5-067/station5-067.crt ./certs/
mc cp robot-cloud/robot-nebula-cert/STATION/station5-067/station5-067.key ./certs/

# 设置正确的文件权限
chmod 600 ./certs/station5-067.key
chmod 644 ./certs/station5-067.crt

# 验证证书
openssl x509 -in ./certs/station5-067.crt -noout -text
```

---

## 验证下载的证书

### 1. 查看证书详细信息

```bash
openssl x509 -in certs/station5-067.crt -text -noout

# 应该包含:
# Subject: CN=station5-067, O=Aurotek, OU=Station
# Issuer: CN=Aurotek Root CA
# Validity (有效期)
# Subject Alternative Name: IP:10.43.0.67, DNS:station5-067
```

### 2. 验证证书和私钥匹配

```bash
# 比较 modulus (应该相同)
cert_md5=$(openssl x509 -noout -modulus -in certs/station5-067.crt | openssl md5)
key_md5=$(openssl rsa -noout -modulus -in certs/station5-067.key | openssl md5)

echo "Cert MD5: $cert_md5"
echo "Key MD5:  $key_md5"

# 如果 MD5 相同，证书和私钥匹配 ✓
```

### 3. 验证证书链

```bash
# 用 CA 验证证书
openssl verify -CAfile certs/ca.crt certs/station5-067.crt

# 应该输出: certs/station5-067.crt: OK
```

### 4. 检查证书有效期

```bash
openssl x509 -in certs/station5-067.crt -noout -dates

# 输出:
# notBefore=Jan  1 00:00:00 2024 GMT
# notAfter=Dec 31 23:59:59 2025 GMT
```

---

## 故障排查

### 问题 1: SSL 验证失败

```bash
# 使用 --insecure 参数 (mc)
mc alias set robot-cloud https://robot-s3.aurotek.com ACCESS_KEY SECRET_KEY --insecure

# 使用 --no-verify-ssl 参数 (aws)
aws s3 ls --endpoint-url https://robot-s3.aurotek.com --no-verify-ssl

# Python 设置 verify=False
s3_client = boto3.client('s3', ..., verify=False)
```

### 问题 2: 找不到存储桶

```bash
# 1. 确认凭证正确
mc admin info robot-cloud/

# 2. 检查权限
mc ls robot-cloud/

# 3. 查看所有可用的存储桶
mc ls robot-cloud/ --recursive
```

### 问题 3: 下载失败

```bash
# 检查对象是否存在
mc stat robot-cloud/certs/STATION/station5-067/tls.crt

# 检查权限
mc ls robot-cloud/certs/STATION/station5-067/ --recursive

# 尝试分段下载
mc cp robot-cloud/certs/STATION/station5-067/tls.crt ./certs/test.crt --limit-download 1MB
```

### 问题 4: 证书不在预期位置

如果在 MinIO 中找不到证书:

1. **检查 Kubernetes Secrets** (见 `CLOUD_CERT_QUERY_GUIDE.md`)
2. **检查 Vault** (如果使用密钥管理服务)
3. **联系运维团队**确认证书存储策略
4. **从 Station 设备直接获取**

---

## 安全注意事项

### 1. 凭证保护

```bash
# 不要将凭证提交到版本控制
echo ".aws/" >> .gitignore
echo "certs/*.key" >> .gitignore

# 使用环境变量而非硬编码
export MINIO_ACCESS_KEY=i9khdUiVt4ygn46CvX8pNmhgNNw6Vd35
export MINIO_SECRET_KEY=YHyNE3o4SoV9cjHSzAsoLVAaG5JwBZXx
```

### 2. 证书文件权限

```bash
# 私钥应该只对所有者可读
chmod 600 certs/*.key

# 证书可以对所有人可读
chmod 644 certs/*.crt

# 确认权限
ls -la certs/
```

### 3. 清理临时文件

```bash
# 使用完证书后，删除本地副本
rm -rf ./certs/

# 或加密存储
tar czf certs.tar.gz certs/
gpg -c certs.tar.gz
rm -rf certs/ certs.tar.gz
```

---

## 📊 高级用法

### 批量操作

```bash
# 镜像整个存储桶（本地备份）
mc mirror myminio/robot-nebula-cert/ ./backup/certs/

# 同步指定目录（仅同步变更）
mc mirror myminio/robot-jarvis-site/product/robot/maps/501/ ./maps/site_501/

# 删除旧文件（30天前）
mc rm --recursive --force --older-than 30d myminio/robot-local-data/things/

# 批量下载特定文件类型
mc find myminio/robot-mapscan --name "*.png" --exec "mc cp {} ./maps/"
```

### 监控和通知

```bash
# 监控存储桶变化
mc watch myminio/robot-nebula-cert

# 监控特定前缀
mc watch myminio/robot-jarvis-site/product/robot/maps/504/

# 查看存储桶事件
mc event list myminio/robot-nebula-cert
```

### 性能优化

```bash
# 并发下载（提高速度）
mc cp --recursive myminio/robot-jarvis-site/product/robot/maps/ ./maps/

# 续传下载（网络中断后继续）
mc cp --recursive --continue myminio/robot-jarvis-dobby/ ./dobby_data/

# 限速下载（避免占用带宽）
mc cp --limit-download 1MB myminio/robot-c-app/app-torch/data/data/rsync/1340010011/2025-08-08/11/06/1754622365849.jpg ./
```

### 数据统计和分析

```bash
# 统计存储桶大小
mc du myminio/robot-nebula-cert/

# 统计文件数量
mc ls myminio/robot-local-data/things/ --recursive | wc -l

# 查找最大的文件
mc ls myminio/robot-jarvis-dobby/robot/sensor/ --recursive | sort -k3 -hr | head -10

# 查找最新的文件
mc ls myminio/robot-jarvis-site/product/robot/maps/504/ --recursive | tail -1
```

### 访问策略管理

```bash
# 查看存储桶策略
mc policy get myminio/robot-nebula-cert

# 查看存储桶信息
mc stat myminio/robot-nebula-cert
```

---

## 参考资料

### MinIO 文档
- 官方文档: https://min.io/docs/minio/linux/index.html
- MinIO Client 指南: https://min.io/docs/minio/linux/reference/minio-mc.html

### 相关配置文件
- `apps/spf_yaml/jarvis-iot-deploy.yaml:126-141` - MinIO 凭证配置
- `apps/spf_yaml/jarvis-alarm.yaml:85-90` - OSS 访问配置
- `traefik/minio.yaml` - MinIO Ingress 路由
- `CLOUD_CERT_QUERY_GUIDE.md` - 证书查询完整指南

### 相关文档
- `minio-buckets-analysis.md` - **MinIO 存储桶详细分析** ⭐ 新增
- `CLOUD_CERT_QUERY_GUIDE.md` - 云平台证书查询指南
- `STATION_CLOUD_AUTH_GUIDE.md` - Station 云端认证指南
- `../QUICK_DOWNLOAD_CERTS.md` - 快速下载命令备忘单
- `../MINIO_CERT_SUMMARY.md` - MinIO 发现总结

### 外部资源
- [AWS CLI S3 命令参考](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [boto3 S3 文档](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html)

---

**文档版本**: 2.0
**最后更新**: 2025-11-18
**维护者**: DevOps Team

**变更历史**:
- v2.0 (2025-11-18): 增加高级用法、批量操作、性能优化章节；添加存储桶分析文档链接
- v1.0 (2025-11-18): 初始版本，包含基本访问方法和证书下载指南
