# 模擬器對自架雲平台的價值分析

**版本**: v1.0
**日期**: 2025/12/30
**目的**: 分析現有模擬器設計對自架雲平台專案的幫助與複用可能性

---

## 目錄

1. [背景說明](#1-背景說明)
2. [角色對比分析](#2-角色對比分析)
3. [可複用性評估](#3-可複用性評估)
4. [具體複用建議](#4-具體複用建議)
5. [模擬器作為測試工具](#5-模擬器作為測試工具)
6. [結論](#6-結論)

---

## 1. 背景說明

### 1.1 現有模擬器設計

根據 `docs/analysis/simulate/` 資料夾的文檔，已設計了：

| 組件 | 說明 | 狀態 |
|------|------|------|
| **Station 模擬器** | 模擬 Station 設備與原廠雲平台通訊 | 規劃完成 |
| **Robot 模擬器** | 模擬 Robot 設備與原廠雲平台通訊 | 規劃完成 |
| **GrpcClient** | 與原廠雲平台的 gRPC 通訊 | 骨架完成 |
| **TaskStateMachine** | 任務狀態機 | 骨架完成 |
| **LocalMessageBus** | 模擬 2.4G 本地通訊 | 骨架完成 |
| **NearfieldBus** | 增強版 2.4G 模擬 (含 air_says) | 設計完成 |

### 1.2 自架雲平台目標

- **場域**: 智慧建築
- **設備**: 自研智取櫃 + 普渡機器人 (多品牌擴展)
- **特點**: 普渡機器人僅提供 API，無 2.4G 本地通訊

---

## 2. 角色對比分析

### 2.1 核心差異：角色反轉

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    模擬器 vs 自架雲平台 角色對比                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   模擬器設計 (Client 角色)          自架雲平台 (Server 角色)             │
│   ────────────────────────────────────────────────────────────────     │
│                                                                         │
│   ┌────────────────┐                ┌────────────────┐                 │
│   │ 原廠雲平台     │                │ 自架雲平台     │                 │
│   │ (Server)       │                │ (Server)       │  ← 你要做的     │
│   └───────┬────────┘                └───────┬────────┘                 │
│           │ gRPC                            │ WebSocket / REST          │
│           │                                 │                           │
│   ┌───────▼────────┐                ┌───────▼────────┐                 │
│   │ Station 模擬器 │                │ Station (自研) │                 │
│   │ (Client)       │                │ (Client)       │                 │
│   └───────┬────────┘                └───────┬────────┘                 │
│           │ 2.4G 本地通訊                   │ 雲端協調 (無 2.4G)        │
│   ┌───────▼────────┐                ┌───────▼────────┐                 │
│   │ Robot 模擬器   │                │ 普渡機器人     │                 │
│   │ (Client)       │                │ (API 控制)     │                 │
│   └────────────────┘                └────────────────┘                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 關鍵差異點

| 項目 | 模擬器設計 | 自架雲平台 |
|------|-----------|-----------|
| **雲平台角色** | Client (連接原廠) | Server (自己就是雲) |
| **通訊協議** | gRPC (原廠定義) | WebSocket + REST (自定義) |
| **Robot 通訊** | 2.4G 本地 + 4G | 僅雲端 API (普渡) |
| **Station-Robot 協作** | 2.4G 握手 | 雲端協調 |
| **任務來源** | 原廠雲平台下發 | 自己創建和管理 |

---

## 3. 可複用性評估

### 3.1 複用程度總覽

| 模擬器組件 | 自架雲平台用途 | 複用程度 | 說明 |
|-----------|---------------|---------|------|
| **TaskStateMachine** | Task Service 核心 | ⭐⭐⭐⭐⭐ 高 | 狀態機邏輯直接適用 |
| **任務流程知識** | 系統設計參考 | ⭐⭐⭐⭐⭐ 高 | 完整流程理解 |
| **LocalMessageBus** | Redis Pub/Sub 參考 | ⭐⭐⭐ 中 | 設計模式可參考 |
| **2.4G 消息格式** | 未來自研 Station | ⭐⭐⭐ 中 | 如有 2.4G 硬體可用 |
| **GrpcClient** | 需改為 Server | ⭐⭐ 低 | 角色完全反轉 |
| **2.4G 握手邏輯** | 不適用 | ⭐ 極低 | 普渡無 2.4G |

### 3.2 詳細分析

#### ✅ 高複用價值

**1. 任務狀態機 (TaskStateMachine)**

```python
# 模擬器定義的狀態，直接適用於自架雲平台
class TaskState(Enum):
    CREATED = auto()      # 已創建
    ASSIGNED = auto()     # 已分配給 Station
    REASSIGNED = auto()   # Station 已分配給 Robot
    EXECUTING = auto()    # Robot 執行中
    COMPLETED = auto()    # 已完成
    FAILED = auto()       # 失敗

# 狀態轉換規則同樣適用
ALLOWED_TRANSITIONS = {
    TaskState.CREATED: [TaskState.ASSIGNED],
    TaskState.ASSIGNED: [TaskState.REASSIGNED, TaskState.FAILED],
    TaskState.REASSIGNED: [TaskState.EXECUTING, TaskState.FAILED],
    TaskState.EXECUTING: [TaskState.COMPLETED, TaskState.FAILED],
    TaskState.COMPLETED: [],
    TaskState.FAILED: []
}
```

**複用建議**:
- 直接採用狀態定義
- 可擴展為更細粒度狀態 (如 ROBOT_MOVING, ROBOT_ARRIVED, STATION_OPEN)

**2. 任務流程知識**

模擬器文檔完整記錄了原廠的任務流程：

```
任務創建 → 分配到 Station → Station 轉派給 Robot
→ Robot 執行 → 進度回報 → 任務完成
```

這個流程概念直接適用於自架雲平台，只是實現方式不同。

#### ⚠️ 需要改造

**1. GrpcClient → WebSocket Server**

| 模擬器 | 自架雲平台 |
|-------|-----------|
| `GrpcClient.connect()` | `WebSocketServer.accept()` |
| `GrpcClient.register_unit()` | `handle_station_register()` |
| `GrpcClient.login_unit()` | `handle_station_auth()` |
| 發送心跳 | 接收/驗證心跳 |

**2. LocalMessageBus → Redis Pub/Sub**

```
模擬器:                        自架雲平台:
LocalMessageBus               Redis Pub/Sub
├── subscribe()          →    ├── redis.subscribe()
├── publish()            →    ├── redis.publish()
└── 內存隊列             →    └── Redis Channel
```

設計模式相同，只是底層實現從內存改為 Redis。

#### ❌ 不適用

**1. 2.4G 握手邏輯**

- 普渡機器人沒有 2.4G 模組
- Station-Robot 協作改用雲端協調
- 原本的 `NearfieldBus` 不適用

**2. 原廠 gRPC Proto**

- 自架雲平台使用自定義的 REST + WebSocket 協議
- 原廠的 `jarvis_iot.proto` 不需要

---

## 4. 具體複用建議

### 4.1 直接複用

| 來源 | 目標 | 複用方式 |
|------|------|---------|
| `TaskStateMachine` 狀態定義 | Task Service | 直接採用狀態枚舉和轉換規則 |
| 任務流程時序圖 | 系統設計 | 作為設計參考 |
| 心跳機制設計 | Device Service | 參考心跳間隔和超時設計 |

### 4.2 改造後複用

| 來源 | 目標 | 改造方式 |
|------|------|---------|
| `LocalMessageBus` | Redis Pub/Sub 封裝 | 保留 Topic 設計，改用 Redis 實現 |
| `GrpcClient` 結構 | WebSocket Handler | 保留連線管理邏輯，改為 Server 端 |
| 2.4G 消息格式 (`SAirSay`) | 未來自研 Station | 保留格式定義，待有 2.4G 硬體時使用 |

### 4.3 作為知識庫

| 文檔 | 價值 |
|------|------|
| `02-2.4g-simulation-enhancement.md` | 了解 2.4G 協議細節 |
| `08-communication-loop-sequence.md` | 完整通訊時序參考 |
| `09-communication-design-analysis.md` | 業界方案對比 |

---

## 5. 模擬器作為測試工具

### 5.1 測試架構

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    使用模擬器測試自架雲平台                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                     自架雲平台 (待測目標)                        │   │
│   │                                                                  │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │   │
│   │   │ Task Service │  │Device Service│  │Robot Adapter │        │   │
│   │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │   │
│   │          │                 │                  │                 │   │
│   └──────────┼─────────────────┼──────────────────┼─────────────────┘   │
│              │ WebSocket       │ WebSocket        │ REST API            │
│              │                 │                  │                     │
│   ┌──────────▼─────────────────▼──────────────────▼─────────────────┐   │
│   │                       測試工具                                    │   │
│   │                                                                  │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │   │
│   │   │ Station      │  │ Robot        │  │ Pudu API     │        │   │
│   │   │ 模擬器       │  │ 模擬器       │  │ Mock Server  │        │   │
│   │   │ (改造版)     │  │ (改造版)     │  │              │        │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘        │   │
│   │                                                                  │   │
│   └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 測試場景

| 階段 | 測試工具 | 測試目標 |
|------|---------|---------|
| **開發階段** | Station 模擬器 (改造) | 測試 WebSocket 連線、心跳、指令下發 |
| **整合階段** | Robot 模擬器 (改造) | 測試任務分配、狀態同步 |
| **端到端** | 模擬器 + Pudu Mock | 測試完整收件/寄件流程 |

### 5.3 模擬器改造要點

**Station 模擬器改造**:

```python
# 原本 (連接原廠雲平台)
class StationSimulator:
    def __init__(self):
        self.grpc_client = GrpcClient("robot-rpc.aurotek.com:443")

    def start(self):
        self.grpc_client.connect()
        self.grpc_client.register_unit()
        self.grpc_client.login_unit()

# 改造後 (連接自架雲平台)
class StationSimulator:
    def __init__(self):
        self.ws_client = WebSocketClient("ws://localhost:8080/ws/station")

    def start(self):
        self.ws_client.connect()
        self.ws_client.send_register()
        self.ws_client.start_heartbeat()
```

---

## 6. 結論

### 6.1 最大價值

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        模擬器帶來的最大價值                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   1. 系統理解 ⭐⭐⭐⭐⭐                                                  │
│      • 完整的任務流程理解                                               │
│      • 原廠通訊協議分析                                                 │
│      • Station-Robot 協作機制                                           │
│      • 雙層通訊架構 (4G + 2.4G) 設計理念                                │
│                                                                         │
│   2. 狀態機設計 ⭐⭐⭐⭐                                                  │
│      • 任務狀態定義                                                     │
│      • 狀態轉換規則                                                     │
│      • 錯誤處理流程                                                     │
│                                                                         │
│   3. 測試工具潛力 ⭐⭐⭐⭐                                                │
│      • 改造後可測試自架雲平台                                           │
│      • 不需實體硬體就能驗證雲端邏輯                                     │
│      • 可模擬各種異常場景                                               │
│                                                                         │
│   4. 2.4G 知識儲備 ⭐⭐⭐                                                 │
│      • 如果未來自研 Station 硬體，知識直接可用                          │
│      • air_says 消息格式、編碼方式已分析完成                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 總結

| 面向 | 結論 |
|------|------|
| **程式碼直接複用** | 低 - 角色反轉、協議不同 |
| **設計知識複用** | 高 - 狀態機、流程、架構理念 |
| **測試工具價值** | 高 - 改造後可作為測試客戶端 |
| **未來擴展價值** | 中 - 2.4G 知識待自研 Station 時使用 |

### 6.3 建議行動

| 優先級 | 行動 | 說明 |
|-------|------|------|
| **P0** | 採用任務狀態機設計 | 直接複用狀態定義和轉換規則 |
| **P1** | 參考消息總線設計 | 實現 Redis Pub/Sub 時參考 Topic 設計 |
| **P2** | 改造模擬器為測試工具 | 開發階段用於測試自架雲平台 |
| **P3** | 保留 2.4G 文檔 | 未來自研 Station 時參考 |

---

## 附錄：相關文檔索引

| 文檔 | 路徑 | 說明 |
|------|------|------|
| 模擬器總體規劃 | `docs/analysis/simulate/00-implementation-plan.md` | 26 Stage 實作計劃 |
| 基礎設施實作 | `docs/analysis/simulate/01-phase1-infrastructure.md` | gRPC/狀態機/消息總線骨架 |
| 2.4G 增強方案 | `docs/analysis/simulate/02-2.4g-simulation-enhancement.md` | SAirSay 消息格式 |
| 驗證方案 | `docs/analysis/simulate/04-station-robot-verification.md` | 三階段驗證計劃 |
| 可行性評估 | `docs/analysis/simulate/05-feasibility-evaluation.md` | 技術可行性分析 |
| 自架雲平台架構 | `docs/self-hosted-cloud-platform-architecture.md` | 新雲平台設計 |

---

**文件版本**: v1.0
**日期**: 2025/12/30
