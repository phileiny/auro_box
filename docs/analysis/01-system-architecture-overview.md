# 一站式系統架構總覽

**文檔版本**: v1.0
**最後更新**: 2025-12-10
**作者**: 基於系統探索成果整理
**狀態**: ✅ 已完成

---

## 目錄

1. [系統簡介](#系統簡介)
2. [整體架構](#整體架構)
3. [核心服務層](#核心服務層)
4. [通信架構](#通信架構)
5. [數據流向](#數據流向)
6. [技術棧](#技術棧)
7. [部署架構](#部署架構)

---

## 系統簡介

### 系統定位

一站式智慧配送系統是一個基於 Kubernetes 的雲原生微服務架構，用於管理和調度智慧配送機器人（Robot）和配送站點（Station）的任務執行。

### 核心功能

- ✅ **任務管理**：創建、分配、追蹤、完成配送任務
- ✅ **設備管理**：管理Robot、Station、Locker等IoT設備
- ✅ **實時調度**：基於位置和負載的智慧任務分配
- ✅ **狀態同步**：多層架構實現設備狀態實時同步
- ✅ **事件驅動**：DAPR事件總線解耦服務間通信

### 系統規模

- **Robot數量**：100+ 台
- **Station數量**：50+ 個
- **日任務量**：10,000+ 筆
- **微服務數量**：50+ 個
- **Kubernetes節點**：10+ 個

---

## 整體架構

### 系統架構圖

```
┌────────────────────────────────────────────────────────────────┐
│                        客戶端層                                  │
│  Web後台 / Mobile App / 第三方系統                               │
└──────────────────────┬─────────────────────────────────────────┘
                       │ HTTPS / WebSocket
┌──────────────────────▼─────────────────────────────────────────┐
│                    接入層（Traefik）                              │
│  - JWT驗證                                                       │
│  - 路由轉發（gRPC/HTTP）                                         │
│  - SSL終止                                                       │
└──────────────────────┬─────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼─────┐ ┌──────▼──────┐ ┌────▼──────────┐
│ Application │ │     SPF     │ │ Infrastructure│
│  Namespace  │ │  Namespace  │ │   Namespace   │
└─────────────┘ └─────────────┘ └───────────────┘
```

### 三層命名空間架構

#### 1. Application Namespace（應用層）
**職責**：業務邏輯和任務管理

**核心服務**：
- `app-delivery-rpc`：任務流程主控服務
- `app-delivery-actor-robot`：Robot任務Actor（處理 RobotActorV2）
- `app-delivery-actor-station`：Station/Vendor任務Actor（同時處理 StationActorV2 和 VendorActorV2）
- `app-delivery-pubsub`：任務事件訂閱分發
- `app-shop-rpc`：訂單和商城服務
- `notification-actor/pubsub`：客戶端通知服務

**技術特徵**：
- DAPR Actor模式
- DAPR Pubsub事件驅動
- TimescaleDB持久化
- Redis狀態快取

#### 2. SPF Namespace（設備層）
**職責**：IoT設備管理和通信

**核心服務**：
- `jarvis-iot-v6`：IoT設備通信核心
- `jarvis-cmdb`：配置管理數據庫
- `jarvis-site`：站點管理
- `jarvis-alarm`：告警服務
- `jarvis-cmdb-consumer`：Kafka事件消費
- `jarvis-cmdb-cronjob-*`：定時任務和重試

**技術特徵**：
- gRPC通信（LoginUnit、SyncMeteorEvents）
- Kafka事件流
- Cronjob定時任務
- OTA升級管理

#### 3. Infrastructure Namespace（基礎設施層）
**職責**：基礎設施服務

**核心組件**：
- `Kafka`：事件總線
- `Grafana`：監控視覺化
- `EMQX`：MQTT Broker
- `Minio`：對象存儲
- `CoreDNS`：DNS服務
- `Argo Rollouts`：漸進式發布

---

## 核心服務層

### 任務管理層

```
┌─────────────────────────────────────────────────┐
│            任務生命週期管理                       │
├─────────────────────────────────────────────────┤
│                                                 │
│  app-shop-rpc (訂單創建)                        │
│       ↓                                         │
│  app-delivery-rpc (任務主控)                    │
│       ↓                                         │
│  app-delivery-actor-* (Actor狀態機)             │
│       ↓                                         │
│  app-delivery-pubsub (事件分發)                 │
│                                                 │
└─────────────────────────────────────────────────┘
```

#### app-delivery-rpc（任務主控）

**服務資訊**：
- Namespace: `application`
- Service: `app-delivery-rpc-svc:5000`
- Protocol: `h2c` (HTTP/2 Cleartext)
- DAPR: Enabled (app-id: app-delivery-rpc)

**核心功能**：
1. **任務CRUD**：創建、查詢、更新、刪除任務
2. **任務分配**：根據算法分配任務給Robot/Station
3. **狀態管理**：維護任務狀態機
4. **數據庫操作**：讀寫 `t_app_task_ledger2`

**gRPC服務**（推導）：
```protobuf
service AppDeliveryV2 {
  rpc AssignTask(AssignTaskRequest) returns (AssignTaskResponse);
  rpc GetDispatchTaskBoard(GetTaskBoardRequest) returns (TaskBoardResponse);
  rpc QueryLinkOrders(QueryLinkOrdersRequest) returns (QueryLinkOrdersResponse);
  rpc SyncStorage(SyncStorageRequest) returns (SyncStorageResponse);
}
```

#### app-delivery-actor-robot（Robot Actor）

**Actor模式實現**：
```python
class RobotActorV2:
    actor_id: str = "s{site_uid}.u{unit_uid}"  # 例: s504.u1010020021

    def ping(self):
        """每秒心跳，更新Robot狀態到Redis"""
        - 更新UnitConsensus到Redis
        - 處理dispatch_tasks
        - 發布TaskUpdateSignal事件

    def assign_task(self, task_id):
        """分配任務給Robot"""

    def report_progress(self, task_id, progress):
        """Robot回報任務進度"""
```

#### app-delivery-actor-station（Station/Vendor Actor）

**服務特點**：此服務同時處理兩種 Actor 類型：

| Actor 類型 | unit_uid 範例 | 用途 |
|-----------|--------------|------|
| `StationActorV2` | s504.u1020025026 | 配送站點，與 Robot 進行 2.4G 本地協作 |
| `VendorActorV2` | s504.u1330026027 | 取貨點/供應商，處理取件任務 |

**Actor模式實現**：
```python
class StationActorV2:
    actor_id: str = "s{site_uid}.u{unit_uid}"  # 例: s504.u1020025026

    def ping(self):
        """心跳，更新Station狀態到Redis"""

class VendorActorV2:
    actor_id: str = "s{site_uid}.u{unit_uid}"  # 例: s504.u1330026027

    def ping(self):
        """心跳，更新Vendor狀態到Redis"""
```

**生命週期管理**（從 log 觀察）：
- `UnitConsensus UPDATED` → Actor 開始運行
- `ping OK` / `ping SKIPPED` → 心跳狀態
- `no ping in 60s` → 設備離線警告
- `loop BREAK` → `deactivate` → Actor 停止

**狀態追蹤**：
- Redis Key: `UnitConsensus@s{site}.u{unit_uid}`
- 更新頻率: 每秒
- 狀態內容: space、purpose、dispatch_tasks、task_uuid、battery等

**事件發布**：
```python
# DAPR Publish
event_type = "T_TaskUpdateSignal"
event_id = 11961  # 範例
payload = {
    "task_id": 67985120,
    "unit_uid": 1010020021,
    "status": "COMPLETED",
    "timestamp": "2025-10-28T15:43:52+08:00"
}
```

### IoT設備層

#### jarvis-iot-v6（IoT核心）

**服務資訊**：
- Namespace: `spf`
- Service: `jarvis-iot-v6-svc:5000`
- Host: `robot-rpc.aurotek.com`
- Path Prefix: `/jarvis_iot.JarvisIot`

**核心功能**：
1. **設備登錄**：LoginUnit、RSLoginUnit
2. **事件同步**：SyncMeteorEvents（高頻）
3. **心跳處理**：Heartbeat（每分鐘）
4. **配置下發**：GetDispatchUnitConfig
5. **事件流處理**：Broker.Stream、process_unit_event

**gRPC服務**：
```protobuf
service JarvisIot {
  rpc LoginUnit(LoginUnitRequest) returns (LoginUnitResponse);
  rpc RSLoginUnit(RSLoginUnitRequest) returns (RSLoginUnitResponse);
  rpc SyncMeteorEvents(SyncMeteorEventsRequest) returns (SyncMeteorEventsResponse);
  rpc QueryLinkOrders(QueryLinkOrdersRequest) returns (QueryLinkOrdersResponse);
  rpc GetDispatchUnitConfig(GetConfigRequest) returns (ConfigResponse);
  rpc CreateResource(CreateResourceRequest) returns (CreateResourceResponse);
}
```

**事件流機制**：
```
Robot/Station → SyncMeteorEvents (gRPC) → jarvis-iot-v6
                                            ↓
                                    Broker.Stream.start
                                            ↓
                                    process_unit_event
                                            ↓
                          DAPR EventBus / Kafka Topic
                                            ↓
                          app-delivery-actor-* / jarvis-cmdb-consumer
```

---

## 通信架構

### 雙層通信模式 ⭐⭐⭐

這是系統最重要的架構特徵之一。

#### 層1：雲端調度層（4G gRPC）

**用途**：
- ✅ 任務創建、分配、狀態同步
- ✅ 設備登錄和配置下發
- ✅ 事件流處理
- ✅ 實時監控和告警

**協議**：
- gRPC over 4G LTE
- HTTP/2 (h2c)
- WebSocket (客戶端通知)

**通信路徑**：
```
雲端 (Kubernetes Services)
  ↕ 4G gRPC
Robot / Station (本地設備)
```

**頻率**：
- 心跳：每分鐘 (QueryLinkOrders)
- 事件同步：高頻 (SyncMeteorEvents，每1-5秒)
- 狀態更新：每秒 (Actor ping → Redis)

#### 層2：本地協作層（2.4G無線）⭐

**用途**（經物理實驗驗證）：
- ✅ Station-Robot身份驗證和握手
- ✅ 艙門控制和開鎖指令
- ✅ 實時協作信號
- ✅ 本地狀態同步

**重要性**：
- 🔴 **關鍵組件**：2.4G斷線會導致Robot無法進入Station
- 🔴 **不可替代**：無法僅靠4G實現本地協作
- 🔴 **實時性要求**：艙門控制需要低延遲

**實驗證據**（2025-10-28 16:32-16:36）：
```
拔除2.4G電源線後：
- Robot到達Station艙門但無法進入
- Station出現process_unit_event錯誤
- 分配給Station的任務68009596無法轉派
- 系統自動切換策略，後續任務直接分配給Robot
```

**架構圖**：
```
        ┌─────────────┐
        │  雲端平台   │
        └──────┬──────┘
               │ 4G gRPC
               │ (任務調度)
        ┌──────┼──────┐
        │      │      │
    ┌───▼──┐      ┌──▼───┐
    │Station│      │Robot │
    └───┬──┘      └──┬───┘
        │  2.4G無線   │
        └────────────┘
         (本地協作)
```

### 事件驅動架構（DAPR）

#### DAPR Pubsub模式

**組件**：
```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: application
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
```

**事件流**：
```
發布者 (app-delivery-actor-robot)
  ↓ DAPR Publish
DAPR Sidecar :3500
  ↓ Redis Pubsub
Redis :6379
  ↓ DAPR Subscribe
DAPR Sidecar (多個訂閱者)
  ↓
訂閱者 (app-delivery-pubsub, notification-pubsub, ...)
```

**核心事件類型**：
- `T_TaskUpdateSignal`：任務狀態更新
- `unit_ping`：設備心跳
- `task_update_subscribe`：任務訂閱

#### Kafka事件總線

**Kafka集群**：
- 地址: `kafka.infrastructure.svc:9092`
- Namespace: `infrastructure`

**核心Topics**：

| Topic | 用途 | 生產者 | 消費者 |
|-------|-----|--------|--------|
| `prod-robot-events-v3` | Robot事件流 | jarvis-iot-deploy | - |
| `prod-station-events-v3` | Station事件流 | jarvis-iot-deploy | - |
| `prod-iot-v6-heartbeats` | IoT心跳數據 | jarvis-iot-deploy | - |
| `prod-salt-unit-deploy` | Robot部署事件 | salt-remote-call | jarvis-cmdb-salt-scheduler |
| `prod-simcard-events` | SIM卡狀態事件 | jarvis-cmdb | jarvis-cmdb-consumer |

---

## 數據流向

### 任務下發流程

```mermaid
sequenceDiagram
    participant User as 用戶/系統
    participant Shop as app-shop-rpc
    participant Delivery as app-delivery-rpc
    participant Actor as app-delivery-actor-robot
    participant DAPR as DAPR EventBus
    participant IoT as jarvis-iot-v6
    participant Robot as Robot (4G)

    User->>Shop: POST /order2/new
    Shop->>Delivery: 創建配送任務
    Delivery->>Delivery: INSERT t_app_task_ledger2<br/>status=CREATED
    Delivery->>Actor: 分配任務
    Actor->>DAPR: Publish TaskUpdateSignal<br/>status=ASSIGNED
    DAPR->>IoT: 任務事件通知
    IoT->>Robot: 任務推送 (4G gRPC)
    Robot->>Robot: 開始執行
```

### 任務回報流程

```mermaid
sequenceDiagram
    participant Robot as Robot (4G)
    participant IoT as jarvis-iot-v6
    participant Actor as app-delivery-actor-robot
    participant DAPR as DAPR EventBus
    participant Delivery as app-delivery-rpc
    participant DB as TimescaleDB
    participant Redis as Redis
    participant Notif as notification-pubsub
    participant Client as 客戶端

    loop 執行中高頻同步
        Robot->>IoT: SyncMeteorEvents (每1-5秒)
        IoT->>Actor: process_unit_event
        Actor->>Redis: UPDATE UnitConsensus (每秒)
    end

    Note over Robot: 任務完成
    Robot->>IoT: SyncMeteorEvents<br/>status=COMPLETED
    IoT->>Actor: 任務完成事件
    Actor->>DAPR: Publish TaskUpdateSignal<br/>status=COMPLETED

    par 並行處理
        DAPR->>Delivery: 任務完成事件
        Delivery->>DB: UPDATE t_app_task_ledger2<br/>SET status=COMPLETED
    and
        DAPR->>Notif: 通知訂閱者
        Notif->>Client: WebSocket推送
    end
```

### 數據存儲層次

```
┌────────────────────────────────────────┐
│  TimescaleDB (持久化層)                │
│  - t_app_task_ledger2 (任務主表)       │
│  - 時序優化，自動分區                   │
│  - 適合歷史查詢和統計                   │
└────────────────┬───────────────────────┘
                 │ 寫入觸發
┌────────────────▼───────────────────────┐
│  Redis (快取層)                        │
│  - UnitConsensus@s{site}.u{unit} (設備狀態) │
│  - task:{task_id} (任務快取)           │
│  - 每秒更新，實時查詢                   │
└────────────────┬───────────────────────┘
                 │ 變更事件
┌────────────────▼───────────────────────┐
│  DAPR EventBus (事件層)                │
│  - TaskUpdateSignal                    │
│  - 解耦發布者和訂閱者                   │
└────────────────────────────────────────┘
```

---

## 技術棧

### 後端技術

| 技術 | 版本 | 用途 |
|------|------|------|
| **Kubernetes** | v1.21+ | 容器編排 |
| **DAPR** | v1.9+ | 微服務運行時 |
| **Python** | 3.9+ | 主要開發語言 |
| **gRPC** | - | 服務間通信 |
| **PostgreSQL** | 14 | 主數據庫 |
| **TimescaleDB** | 2.8+ | 時序數據庫擴展 |
| **Redis** | 6.2+ | 快取和Pubsub |
| **Kafka** | 2.8+ | 事件總線 |
| **Traefik** | v2.5+ | API網關和路由 |

### 監控技術

| 技術 | 版本 | 用途 |
|------|------|------|
| **Prometheus** | - | Metrics收集 |
| **Grafana** | 8.4.5 | 監控視覺化 |
| **阿里雲Log Service** | - | 日誌聚合 |
| **DAPR Metrics** | - | Sidecar指標 |

### 其他組件

| 技術 | 版本 | 用途 |
|------|------|------|
| **EMQX** | - | MQTT Broker（硬體模組） |
| **Minio** | - | 對象存儲 |
| **Argo Rollouts** | - | 漸進式發布 |
| **Salt** | - | Robot OTA升級 |

---

## 部署架構

### Kubernetes集群拓撲

```
┌────────────────────────────────────────────────┐
│              SLB (負載均衡)                     │
│  robot-rpc.aurotek.com                        │
│  robot-api.aurotek.com                        │
└──────────────────┬─────────────────────────────┘
                   │
┌──────────────────▼─────────────────────────────┐
│           Traefik IngressRoute                 │
│  - gRPC路由 (app-grpc, javis-grpc)            │
│  - HTTP路由 (app-web, javis-web)              │
│  - JWT中間件                                   │
└──────────────────┬─────────────────────────────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
┌───────▼───┐ ┌────▼─────┐ ┌─▼────────┐
│Application│ │   SPF    │ │Infra     │
│ Namespace │ │Namespace │ │Namespace │
│           │ │          │ │          │
│ 30+ Pods  │ │ 25+ Pods │ │ 10+ Pods │
└───────────┘ └──────────┘ └──────────┘
```

### StatefulSet組件

**TimescaleDB**：
```yaml
Name: tsdb-timescaledb-0
Namespace: infrastructure
Port: 30432 (NodePort)
Storage: PVC (持久化存儲)
Database: app_aiyo
```

**Kafka**：
```yaml
Name: kafka-0, kafka-1, kafka-2 (推測)
Namespace: infrastructure
Port: 9092
Service: kafka.infrastructure.svc
```

### DAPR Sidecar注入

**配置**：
```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "app-delivery-rpc"
  dapr.io/app-port: "5000"
  dapr.io/app-protocol: "grpc"
  dapr.io/log-level: "info"
  prometheus.io/port: "9090"
  prometheus.io/path: "/metrics"
  prometheus.io/scrape: "true"
```

**Sidecar容器**：
- 名稱: `daprd`
- 端口: 3500 (HTTP), 50001 (gRPC), 9090 (Metrics)
- 資源: 根據服務配置

---

## 系統特性

### 高可用性設計

1. **多副本部署**：
   - 關鍵服務 ≥ 2 副本
   - Kafka、TimescaleDB使用StatefulSet

2. **健康檢查**：
   - livenessProbe：存活探針
   - readinessProbe：就緒探針

3. **漸進式發布**：
   - Argo Rollouts實現金絲雀發布
   - 自動回滾機制

### 可擴展性

1. **水平擴展**：
   - 無狀態服務可任意擴展
   - HPA（Horizontal Pod Autoscaler）自動擴縮容

2. **Actor模式**：
   - 每個Robot/Station獨立Actor實例
   - 避免資源競爭

3. **事件驅動**：
   - 解耦服務依賴
   - 支持異步處理

### 性能優化

1. **多層快取**：
   - Redis快取熱點數據
   - Actor內存狀態

2. **批量操作**：
   - SyncMeteorEvents批量上報事件
   - Redis Pipeline批量更新

3. **時序優化**：
   - TimescaleDB自動分區
   - 時間範圍查詢優化

---

## 關鍵指標

### SLA目標（推測）

- **任務成功率**: ≥ 99.5%
- **任務平均時長**: < 10分鐘
- **API響應時間**: p95 < 500ms
- **設備在線率**: ≥ 95%
- **事件延遲**: < 5秒

### 容量規劃

- **並發任務**: 500+
- **TPS**: 1000+ (預估)
- **數據增長**: ~100GB/月 (TimescaleDB)
- **Kafka保留期**: 7天
- **日誌保留期**: 1-14天

---

## 總結

### 架構亮點

1. ⭐⭐⭐⭐⭐ **雙層通信架構**：4G雲端調度 + 2.4G本地協作
2. ⭐⭐⭐⭐⭐ **事件驅動設計**：DAPR解耦服務，易於擴展
3. ⭐⭐⭐⭐ **Actor模式**：設備狀態管理清晰
4. ⭐⭐⭐⭐ **多層數據同步**：TimescaleDB + Redis + DAPR
5. ⭐⭐⭐ **時序數據庫**：TimescaleDB優化任務查詢

### 改進空間

1. ⚠️⚠️⚠️ **缺少分散式追蹤**：難以追蹤跨服務調用
2. ⚠️⚠️ **日誌保留期短**：1天保留期不利於問題回溯
3. ⚠️⚠️ **死信佇列未配置**：失敗任務處理依賴Cronjob
4. ⚠️ **監控覆蓋不全**：缺少完整的告警規則

---

## 相關文檔

- [任務閉環流程技術文檔](./02-task-loop-technical-doc.md)
- [API參考文檔](./03-api-reference.md)
- [運維和故障排查手冊](./04-operations-troubleshooting.md)
- [最佳實踐和改進建議](./05-best-practices-improvements.md)

---

**文檔維護**: 本文檔基於2025-10-28至2025-10-30期間的系統探索成果編寫，應隨系統演進持續更新。
