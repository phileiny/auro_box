# 最佳實踐和改進建議

**文檔版本**: v1.0
**最後更新**: 2025-10-31
**目標讀者**: 開發團隊、架構師、技術Lead
**狀態**: ✅ 已完成

---

## 目錄

1. [系統設計最佳實踐](#系統設計最佳實踐)
2. [短期改進建議（1-2週）](#短期改進建議)
3. [中期改進建議（1-2個月）](#中期改進建議)
4. [長期改進建議（3-6個月）](#長期改進建議)
5. [性能優化建議](#性能優化建議)
6. [安全加固建議](#安全加固建議)
7. [開發規範](#開發規範)

---

## 系統設計最佳實踐

### 1.1 現有架構亮點 ⭐

#### 1. 雙層通信架構
**亮點**: 4G雲端調度 + 2.4G本地協作
**優勢**:
- ✅ 雲端統一調度，全局最優
- ✅ 本地協作低延遲，實時控制
- ✅ 分層解耦，各司其職

**最佳實踐**:
```python
# 明確區分雲端和本地職責

# 雲端職責（4G gRPC）
class CloudController:
    def assign_task(self, robot_uid, task):
        """任務分配和調度"""
        pass

    def sync_status(self, robot_uid, status):
        """狀態同步和監控"""
        pass

# 本地職責（2.4G）
class LocalController:
    def authenticate_robot(self, robot_id):
        """Robot身份驗證"""
        pass

    def control_door(self, action):
        """艙門實時控制"""
        pass
```

#### 2. 事件驅動架構
**亮點**: DAPR + TaskUpdateSignal統一事件

**優勢**:
- ✅ 發布者和訂閱者解耦
- ✅ 易於水平擴展
- ✅ 支持異步處理

**最佳實踐**:
```python
# 遵循事件驅動模式

# ✅ 好的做法：使用DAPR Pubsub
await dapr_client.publish_event(
    pubsub_name="pubsub",
    topic_name="task_update_signal",
    data={"task_id": 123, "status": "COMPLETED"}
)

# ❌ 避免：直接RPC調用多個服務
# await notification_service.notify(task_id)
# await analytics_service.record(task_id)
# await monitoring_service.log(task_id)
```

#### 3. Actor模式狀態管理
**亮點**: 每個Robot/Station獨立Actor實例

**優勢**:
- ✅ 狀態變更集中管理
- ✅ 避免分散式鎖
- ✅ 易於推理和調試

**最佳實踐**:
```python
# Actor ID命名規範
actor_id = f"s{site_uid}.u{unit_uid}"  # 例: s504.u1010020021

# Actor內部狀態管理
class RobotActorV2:
    def __init__(self, actor_id):
        self.actor_id = actor_id
        self.state = {}  # Actor內部狀態

    async def ping(self):
        """每秒更新狀態到Redis"""
        await self.update_redis()

    async def update_redis(self):
        """統一的狀態更新入口"""
        await redis.hset(
            f"UnitConsensus@{self.actor_id}",
            mapping=self.state
        )
```

#### 4. 多層數據同步
**亮點**: DAPR EventBus + TimescaleDB + Redis + WebSocket

**優勢**:
- ✅ 實時性（DAPR、Redis、WebSocket）
- ✅ 持久化（TimescaleDB）
- ✅ 快速查詢（Redis）
- ✅ 歷史分析（TimescaleDB時序優化）

**最佳實踐**:
```python
# 寫入策略：先寫DB，再更新Cache

async def complete_task(task_id):
    # 1. 持久化到數據庫
    await db.update_task(task_id, status='COMPLETED')

    # 2. 更新Redis快取
    await redis.hset(f"task:{task_id}", "status", "COMPLETED")

    # 3. 發布事件通知
    await dapr.publish_event("task_update_signal", {...})

# 讀取策略：優先從Cache

async def get_task_status(task_id):
    # 1. 先查Redis
    cached = await redis.hgetall(f"task:{task_id}")
    if cached:
        return cached

    # 2. Cache miss，查數據庫
    task = await db.get_task(task_id)

    # 3. 寫入Cache
    await redis.hset(f"task:{task_id}", mapping=task)

    return task
```

---

## 短期改進建議

### 2.1 整合分散式追蹤 🔴高優先

**問題**: 當前缺少分散式追蹤，難以追蹤跨服務調用鏈路

**影響**:
- ⚠️ 性能瓶頸難以定位
- ⚠️ 故障排查耗時長
- ⚠️ 依賴手動關聯task_uuid

**解決方案**: 整合DAPR + Zipkin

**實施步驟**:

#### Step 1: 部署Zipkin（預估：1天）

```yaml
# zipkin-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zipkin
  namespace: infrastructure
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zipkin
  template:
    metadata:
      labels:
        app: zipkin
    spec:
      containers:
      - name: zipkin
        image: openzipkin/zipkin:latest
        ports:
        - containerPort: 9411
        env:
        - name: STORAGE_TYPE
          value: "elasticsearch"  # 可選，持久化存儲
        - name: ES_HOSTS
          value: "http://elasticsearch:9200"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
---
apiVersion: v1
kind: Service
metadata:
  name: zipkin
  namespace: infrastructure
spec:
  selector:
    app: zipkin
  ports:
  - port: 9411
    targetPort: 9411
  type: ClusterIP
```

#### Step 2: 配置DAPR追蹤（預估：0.5天）

```yaml
# dapr-config-tracing.yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: tracing
  namespace: application
spec:
  tracing:
    samplingRate: "0.1"  # 10%採樣（生產環境）
    zipkin:
      endpointAddress: "http://zipkin.infrastructure.svc:9411/api/v2/spans"
```

#### Step 3: 更新Deployment使用追蹤配置（預估：0.5天）

```yaml
# 在所有Deployment的annotations中添加
annotations:
  dapr.io/config: "tracing"
```

#### Step 4: 應用層添加Trace Context傳播（預估：1天）

```python
# Python範例（使用OpenTelemetry）
from opentelemetry import trace
from opentelemetry.exporter.zipkin.json import ZipkinExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# 初始化Tracer
zipkin_exporter = ZipkinExporter(
    endpoint="http://zipkin.infrastructure.svc:9411/api/v2/spans"
)
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(zipkin_exporter)
)

tracer = trace.get_tracer(__name__)

# 在業務代碼中添加Span
async def assign_task(task_id, unit_uid):
    with tracer.start_as_current_span("assign_task") as span:
        span.set_attribute("task_id", task_id)
        span.set_attribute("unit_uid", unit_uid)

        # 業務邏輯
        await db.update_task(task_id, unit_uid=unit_uid)
        await dapr.publish_event(...)

        span.add_event("task_assigned")
```

**預期收益**:
- ✅ 端到端追蹤任務流程
- ✅ 識別性能瓶頸
- ✅ 快速定位故障
- ✅ 可視化服務依賴

**總工作量**: 2-3天
**投資回報**: ⭐⭐⭐⭐⭐ 極高

### 2.2 增加日誌保留期 🟡中優先

**問題**: 當前日誌保留期僅1天，無法回溯歷史問題

**解決方案**: 增加到7天

**實施步驟**:

```yaml
# 修改所有關鍵服務的日誌TTL
env:
  - name: aliyun_logs_app-delivery-rpc_ttl
    value: "7"  # 從1天改為7天

  - name: aliyun_logs_jarvis-iot-v6_ttl
    value: "7"

  - name: aliyun_logs_app-delivery-actor-robot_ttl
    value: "7"
```

**成本影響**: 日誌存儲費用增加約7倍（估算：$50/月 → $350/月）

**總工作量**: 0.5天
**投資回報**: ⭐⭐⭐

### 2.3 部署Prometheus Server 🟡中優先

**問題**: 當前僅有DAPR Sidecar metrics，缺少集中式Prometheus Server

**解決方案**: 部署Prometheus + AlertManager

**實施步驟**（使用kube-prometheus-stack）:

```bash
# 使用Helm安裝
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace infrastructure \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

**總工作量**: 1天
**投資回報**: ⭐⭐⭐⭐

---

## 中期改進建議

### 3.1 配置死信佇列 🟡中優先

**問題**: 失敗任務處理依賴Cronjob，無明確DLQ

**解決方案**: DAPR Resiliency + Kafka DLQ

#### 方案A: DAPR Resiliency

```yaml
# dapr-resiliency.yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: app-resiliency
  namespace: application
spec:
  policies:
    retries:
      task-retry:
        policy: exponential
        maxRetries: 3
        initialInterval: 1s
        maxInterval: 30s
        multiplier: 2

    circuitBreakers:
      task-circuit-breaker:
        maxRequests: 5
        timeout: 30s
        trip: consecutiveFailures > 3

    timeouts:
      task-timeout:
        general: 60s

  targets:
    actors:
      RobotActorV2:
        retry: task-retry
        circuitBreaker: task-circuit-breaker
        timeout: task-timeout

    apps:
      app-delivery-rpc:
        retry: task-retry
        circuitBreaker: task-circuit-breaker
```

#### 方案B: Kafka DLQ Topic

```python
# 創建DLQ Topic
kafka-topics.sh --create \
  --bootstrap-server kafka:9092 \
  --topic prod-task-dlq \
  --partitions 3 \
  --replication-factor 2

# 應用代碼處理
async def process_task_event(event):
    try:
        await handle_task_update(event)
    except Exception as e:
        # 重試3次後發送到DLQ
        if retry_count >= 3:
            await kafka_producer.send(
                topic='prod-task-dlq',
                value=event,
                headers={
                    'original-topic': 'task_update_signal',
                    'error-message': str(e),
                    'retry-count': str(retry_count)
                }
            )
```

**總工作量**: 1-2週
**投資回報**: ⭐⭐⭐⭐

### 3.2 實作告警系統 🟡中優先

**解決方案**: Prometheus AlertManager + Grafana

**關鍵告警規則**:

```yaml
# prometheus-alerts.yaml
groups:
- name: task-alerts
  rules:
  # 任務失敗率告警
  - alert: TaskFailureRateHigh
    expr: |
      rate(app_delivery_tasks_failed_total[5m])
      /
      rate(app_delivery_tasks_created_total[5m])
      > 0.1
    for: 5m
    labels:
      severity: warning
      team: backend
    annotations:
      summary: "任務失敗率過高: {{ $value | humanizePercentage }}"
      description: "Site {{ $labels.site_uid }} 最近5分鐘任務失敗率超過10%"
      runbook: "https://docs.company.com/runbooks/task-failure-rate-high"

  # Robot離線告警
  - alert: RobotOfflineCount
    expr: |
      sum(jarvis_iot_units_online_total{type="robot"})
      < 80
    for: 10m
    labels:
      severity: critical
      team: operations
    annotations:
      summary: "Robot離線數量異常"
      description: "在線Robot數量: {{ $value }}，少於80台"

  # Pod重啟告警
  - alert: PodRestartingTooOften
    expr: |
      rate(kube_pod_container_status_restarts_total[1h]) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} 重啟過於頻繁"
      description: "最近1小時平均每10分鐘重啟1次"

  # 數據庫連接池告警
  - alert: DatabaseConnectionPoolExhausted
    expr: |
      pg_stat_database_connections{datname="app_aiyo"}
      > 90
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "數據庫連接池接近耗盡"
      description: "當前連接數: {{ $value }}/100"

  # Kafka Lag告警
  - alert: KafkaConsumerLagHigh
    expr: |
      kafka_consumer_group_lag{group="prod-salt-events-cmdb-consumer-group"}
      > 10000
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Kafka消費延遲過高"
      description: "Consumer group {{ $labels.group }} lag: {{ $value }}"
```

**告警通知配置**:

```yaml
# alertmanager-config.yaml
receivers:
- name: 'team-backend'
  email_configs:
  - to: 'backend-team@company.com'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
    channel: '#backend-alerts'

- name: 'team-operations'
  email_configs:
  - to: 'ops-team@company.com'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
    channel: '#ops-alerts'
  pagerduty_configs:
  - service_key: 'PAGERDUTY_SERVICE_KEY'

route:
  receiver: 'team-backend'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  routes:
  - match:
      severity: critical
    receiver: 'team-operations'
    repeat_interval: 5m
```

**總工作量**: 1週
**投資回報**: ⭐⭐⭐⭐

### 3.3 優化監控儀表板 🟢低優先

**推薦的Grafana Dashboard**:

#### Dashboard 1: 任務執行總覽

```json
{
  "title": "任務執行總覽",
  "panels": [
    {
      "title": "任務創建/完成趨勢",
      "targets": [
        {
          "expr": "rate(app_delivery_tasks_created_total[5m])"
        },
        {
          "expr": "rate(app_delivery_tasks_completed_total[5m])"
        }
      ]
    },
    {
      "title": "任務成功率",
      "targets": [
        {
          "expr": "rate(app_delivery_tasks_completed_total[5m]) / rate(app_delivery_tasks_created_total[5m])"
        }
      ]
    },
    {
      "title": "平均任務時長 (秒)",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, app_delivery_task_duration_seconds_bucket)"
        }
      ]
    }
  ]
}
```

**總工作量**: 2-3天
**投資回報**: ⭐⭐⭐

---

## 長期改進建議

### 4.1 遷移到OpenTelemetry 🟡中優先

**目標**: 統一Metrics/Logs/Traces採集

**優勢**:
- ✅ 供應商中立，避免鎖定
- ✅ 統一SDK和API
- ✅ 自動化Instrumentation
- ✅ 與DAPR天然集成

**實施計劃**（3-6個月）:

#### Phase 1: 基礎設施準備（1個月）

```yaml
# 部署OpenTelemetry Collector
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: infrastructure
spec:
  mode: daemonset
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
      prometheus:
        config:
          scrape_configs:
          - job_name: 'dapr-sidecars'
            kubernetes_sd_configs:
            - role: pod

    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024

      attributes:
        actions:
        - key: cluster
          value: prod-k8s
          action: insert

    exporters:
      prometheus:
        endpoint: "prometheus:9090"
      jaeger:
        endpoint: "jaeger:14250"
        tls:
          insecure: true
      logging:
        loglevel: debug

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, attributes]
          exporters: [jaeger, logging]
        metrics:
          receivers: [otlp, prometheus]
          processors: [batch]
          exporters: [prometheus]
```

#### Phase 2: 應用層整合（2個月）

```python
# Python應用整合OpenTelemetry
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.grpc import GrpcInstrumentorClient, GrpcInstrumentorServer
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor

# 初始化
trace.set_tracer_provider(TracerProvider())
otlp_exporter = OTLPSpanExporter(endpoint="otel-collector:4317")
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(otlp_exporter)
)

# 自動Instrumentation
FastAPIInstrumentor().instrument()
GrpcInstrumentorClient().instrument()
GrpcInstrumentorServer().instrument()
RedisInstrumentor().instrument()
Psycopg2Instrumentor().instrument()

# 手動Span
tracer = trace.get_tracer(__name__)

async def assign_task(task_id, unit_uid):
    with tracer.start_as_current_span("assign_task") as span:
        span.set_attribute("task_id", task_id)
        span.set_attribute("unit_uid", unit_uid)

        # 自動記錄DB查詢、Redis操作、gRPC調用
        await db.update_task(...)
        await redis.hset(...)
        await actor_client.invoke_actor(...)
```

#### Phase 3: 優化和推廣（3個月）

- 為所有微服務添加OpenTelemetry
- 優化採樣率和性能
- 建立監控SOP

**總工作量**: 3-6個月
**投資回報**: ⭐⭐⭐⭐⭐

### 4.2 實施SLO/SLI監控 🟢低優先

**定義服務級別指標（SLI）**:

```yaml
# SLI定義
SLIs:
  - name: task_success_rate
    description: "任務成功率"
    query: |
      sum(rate(app_delivery_tasks_completed_total[5m]))
      /
      sum(rate(app_delivery_tasks_created_total[5m]))

  - name: api_latency_p95
    description: "API響應時間 P95"
    query: |
      histogram_quantile(0.95,
        rate(http_request_duration_seconds_bucket[5m])
      )

  - name: robot_availability
    description: "Robot可用率"
    query: |
      sum(jarvis_iot_units_online_total{type="robot"})
      /
      sum(jarvis_iot_units_total{type="robot"})
```

**定義服務級別目標（SLO）**:

```yaml
# SLO定義
SLOs:
  - name: task_success_rate_slo
    sli: task_success_rate
    target: 0.995  # 99.5%
    window: 30d

  - name: api_latency_slo
    sli: api_latency_p95
    target: 0.5  # 500ms
    window: 30d

  - name: robot_availability_slo
    sli: robot_availability
    target: 0.95  # 95%
    window: 7d
```

**總工作量**: 1-2個月
**投資回報**: ⭐⭐⭐⭐

---

## 性能優化建議

### 5.1 數據庫優化

#### 索引優化

```sql
-- 為常用查詢添加索引
CREATE INDEX CONCURRENTLY idx_task_status_created
ON t_app_task_ledger2 (status, created_at DESC);

CREATE INDEX CONCURRENTLY idx_task_unit_status
ON t_app_task_ledger2 (unit_uid, status);

CREATE INDEX CONCURRENTLY idx_task_site_date
ON t_app_task_ledger2 (site_uid, created_at DESC);

-- TimescaleDB連續聚合（加速統計查詢）
CREATE MATERIALIZED VIEW task_hourly_stats
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', created_at) AS bucket,
  site_uid,
  status,
  COUNT(*) AS task_count,
  AVG(EXTRACT(EPOCH FROM (completed_at - assigned_at))) AS avg_duration_seconds
FROM t_app_task_ledger2
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY bucket, site_uid, status;

-- 自動刷新
SELECT add_continuous_aggregate_policy('task_hourly_stats',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');
```

#### 查詢優化

```python
# ❌ N+1查詢問題
async def get_tasks_with_orders():
    tasks = await db.query("SELECT * FROM t_app_task_ledger2 WHERE status='EXECUTING'")
    for task in tasks:
        order = await db.query("SELECT * FROM t_app_order WHERE task_id=?", task.id)  # N次查詢
        task.order = order

# ✅ 使用JOIN
async def get_tasks_with_orders():
    results = await db.query("""
        SELECT t.*, o.*
        FROM t_app_task_ledger2 t
        LEFT JOIN t_app_order o ON o.task_id = t.task_id
        WHERE t.status = 'EXECUTING'
    """)
    return results
```

### 5.2 Redis優化

```python
# ❌ 多次往返
async def get_robot_info(unit_uid):
    space = await redis.hget(f"UnitConsensus@s504.u{unit_uid}", "space")
    purpose = await redis.hget(f"UnitConsensus@s504.u{unit_uid}", "purpose")
    battery = await redis.hget(f"UnitConsensus@s504.u{unit_uid}", "battery")
    # 3次網絡往返

# ✅ 使用Pipeline
async def get_robot_info(unit_uid):
    pipe = redis.pipeline()
    pipe.hgetall(f"UnitConsensus@s504.u{unit_uid}")
    result = await pipe.execute()
    return result[0]  # 1次網絡往返
```

### 5.3 gRPC優化

```python
# ✅ 啟用gRPC壓縮
channel = grpc.aio.insecure_channel(
    'robot-rpc.aurotek.com:443',
    options=[
        ('grpc.default_compression_algorithm', grpc.Compression.Gzip),
        ('grpc.grpc.default_compression_level', grpc.CompressionLevel.high),
    ]
)

# ✅ 啟用KeepAlive
channel = grpc.aio.insecure_channel(
    'robot-rpc.aurotek.com:443',
    options=[
        ('grpc.keepalive_time_ms', 30000),
        ('grpc.keepalive_timeout_ms', 10000),
        ('grpc.keepalive_permit_without_calls', True),
    ]
)

# ✅ 連接池
class GrpcClientPool:
    def __init__(self, size=10):
        self.channels = [
            grpc.aio.insecure_channel('robot-rpc.aurotek.com:443')
            for _ in range(size)
        ]
        self.index = 0

    def get_channel(self):
        channel = self.channels[self.index]
        self.index = (self.index + 1) % len(self.channels)
        return channel
```

---

## 安全加固建議

### 6.1 密碼管理 🔴高優先

**問題**: Grafana密碼明文存在ConfigMap

```yaml
# ❌ 當前做法
env:
  - name: GF_SECURITY_ADMIN_PASSWORD
    value: "SYbcvyNee7UKpq2poc"  # 明文密碼
```

**解決方案**: 使用Kubernetes Secret

```yaml
# 創建Secret
kubectl create secret generic grafana-admin \
  --from-literal=password=$(openssl rand -base64 32) \
  -n infrastructure

# 使用Secret
env:
  - name: GF_SECURITY_ADMIN_PASSWORD
    valueFrom:
      secretKeyRef:
        name: grafana-admin
        key: password
```

**總工作量**: 1天
**投資回報**: ⭐⭐⭐⭐⭐

### 6.2 網絡策略 🟡中優先

```yaml
# 限制namespace間通信
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: application-network-policy
  namespace: application
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: kube-system  # 允許Traefik
    - namespaceSelector:
        matchLabels:
          name: spf  # 允許IoT層
    ports:
    - protocol: TCP
      port: 5000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: infrastructure  # 允許訪問基礎設施
```

### 6.3 RBAC加固 🟡中優先

```yaml
# 最小權限原則
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-delivery-rpc-sa
  namespace: application
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-delivery-rpc-role
  namespace: application
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]  # 僅讀權限
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-delivery-rpc-rolebinding
  namespace: application
subjects:
- kind: ServiceAccount
  name: app-delivery-rpc-sa
roleRef:
  kind: Role
  name: app-delivery-rpc-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 開發規範

### 7.1 代碼規範

#### Python代碼風格

```python
# 使用type hints
async def assign_task(
    task_id: int,
    unit_uid: int,
    task_type: str
) -> Dict[str, Any]:
    """分配任務給Robot或Station

    Args:
        task_id: 任務ID
        unit_uid: 設備UID
        task_type: 任務類型（DELIVERY, PATROL等）

    Returns:
        Dict包含success和message

    Raises:
        RobotNotAvailableException: 無可用Robot
    """
    pass

# 使用asyncio最佳實踐
# ✅ 好的做法
tasks = [
    assign_task(1, 1001),
    assign_task(2, 1002),
    assign_task(3, 1003),
]
results = await asyncio.gather(*tasks)  # 並行執行

# ❌ 避免
for task_id, unit_uid in assignments:
    await assign_task(task_id, unit_uid)  # 順序執行，慢
```

#### gRPC Proto規範

```protobuf
// ✅ 好的做法
syntax = "proto3";

package app_delivery.v2;

option go_package = "github.com/company/app-delivery/v2;app_delivery_v2";

// 使用明確的消息名稱
message AssignTaskRequest {
  int64 task_id = 1;
  int64 unit_uid = 2;
  TaskType task_type = 3;  // 使用枚舉
  google.protobuf.Timestamp deadline = 4;  // 使用標準類型
  map<string, string> metadata = 5;
}

enum TaskType {
  TASK_TYPE_UNSPECIFIED = 0;
  TASK_TYPE_DELIVERY = 1;
  TASK_TYPE_PATROL = 2;
  TASK_TYPE_MAINTENANCE = 3;
}
```

### 7.2 Git提交規範

```bash
# 使用Conventional Commits
feat(task): 添加任務重試機制
fix(robot): 修復Robot登錄失敗問題
docs(api): 更新gRPC API文檔
refactor(actor): 重構Actor狀態管理
test(task): 添加任務分配單元測試
chore(deps): 升級DAPR到v1.10
```

### 7.3 監控指標規範

```python
# 使用Prometheus client庫
from prometheus_client import Counter, Histogram, Gauge

# 定義metrics
task_created_total = Counter(
    'app_delivery_tasks_created_total',
    'Total number of tasks created',
    ['site_uid', 'task_type']
)

task_duration_seconds = Histogram(
    'app_delivery_task_duration_seconds',
    'Task execution duration in seconds',
    ['site_uid', 'task_type', 'status'],
    buckets=[10, 30, 60, 120, 300, 600, 1800, 3600]
)

robot_online_count = Gauge(
    'jarvis_iot_robot_online_count',
    'Number of online robots',
    ['site_uid']
)

# 使用metrics
task_created_total.labels(site_uid='504', task_type='DELIVERY').inc()

with task_duration_seconds.labels(site_uid='504', task_type='DELIVERY', status='COMPLETED').time():
    await execute_task()
```

---

## 總結

### 優先級矩陣

| 改進項目 | 優先級 | 工作量 | 投資回報 | 建議時間表 |
|---------|-------|--------|---------|-----------|
| 整合分散式追蹤 | 🔴 高 | 2-3天 | ⭐⭐⭐⭐⭐ | 1週內完成 |
| 修改Grafana密碼 | 🔴 高 | 0.5天 | ⭐⭐⭐⭐⭐ | 立即執行 |
| 增加日誌保留期 | 🟡 中 | 0.5天 | ⭐⭐⭐ | 2週內完成 |
| 部署Prometheus Server | 🟡 中 | 1天 | ⭐⭐⭐⭐ | 2週內完成 |
| 配置死信佇列 | 🟡 中 | 1-2週 | ⭐⭐⭐⭐ | 1個月內完成 |
| 實作告警系統 | 🟡 中 | 1週 | ⭐⭐⭐⭐ | 1個月內完成 |
| 遷移OpenTelemetry | 🟡 中 | 3-6個月 | ⭐⭐⭐⭐⭐ | Q2-Q3執行 |
| SLO/SLI監控 | 🟢 低 | 1-2個月 | ⭐⭐⭐⭐ | Q3執行 |

### 快速勝利（Quick Wins）

1. **修改Grafana密碼**（0.5天，立即可做）
2. **增加日誌保留期**（0.5天，立即可做）
3. **部署Zipkin**（1天，高價值）
4. **添加關鍵告警規則**（2天，高價值）

### 長期路線圖

**Q1**: 基礎監控完善
- 分散式追蹤
- Prometheus Server
- 告警系統

**Q2**: 可靠性提升
- 死信佇列
- 重試機制優化
- 性能優化

**Q3-Q4**: 可觀測性升級
- OpenTelemetry遷移
- SLO/SLI監控
- 完整的可觀測性平台

---

## 相關文檔

- [系統架構總覽](./01-system-architecture-overview.md)
- [任務閉環流程技術文檔](./02-task-loop-technical-doc.md)
- [API參考文檔](./03-api-reference.md)
- [運維和故障排查手冊](./04-operations-troubleshooting.md)

---

**文檔維護**: 本文檔應根據系統演進和實際實施情況持續更新。建議每季度審查一次優先級和進度。
