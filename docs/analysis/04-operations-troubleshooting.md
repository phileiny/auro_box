# 運維和故障排查手冊

**文檔版本**: v1.0
**最後更新**: 2025-10-31
**目標讀者**: 運維工程師、DevOps團隊
**狀態**: ✅ 已完成

---

## 目錄

1. [日常運維](#日常運維)
2. [監控和告警](#監控和告警)
3. [常見故障排查](#常見故障排查)
4. [日誌查詢指南](#日誌查詢指南)
5. [應急響應](#應急響應)
6. [定期維護](#定期維護)

---

## 日常運維

### 1.1 K8s集群管理

#### 連接K8s集群

```bash
# SSH到K8s master節點
ssh -i "Aurotek-Taipei-ec2.pem" ubuntu@ec2-43-213-110-250.ap-east-2.compute.amazonaws.com

# 或者使用端口轉發訪問K8s API
ssh -i "Aurotek-Taipei-ec2.pem" -L 6443:127.0.0.1:6443 ubuntu@ec2-43-213-110-250.ap-east-2.compute.amazonaws.com

# 本地訪問
kubectl get nodes
```

#### 查看服務狀態

```bash
# 查看所有命名空間的Pod
kubectl get pods --all-namespaces

# 查看特定命名空間
kubectl get pods -n application
kubectl get pods -n spf
kubectl get pods -n infrastructure

# 查看服務
kubectl get svc -n application
kubectl get svc -n spf

# 查看Deployment
kubectl get deploy -n application
kubectl get deploy -n spf
```

#### 查看Pod詳情和日誌

```bash
# 查看Pod詳情
kubectl describe pod <pod-name> -n <namespace>

# 查看Pod日誌（實時）
kubectl logs -f <pod-name> -n <namespace>

# 查看多副本Pod的所有日誌
kubectl logs -f -l app=app-delivery-rpc -n application --all-containers

# 查看DAPR sidecar日誌
kubectl logs <pod-name> -c daprd -n <namespace>

# 查看前100行日誌
kubectl logs <pod-name> -n <namespace> --tail=100
```

### 1.2 服務重啟和擴縮容

#### 重啟服務

```bash
# 方法1：刪除Pod讓K8s自動重建
kubectl delete pod <pod-name> -n <namespace>

# 方法2：滾動重啟Deployment
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# 範例：重啟app-delivery-rpc
kubectl rollout restart deployment/app-delivery-rpc -n application
```

#### 擴縮容

```bash
# 擴展到3個副本
kubectl scale deployment/<deployment-name> --replicas=3 -n <namespace>

# 範例：擴展app-delivery-rpc
kubectl scale deployment/app-delivery-rpc --replicas=3 -n application

# 查看擴縮容狀態
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

### 1.3 檢查資源使用

```bash
# 查看節點資源使用
kubectl top nodes

# 查看Pod資源使用
kubectl top pods -n application
kubectl top pods -n spf

# 查看特定Pod的資源使用
kubectl top pod <pod-name> -n <namespace>
```

---

## 監控和告警

### 2.1 Grafana監控

**訪問地址**: `https://robot-api.aurotek.com/grafana`
**登錄憑證**:
- 用戶名: `admin`
- 密碼: `SYbcvyNee7UKpq2poc`

⚠️ **安全提醒**: 建議立即修改默認密碼！

#### 關鍵儀表板（推薦創建）

1. **任務執行監控**
   - 任務創建/完成/失敗趨勢圖
   - 任務平均執行時長
   - 任務成功率

2. **Robot狀態監控**
   - Robot在線數量
   - Robot電量分佈
   - Robot任務負載分佈

3. **服務健康監控**
   - Pod CPU/Memory使用率
   - gRPC請求延遲（p50/p95/p99）
   - DAPR metrics

4. **基礎設施監控**
   - Kafka lag監控
   - Redis連接數和內存使用
   - TimescaleDB連接池狀態

### 2.2 DAPR Metrics

**Metrics端點**: `http://<pod-ip>:9090/metrics`

```bash
# 在Pod內查看DAPR metrics
kubectl exec -it <pod-name> -c daprd -n <namespace> -- curl localhost:9090/metrics

# 關鍵metrics：
# - dapr_http_server_request_count
# - dapr_grpc_io_server_completed_rpcs
# - dapr_runtime_actor_pending_actor_calls
# - dapr_component_pubsub_ingress_count
```

### 2.3 告警設置建議

**Prometheus AlertManager規則**（需創建）：

```yaml
groups:
- name: task-alerts
  rules:
  - alert: TaskFailureRateHigh
    expr: rate(app_delivery_tasks_failed_total[5m]) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "任務失敗率過高"
      description: "最近5分鐘任務失敗率 > 10%"

  - alert: RobotOfflineCount
    expr: jarvis_iot_units_online_total{type="robot"} < 80
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "Robot離線數量異常"
      description: "在線Robot數量少於80台，持續10分鐘"

  - alert: PodCrashLooping
    expr: kube_pod_container_status_restarts_total > 5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Pod重啟次數過多"
      description: "Pod {{ $labels.pod }} 在10分鐘內重啟超過5次"
```

---

## 常見故障排查

### 3.1 任務分配失敗

**症狀**: 訂單創建成功，但任務一直處於CREATED狀態，未分配

**排查步驟**:

```bash
# 1. 查看app-delivery-rpc日誌
kubectl logs -f -l app=app-delivery-rpc -n application --tail=100

# 2. 查看app-delivery-pubsub日誌
kubectl logs -f -l app=app-delivery-pubsub -n application --tail=100

# 3. 查看是否有可用Robot
curl -H "Authorization: Bearer $JWT_TOKEN" \
  https://robot-api.aurotek.com/jarvis-iot/units?site_uid=504&unit_type=ROBOT&is_online=true

# 4. 查看Redis pubsub連接
kubectl exec -it <app-delivery-pubsub-pod> -n application -- \
  redis-cli -h redis ping

# 5. 檢查DAPR sidecar狀態
kubectl logs <app-delivery-rpc-pod> -c daprd -n application --tail=50
```

**可能原因**:
1. ❌ 所有Robot離線或電量不足
2. ❌ DAPR pubsub連接斷開
3. ❌ app-delivery-actor-robot崩潰

**解決方案**:
```bash
# 重啟pubsub服務
kubectl rollout restart deployment/app-delivery-pubsub -n application

# 重啟actor服務
kubectl rollout restart deployment/app-delivery-actor-robot -n application

# 檢查Redis連接
kubectl exec -it redis-0 -n infrastructure -- redis-cli
> CLIENT LIST
```

### 3.2 Robot無法登錄

**症狀**: Robot無法連接到雲平台，LoginUnit失敗

**排查步驟**:

```bash
# 1. 查看jarvis-iot-v6日誌
kubectl logs -f -l app=jarvis-iot-v6 -n spf --tail=100 | grep LoginUnit

# 2. 檢查Traefik路由
kubectl get ingressroute -n spf
kubectl describe ingressroute javis-grpc -n spf

# 3. 檢查jarvis-iot-v6服務
kubectl get svc jarvis-iot-v6-svc -n spf
kubectl get endpoints jarvis-iot-v6-svc -n spf

# 4. 測試gRPC連接
grpcurl -d '{"unit_uid": 1010020021}' \
  -H "Authorization: Bearer $JWT_TOKEN" \
  robot-rpc.aurotek.com:443 \
  jarvis_iot.JarvisIot/LoginUnit
```

**可能原因**:
1. ❌ jarvis-iot-v6服務異常
2. ❌ Traefik路由配置錯誤
3. ❌ Robot端4G網絡問題
4. ❌ JWT Token過期

**解決方案**:
```bash
# 檢查服務健康狀態
kubectl get pods -n spf | grep jarvis-iot-v6

# 重啟jarvis-iot-v6
kubectl rollout restart deployment/jarvis-iot-v6 -n spf

# 檢查Traefik配置
kubectl get configmap traefik-config -n kube-system -o yaml
```

### 3.3 Robot到達Station無法進入

**症狀**: Robot到達Station艙門，但無法開門進入

**排查步驟**:

```bash
# 1. 確認2.4G通信是否正常（關鍵！）
# 物理檢查2.4G設備電源

# 2. 查看Station日誌（process_unit_event錯誤）
kubectl logs -f -l app=jarvis-iot-v6 -n spf --tail=100 | grep process_unit_event

# 3. 查看任務狀態
curl -H "Authorization: Bearer $JWT_TOKEN" \
  https://robot-api.aurotek.com/app-delivery/tasks/{task_id}

# 4. 查看Robot狀態
curl -H "Authorization: Bearer $JWT_TOKEN" \
  https://robot-api.aurotek.com/jarvis-iot/units/{robot_uid}
```

**可能原因**（基於實驗驗證）:
1. 🔴 **2.4G通信斷開**（最常見，影響最大）
2. ❌ Station艙門機械故障
3. ❌ Robot-Station身份驗證失敗

**解決方案**:
```bash
# 1. 檢查2.4G設備電源和連接
# 物理排查

# 2. 重新分配任務，繞過Station
# 系統會自動切換為Robot直接模式

# 3. 重啟Station相關服務
# （如果是軟件問題）
```

### 3.4 任務一直執行中，無法完成

**症狀**: 任務狀態一直是EXECUTING，超過預期時間未完成

**排查步驟**:

```bash
# 1. 查看任務詳情
curl -H "Authorization: Bearer $JWT_TOKEN" \
  https://robot-api.aurotek.com/app-delivery/tasks/{task_id}

# 2. 查看Robot實時狀態
kubectl exec -it redis-0 -n infrastructure -- redis-cli
> HGETALL UnitConsensus@s504.u{robot_uid}

# 3. 查看Robot Actor日誌
kubectl logs -f -l app=app-delivery-actor-robot -n application --tail=100

# 4. 查看SyncMeteorEvents是否正常上報
kubectl logs -f -l app=jarvis-iot-v6 -n spf --tail=100 | grep SyncMeteorEvents
```

**可能原因**:
1. ❌ Robot卡在某個位置（導航障礙）
2. ❌ Robot與雲端通信中斷
3. ❌ Robot本地程序異常

**解決方案**:
```bash
# 1. 手動標記任務失敗（需連接數據庫）
psql -h tsdb-timescaledb-0 -U postgres -d app_aiyo -p 30432
UPDATE t_app_task_ledger2 SET status='TIMEOUT' WHERE task_id={task_id};

# 2. 重新分配任務給其他Robot

# 3. 檢查Robot本地日誌（需SSH到Robot）
```

### 3.5 數據庫連接池耗盡

**症狀**: 服務日誌出現大量"database connection pool exhausted"錯誤

**排查步驟**:

```bash
# 1. 查看TimescaleDB連接數
kubectl exec -it tsdb-timescaledb-0 -n infrastructure -- \
  psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"

# 2. 查看連接詳情
kubectl exec -it tsdb-timescaledb-0 -n infrastructure -- \
  psql -U postgres -c "SELECT datname, usename, application_name, state, count(*)
                        FROM pg_stat_activity
                        GROUP BY datname, usename, application_name, state;"

# 3. 查看長時間運行的查詢
kubectl exec -it tsdb-timescaledb-0 -n infrastructure -- \
  psql -U postgres -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
                        FROM pg_stat_activity
                        WHERE state != 'idle'
                        ORDER BY duration DESC;"
```

**解決方案**:
```bash
# 1. 殺死長時間運行的查詢
kubectl exec -it tsdb-timescaledb-0 -n infrastructure -- \
  psql -U postgres -c "SELECT pg_terminate_backend(<pid>);"

# 2. 重啟使用數據庫的服務（釋放連接）
kubectl rollout restart deployment/app-delivery-rpc -n application

# 3. 增加數據庫連接池大小（修改Deployment環境變數）
# DB_POOL_MAX_CONNECTIONS: "50" → "100"
```

---

## 日誌查詢指南

### 4.1 阿里雲Log Service查詢

**訪問**: 阿里雲控制台 → 日誌服務 → [Project Name]

#### 查詢任務相關日誌

```sql
-- 查詢特定任務的所有日誌
* | WHERE content LIKE '%task_id:67985120%'
  AND __time__ > to_unixtime(now()) - 3600
  ORDER BY __time__ DESC
  LIMIT 100

-- 查詢特定Robot的日誌
* | WHERE content LIKE '%u1010020021%'
  AND __time__ > to_unixtime(now()) - 3600
  ORDER BY __time__ DESC

-- 查詢任務分配日誌
* | WHERE content LIKE '%AssignTask%'
  OR content LIKE '%task_update_subscribe%'
  ORDER BY __time__ DESC
  LIMIT 50

-- 統計錯誤日誌
* | WHERE content LIKE '%ERROR%' OR content LIKE '%Exception%'
  | SELECT date_format(__time__, '%Y-%m-%d %H:%i') as time_bucket,
           count(*) as error_count
  GROUP BY time_bucket
  ORDER BY time_bucket DESC
```

#### 查詢TaskUpdateSignal事件

```sql
-- 查詢TaskUpdateSignal發布
* | WHERE content LIKE '%DAPR Publish T_TaskUpdateSignal%'
  ORDER BY __time__ DESC
  LIMIT 20

-- 查詢任務完成事件
* | WHERE content LIKE '%TaskUpdateSignal%'
  AND content LIKE '%COMPLETED%'
  ORDER BY __time__ DESC
```

### 4.2 直接查詢K8s Pod日誌

#### 實時日誌

```bash
# 查詢app-delivery-rpc日誌，過濾任務ID
kubectl logs -f -l app=app-delivery-rpc -n application --all-containers \
  | grep "task_id:67985120"

# 查詢jarvis-iot-v6日誌，過濾Robot UID
kubectl logs -f -l app=jarvis-iot-v6 -n spf --all-containers \
  | grep "u1010020021"

# 查詢DAPR事件發布
kubectl logs -f -l app=app-delivery-actor-robot -n application \
  | grep "EventSourcing.*DAPR Publish"
```

#### 歷史日誌

```bash
# 查看過去1小時的日誌
kubectl logs -l app=app-delivery-rpc -n application --all-containers \
  --since=1h > /tmp/app-delivery-rpc.log

# 分析日誌文件
cat /tmp/app-delivery-rpc.log | grep ERROR
cat /tmp/app-delivery-rpc.log | grep "task_id:67985120"
```

### 4.3 追蹤完整任務流程

**使用task_uuid追蹤**（手動關聯多個服務日誌）：

```bash
# Step 1: 從任務創建開始
kubectl logs -l app=app-delivery-rpc -n application --all-containers --tail=1000 \
  | grep "task_uuid:d25d1020-d158-431c-bfd3-106b4ce7618e"

# Step 2: 查看任務分配
kubectl logs -l app=app-delivery-pubsub -n application --all-containers --tail=1000 \
  | grep "s504.k67985120.t67985120"

# Step 3: 查看Robot執行
kubectl logs -l app=app-delivery-actor-robot -n application --all-containers --tail=1000 \
  | grep "task_uuid:d25d1020-d158-431c-bfd3-106b4ce7618e"

# Step 4: 查看IoT層通信
kubectl logs -l app=jarvis-iot-v6 -n spf --all-containers --tail=1000 \
  | grep "u1010020021"

# Step 5: 查看任務完成
kubectl logs -l app=app-delivery-rpc -n application --all-containers --tail=1000 \
  | grep "task_id:67985120.*COMPLETED"
```

---

## 應急響應

### 5.1 緊急事件等級

| 等級 | 定義 | 響應時間 | 範例 |
|-----|------|---------|------|
| P0 - 嚴重 | 系統完全不可用 | 15分鐘 | 所有任務無法創建 |
| P1 - 緊急 | 核心功能不可用 | 1小時 | Robot無法登錄 |
| P2 - 高 | 部分功能受影響 | 4小時 | 特定Robot故障 |
| P3 - 中 | 性能下降 | 1天 | 任務延遲增加 |
| P4 - 低 | 輕微問題 | 1週 | 日誌錯誤 |

### 5.2 P0事件：系統完全不可用

**症狀**: 無法創建訂單/任務，所有Robot離線

**緊急響應流程**:

```bash
# 1. 確認問題範圍
kubectl get pods --all-namespaces | grep -v Running

# 2. 檢查關鍵服務
kubectl get pods -n application | grep "app-delivery\|app-shop"
kubectl get pods -n spf | grep "jarvis-iot"

# 3. 檢查基礎設施
kubectl get pods -n infrastructure | grep "kafka\|redis\|tsdb"

# 4. 如果大面積Pod異常，檢查節點
kubectl get nodes
kubectl describe node <node-name>

# 5. 快速恢復：重啟關鍵服務
kubectl rollout restart deployment/app-delivery-rpc -n application
kubectl rollout restart deployment/app-shop-rpc -n application
kubectl rollout restart deployment/jarvis-iot-v6 -n spf

# 6. 通知相關人員
# 發送告警郵件/Slack通知
```

### 5.3 P1事件：Robot無法登錄

**快速恢復步驟**:

```bash
# 1. 重啟jarvis-iot-v6（最常見解決方案）
kubectl rollout restart deployment/jarvis-iot-v6 -n spf

# 2. 檢查Traefik
kubectl get pods -n kube-system | grep traefik
kubectl logs -f <traefik-pod> -n kube-system | grep jarvis_iot

# 3. 如果Traefik異常，重啟
kubectl delete pod <traefik-pod> -n kube-system

# 4. 驗證恢復
grpcurl robot-rpc.aurotek.com:443 list
```

### 5.4 回滾Deployment

```bash
# 查看Deployment歷史
kubectl rollout history deployment/<deployment-name> -n <namespace>

# 回滾到上一版本
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# 回滾到特定版本
kubectl rollout undo deployment/<deployment-name> --to-revision=<revision> -n <namespace>

# 範例：回滾app-delivery-rpc
kubectl rollout history deployment/app-delivery-rpc -n application
kubectl rollout undo deployment/app-delivery-rpc -n application
```

---

## 定期維護

### 6.1 每日檢查清單

```bash
# 1. 檢查Pod健康狀態
kubectl get pods --all-namespaces | grep -v Running

# 2. 檢查Robot在線數量
curl -H "Authorization: Bearer $JWT_TOKEN" \
  https://robot-api.aurotek.com/jarvis-iot/stats/robots?site_uid=504 \
  | jq '.summary.total_online_robots'

# 3. 檢查任務統計
curl -H "Authorization: Bearer $JWT_TOKEN" \
  https://robot-api.aurotek.com/app-delivery/stats/tasks?site_uid=504&date=$(date +%Y-%m-%d) \
  | jq '.completion_rate'

# 4. 檢查數據庫連接數
kubectl exec -it tsdb-timescaledb-0 -n infrastructure -- \
  psql -U postgres -c "SELECT count(*) FROM pg_stat_activity;"

# 5. 檢查Kafka lag
kubectl exec -it kafka-0 -n infrastructure -- \
  kafka-consumer-groups.sh --describe --group prod-salt-events-cmdb-consumer-group \
  --bootstrap-server localhost:9092
```

### 6.2 每週維護

```bash
# 1. 清理已完成的Cronjob
kubectl delete job -n spf $(kubectl get jobs -n spf | grep Completed | awk '{print $1}')

# 2. 檢查磁盤使用
kubectl exec -it tsdb-timescaledb-0 -n infrastructure -- df -h

# 3. 數據庫健康檢查
kubectl exec -it tsdb-timescaledb-0 -n infrastructure -- \
  psql -U postgres -d app_aiyo -c "SELECT pg_database_size('app_aiyo');"

# 4. 檢查日誌保留期配置
kubectl get pods -n application -o yaml | grep aliyun_logs.*_ttl
```

### 6.3 每月維護

```bash
# 1. 審查資源配額
kubectl describe resourcequota -n application
kubectl describe resourcequota -n spf

# 2. 更新Docker鏡像（根據發布計劃）
# 使用Argo Rollouts漸進式發布

# 3. 數據庫維護
kubectl exec -it tsdb-timescaledb-0 -n infrastructure -- \
  psql -U postgres -d app_aiyo -c "VACUUM ANALYZE t_app_task_ledger2;"

# 4. 審查告警規則和儀表板
# 訪問Grafana，檢查是否有新的監控需求
```

---

## 附錄

### A. 快速參考

**常用命令**:
```bash
# 查看所有Pod
kubectl get pods -A

# 查看服務日誌
kubectl logs -f <pod-name> -n <namespace>

# 進入Pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# 查看Pod資源使用
kubectl top pod <pod-name> -n <namespace>

# 重啟服務
kubectl rollout restart deployment/<deployment-name> -n <namespace>
```

**關鍵端點**:
- Grafana: `https://robot-api.aurotek.com/grafana`
- API: `https://robot-api.aurotek.com`
- gRPC: `robot-rpc.aurotek.com:443`

### B. 緊急聯絡

**升級路徑**:
1. 值班運維工程師
2. 技術Lead
3. 架構師
4. CTO

**外部支持**:
- 阿里雲技術支持
- K8s社群

---

## 相關文檔

- [系統架構總覽](./01-system-architecture-overview.md)
- [任務閉環流程技術文檔](./02-task-loop-technical-doc.md)
- [API參考文檔](./03-api-reference.md)
- [最佳實踐和改進建議](./05-best-practices-improvements.md)

---

**維護提醒**: 本手冊應隨系統演進持續更新，建議每季度審查一次。
