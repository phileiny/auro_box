# K8s Log 抓取指南

## 概述

本文檔說明如何從 Kubernetes 集群抓取各服務的 log，用於問題排查和系統分析。

---

## 連線方式

### SSH 連接到雲平台

```bash
cd /d/sides
ssh -i "Aurotek-Taipei-ec2.pem" ubuntu@ec2-43-213-110-250.ap-east-2.compute.amazonaws.com
```

---

## 基本 kubectl logs 語法

### 語法格式

```bash
kubectl -n <namespace> logs <target> [options] | grep "<pattern>"
```

### 常用參數

| 參數 | 說明 | 範例 |
|------|------|------|
| `-n <namespace>` | 指定命名空間 | `-n application` |
| `-l app=<name>` | 用 label 選擇 pods | `-l app=jarvis-iot-v6` |
| `--all-containers` | 包含所有容器 (含 sidecar) | |
| `--tail=<N>` | 只取最後 N 行 | `--tail=1000` |
| `--since=<duration>` | 取最近一段時間 | `--since=3h` |
| `--since-time=<time>` | 取指定時間後的 log | `--since-time="2025-10-28T15:00:00+08:00"` |
| `-f` | 實時追蹤 (follow) | |
| `deploy/<name>` | 指定 deployment | `deploy/app-delivery-rpc` |

---

## 主要 Namespace 和服務

### `spf` Namespace

| 服務 | 用途 |
|------|------|
| `jarvis-iot-v6` | IoT 通訊服務，處理 Robot/Station 登入和訊息 |

### `application` Namespace

| 服務 | 用途 |
|------|------|
| `app-delivery` | 主要配送服務 |
| `app-delivery-rpc` | RPC 服務，處理任務相關 API |
| `app-delivery-actor-robot` | Robot Actor 服務 |
| `app-delivery-actor-station` | Station Actor 服務 |
| `app-delivery-pubsub` | Pub/Sub 訊息服務 |

---

## 常用抓取命令

### 1. 查看 Pod 列表

```bash
# 查看 spf namespace 的 pods
kubectl get pods -n spf

# 查看 application namespace 的 pods
kubectl get pods -n application

# 查看所有 namespace
kubectl get pods -A
```

### 2. IoT 服務 (jarvis-iot)

```bash
# 抓取最近 1000 行
sudo kubectl -n spf logs -l app=jarvis-iot-v6 --all-containers --tail=1000

# 抓取最近 3 小時
sudo kubectl -n spf logs -l app=jarvis-iot-v6 --all-containers --since=3h

# 過濾特定 Robot
sudo kubectl -n spf logs -l app=jarvis-iot-v6 --all-containers --tail=1000 \
| grep -E "kago5-1393|unit_uid=1010018019"

# 過濾登入相關
sudo kubectl -n spf logs -l app=jarvis-iot-v6 --all-containers --tail=1000 \
| grep -E "RSLoginUnit|UNIT_TYPE_ROBOT|UNIT_TYPE_STATION"
```

### 3. Delivery RPC 服務

```bash
# 抓取特定時間後的 log
sudo kubectl -n application logs <pod-name> --since-time="2025-10-28T15:41:00+08:00"

# 使用 label selector
sudo kubectl -n application logs -l app=app-delivery-rpc --all-containers --tail=500

# 過濾任務相關
sudo kubectl -n application logs -l app=app-delivery-rpc --all-containers --tail=500 \
| grep -E "AssignTask|CompleteTask"
```

### 4. Actor 服務

```bash
# Robot Actor
sudo kubectl -n application logs deploy/app-delivery-actor-robot --all-containers --tail=500

# Station Actor
sudo kubectl -n application logs deploy/app-delivery-actor-station --all-containers --tail=500
```

### 5. Pub/Sub 服務

```bash
sudo kubectl -n application logs -l app=app-delivery-pubsub --all-containers --tail=500
```

---

## 常用過濾關鍵字

### 依功能過濾

```bash
# Robot 登入
grep -E "RSLoginUnit|UNIT_TYPE_ROBOT"

# Station 登入
grep -E "RSLoginUnit|UNIT_TYPE_STATION"

# 任務指派
grep -E "AssignTask"

# 任務完成
grep -E "CompleteTask"

# 特定 unit_uid
grep -E "unit_uid=1010018019"

# 特定設備名稱
grep -E "kago5-1393|station5-067"

# RPC 呼叫
grep -E "\[RPC\]"
```

### 依時間過濾 (在 grep 中)

```bash
# 過濾 15:4x 分的 log
| grep "15:4"

# 過濾特定秒數範圍
| grep -E "15:42:0[0-9]"
```

---

## Log 保存方式

### 方式 1: 直接複製貼上 (推薦)

1. 執行 kubectl 命令
2. 複製終端輸出
3. 在本地建立 `.md` 或 `.log` 檔案
4. 貼上內容並儲存

### 方式 2: 重定向到檔案 + scp 下載

```bash
# 在雲平台上保存
sudo kubectl -n spf logs -l app=jarvis-iot-v6 --all-containers --tail=1000 \
| grep -E "kago5-1393" > /tmp/robot-log.log

# 斷開 SSH 後，在本地執行
scp -i "Aurotek-Taipei-ec2.pem" \
  ubuntu@ec2-43-213-110-250.ap-east-2.compute.amazonaws.com:/tmp/robot-log.log \
  /d/sides/yogo_k8s_onestaion/clue/
```

### 方式 3: 直接重定向 (需要權限)

```bash
sudo kubectl ... > /tmp/output.log 2>&1
```

---

## 檔案命名規範

建議格式：`<服務名稱>_<時間戳記>.md` 或 `.log`

範例：
```
delivery_rpc_1542.md           # 15:42 時間點的 delivery-rpc log
delivery-actor-robot_1632.md   # 16:32 時間點的 robot actor log
jarvis-iot-3h-20251028.log     # 2025-10-28 最近 3 小時的 jarvis-iot log
```

---

## 實用組合命令

### 追蹤特定任務的完整流程

```bash
TASK_UUID="your-task-uuid-here"

# 在 application namespace 追蹤
sudo kubectl -n application logs -l app=app-delivery --all-containers --tail=2000 \
| grep "$TASK_UUID"

# 在 spf namespace 追蹤
sudo kubectl -n spf logs -l app=jarvis-iot-v6 --all-containers --tail=2000 \
| grep "$TASK_UUID"
```

### 查找最近的 task_uuid

```bash
sudo kubectl -n application logs -l app=app-delivery --all-containers --tail=500 \
| grep -E "AssignTask.*unit_uid=1010018019" | head -20
```

### 實時監控

```bash
# 實時監控 jarvis-iot (Ctrl+C 停止)
sudo kubectl -n spf logs -f -l app=jarvis-iot-v6 --all-containers

# 實時監控並過濾
sudo kubectl -n spf logs -f -l app=jarvis-iot-v6 --all-containers \
| grep --line-buffered "kago5-1393"
```

---

## 排錯指令

```bash
# 查看 pod 狀態
kubectl get pods -n spf -o wide
kubectl get pods -n application -o wide

# 查看 pod 詳細資訊
kubectl describe pod <pod-name> -n <namespace>

# 查看最近事件
kubectl get events -n spf --sort-by='.lastTimestamp' | tail -20

# 查看服務
kubectl get svc -n application
```

---

## 注意事項

1. **權限問題**: 大多數命令需要 `sudo`
2. **Pod 名稱變化**: Pod 名稱會隨部署改變，優先使用 `-l app=<name>` label selector
3. **Log 輪替**: K8s log 會輪替，太舊的 log 可能已被清除
4. **輸出量控制**: 使用 `--tail` 限制行數，避免輸出過多
5. **時區**: `--since-time` 需要指定時區，如 `+08:00`

---

## clue 資料夾現有 log 說明

| 檔案 | 來源服務 | 說明 |
|------|----------|------|
| `delivery_rpc_*.md` | app-delivery-rpc | RPC 服務 log |
| `delivery-actor-robot_*.md` | app-delivery-actor-robot | Robot Actor log |
| `delivery_javis_iot_*.md` | jarvis-iot-v6 | IoT 服務 log |
| `app-delivery-pubsub_*.md` | app-delivery-pubsub | Pub/Sub 服務 log |
| `jarvis-iot-3h-*.log` | jarvis-iot-v6 | 3 小時內的 IoT log |
| `k8s-services-list-*.log` | kubectl get | K8s 服務列表 |

---

**建立時間**: 2025-12-10
**參考來源**: `clue/local-terminal-execution-guide.md`
