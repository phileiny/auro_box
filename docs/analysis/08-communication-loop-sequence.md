# 雲平台-Station-Robot 通信閉環時序說明

**文檔版本**: v1.1
**最後更新**: 2025-12-03
**作者**: 基於系統文檔整理
**狀態**: ✅ 已完成

---

## 目錄

1. [通信閉環總覽](#通信閉環總覽)
2. [完整時序圖](#完整時序圖)
3. [各階段詳細說明](#各階段詳細說明)
4. [雙任務機制說明](#雙任務機制說明)
5. [核心通信協議](#核心通信協議)
6. [關鍵閉環節點](#關鍵閉環節點)
7. [2.4G 通訊識別機制](#24g-通訊識別機制)

---

## 通信閉環總覽

系統採用**雙層通信架構**實現雲平台、Station、Robot 之間的完整通信閉環：

- **4G gRPC（雲端調度層）**: 任務分配、狀態同步、事件上報
- **2.4G 無線（本地協作層）**: Station-Robot 握手、艙門控制、實時協作

```
        ┌─────────────┐
        │  雲端平台    │
        │ (K8s 集群)  │
        └──────┬──────┘
               │ 4G gRPC
               │ (任務調度、狀態同步)
        ┌──────┼──────┐
        │      │      │
    ┌───▼──┐      ┌──▼───┐
    │Station│      │Robot │
    └───┬──┘      └──┬───┘
        │  2.4G 無線  │
        └────────────┘
         (本地協作)
```

---

## 完整時序圖

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            通信閉環完整時序                                        │
└──────────────────────────────────────────────────────────────────────────────────┘

 用戶/系統     雲平台(K8s)                              Station          Robot
    │           │                                         │               │
    │ ══════════╪══════════════ 階段1: 任務創建 ══════════╪═══════════════╪══
    │           │                                         │               │
    ├──────────►│ 1. POST /order2/new                     │               │
    │           │    app-shop-rpc 創建訂單                 │               │
    │           │                                         │               │
    │           │ 2. app-delivery-rpc                     │               │
    │           │    INSERT t_app_task_ledger2            │               │
    │           │    status=CREATED, unit_uid=NULL        │               │
    │           │                                         │               │
    │           │ 3. DAPR Publish: TaskCreated            │               │
    │           │    s504.k68001323.t68001323.uNone.p0    │               │
    │           │                                         │               │
    │ ══════════╪══════════ 階段2: 雙重任務分配 ══════════╪═══════════════╪══
    │           │                                         │               │
    │           │ 4. app-delivery-pubsub                  │               │
    │           │    接收 TaskCreated 事件                 │               │
    │           │    執行分配決策（雙重分配）               │               │
    │           │                                         │               │
    │           │         ┌──────────────────────────────►│               │
    │           │         │ 5a. 配送任務 68001323         │               │
    │           │         │     分配給 Station            │               │
    │           │         │     (等待 Robot 取件)         │               │
    │           │                                         │               │
    │           │         ┌────────────────────────────────────────────── ►│
    │           │         │ 5b. 取件任務 task_id=2        │               │
    │           │         │     分配給 Robot              │               │
    │           │         │     (指示去 Station 取件)      │               │
    │           │                                         │               │
    │           │ 6. Pubsub 更新                          │               │
    │           │    配送任務: s504.k68001323.u1020025026 │               │
    │           │    Robot dispatch_tasks: [task_id=2]   │               │
    │           │                                         │               │
    │ ══════════╪══════════ 階段3: Robot 前往 Station ════╪═══════════════╪══
    │           │                                         │               │
    │           │                                         │    Robot 根據  │
    │           │                                         │    task_id=2  │
    │           │                                         │◄──────────────┤
    │           │                                         │    前往       │
    │           │                                         │    Station    │
    │           │                                         │               │
    │           │                                         │◄──────────────┤
    │           │                                         │  7. 2.4G 無線  │
    │           │                                         │     身份驗證   │
    │           │                                         │               │
    │           │                                         │───────────────►
    │           │                                         │  8. 2.4G 握手  │
    │           │                                         │     開啟艙門   │
    │           │                                         │               │
    │           │                                         │  ┌────────────►
    │           │                                         │  │ Robot 進入  │
    │           │                                         │  │ Station    │
    │           │                                         │               │
    │ ══════════╪════════════ 階段4: 高頻狀態同步 ════════╪═══════════════╪══
    │           │                                         │               │
    │           │◄────────────────────────────────────────│───────────────┤
    │           │ 9. SyncMeteorEvents (4G gRPC)           │               │
    │           │    每 1-5 秒批量上報                     │               │
    │           │    space=INSIDE_STATION                 │               │
    │           │    purpose=WAITING_STATION              │               │
    │           │    dispatch_tasks=[task_id=2, RUNNING]  │               │
    │           │                                         │               │
    │           │ 10. jarvis-iot-v6 → ActorRobot          │               │
    │           │     process_unit_event                  │               │
    │           │                                         │               │
    │           │ 11. Redis 每秒更新                       │               │
    │           │     UnitConsensus@s504.u1010020021      │               │
    │           │                                         │               │
    │ ══════════╪════════════ 階段5: 配送任務轉派 ════════╪═══════════════╪══
    │           │                                         │               │
    │           │ 12. Robot 完成取件                      │               │
    │           │     取件任務 task_id=2 完成             │               │
    │           │                                         │               │
    │           │ 13. ActorRobot 發布                     │               │
    │           │     DAPR Publish TaskUpdateSignal       │               │
    │           │     (配送任務從 Station 轉派給 Robot)    │               │
    │           │                                         │               │
    │           │ 14. Pubsub 更新                         │               │
    │           │     s504.k68001323.u1010020021.p2492959 │               │
    │           │     (Robot 正式接收配送任務)             │               │
    │           │                                         │               │
    │ ══════════╪════════════ 階段6: Robot 配送 ══════════╪═══════════════╪══
    │           │                                         │               │
    │           │                                         │   Robot 離開   │
    │           │                                         │◄──────────────┤
    │           │                                         │  2.4G 斷開     │
    │           │                                         │               │
    │           │◄────────────────────────────────────────────────────────┤
    │           │ 15. SyncMeteorEvents (4G)               │               │
    │           │     space=PLAIN                         │               │
    │           │     purpose=GO_POINT                    │               │
    │           │     destination_poi=19                  │               │
    │           │     task_uuid=d25d1020-... (配送任務)   │               │
    │           │                                         │               │
    │           │◄────────────────────────────────────────────────────────┤
    │           │ 16. 持續進度回報                         │               │
    │           │     progress=20%/40%/60%/80%            │               │
    │           │                                         │               │
    │ ══════════╪════════════ 階段7: 任務完成 ════════════╪═══════════════╪══
    │           │                                         │               │
    │           │◄────────────────────────────────────────────────────────┤
    │           │ 17. SyncMeteorEvents                    │               │
    │           │     status=COMPLETED                    │               │
    │           │                                         │               │
    │           │ 18. DAPR Publish TaskUpdateSignal       │               │
    │           │     status=COMPLETED                    │               │
    │           │                                         │               │
    │           │ 19. 並行處理:                           │               │
    │           │     ├─ TimescaleDB UPDATE               │               │
    │           │     ├─ Redis 清空 dispatch_tasks        │               │
    │           │     └─ WebSocket 推送客戶端              │               │
    │           │                                         │               │
    │◄──────────│ 20. 通知: 任務完成                      │               │
    │           │                                         │               │
    └───────────┴─────────────────────────────────────────┴───────────────┘
```

---

## 各階段詳細說明

### 階段1: 任務創建

| 步驟 | 組件 | 動作 | 說明 |
|-----|------|------|------|
| 1 | app-shop-rpc | 接收訂單請求 | POST /order2/new |
| 2 | app-delivery-rpc | 創建任務記錄 | INSERT t_app_task_ledger2, status=CREATED |
| 3 | DAPR EventBus | 發布事件 | TaskCreated, unit_uid=None |

**Pubsub 格式**: `s504.k68001323.t68001323.uNone.p0`
- `s504`: 站點 ID
- `k68001323`: 訂單 Key
- `t68001323`: 任務 ID
- `uNone`: 未分配
- `p0`: 優先級

### 階段2: 雙重任務分配 ⭐重要

系統採用**雙重任務分配**機制，同時分配兩個任務：

| 任務類型 | Task ID | 分配對象 | 用途 |
|---------|---------|---------|------|
| **取件任務** | 2 | Robot | 指示 Robot 前往 Station |
| **配送任務** | 68001323 | Station | 等待 Robot 取件後轉派 |

| 步驟 | 組件 | 動作 | 說明 |
|-----|------|------|------|
| 4 | app-delivery-pubsub | 接收事件 | 執行雙重分配決策 |
| 5a | jarvis-iot-v6 | 4G gRPC | 配送任務 68001323 → Station |
| 5b | jarvis-iot-v6 | 4G gRPC | 取件任務 task_id=2 → Robot |
| 6 | DAPR EventBus | 更新訂閱 | 兩個任務同時生效 |

**Robot 收到的信息**:
```json
{
  "dispatch_tasks": [{"task_id": 2, "state": "PENDING"}],
  "destination": "Station 1020025026"
}
```

**Station 收到的信息**:
```json
{
  "task_id": 68001323,
  "status": "ASSIGNED",
  "waiting_for_robot": true
}
```

### 階段3: Robot 前往 Station

Robot 根據取件任務 (`task_id=2`) 前往指定的 Station。

| 步驟 | 通信方式 | 動作 | 說明 |
|-----|---------|------|------|
| - | 4G | Robot 導航 | 根據 task_id=2 前往 Station |
| 7 | 2.4G 無線 | 身份驗證 | Robot 發送認證請求 |
| 8 | 2.4G 無線 | 握手成功 | Station 開啟艙門 |

**重要**: 2.4G 是關鍵組件，斷線會導致 Robot 無法進入 Station

### 階段4: 高頻狀態同步

| 步驟 | 組件 | 動作 | 頻率 |
|-----|------|------|------|
| 9 | Robot → jarvis-iot | SyncMeteorEvents | 每 1-5 秒 |
| 10 | jarvis-iot → ActorRobot | process_unit_event | 即時 |
| 11 | ActorRobot → Redis | UnitConsensus 更新 | 每秒 |

**Robot 在 Station 內的狀態**:
```json
{
  "space_state": "INSIDE_STATION",
  "purpose_state": "WAITING_STATION",
  "mission_type": "STANDBY",
  "work_status": "BUSYNESS",
  "dispatch_tasks": [{"task_id": 2, "state": "RUNNING"}],
  "task_uuid": "3a39b3bb-5e0d-4886-8d08-46891942cbb7"
}
```

**注意**: 此時 `dispatch_tasks` 中是取件任務 `task_id=2`，而非配送任務 `68001323`

### 階段5: 配送任務轉派

當 Robot 完成取件後，配送任務從 Station 正式轉派給 Robot。

| 步驟 | 組件 | 動作 | 說明 |
|-----|------|------|------|
| 12 | Robot | 取件完成 | task_id=2 執行完畢 |
| 13 | ActorRobot | DAPR Publish | TaskUpdateSignal 觸發轉派 |
| 14 | app-delivery-pubsub | 更新訂閱 | Robot 正式接收配送任務 |

**Pubsub 變化**:
```
配送任務 68001323:
  Station: s504.k68001323.u1020025026.p2492959
      ↓ 轉派
  Robot:   s504.k68001323.u1010020021.p2492959
```

**Robot 狀態變化**:
```
dispatch_tasks: [task_id=2] → []  (取件任務清空)
task_uuid: 3a39b3bb-... → d25d1020-...  (切換到配送任務)
```

### 階段6: Robot 配送

| 步驟 | 組件 | 動作 | 說明 |
|-----|------|------|------|
| 15 | Robot → 雲平台 | SyncMeteorEvents | space=PLAIN, purpose=GO_POINT |
| 16 | Robot → 雲平台 | 進度回報 | 20%/40%/60%/80% |

**狀態轉換**:
```
space: INSIDE_STATION → PLAIN
purpose: WAITING_STATION → GO_POINT
mission: STANDBY → GO_POINT
destination_poi_sid: 16 → 19
```

### 階段7: 任務完成

| 步驟 | 組件 | 動作 | 說明 |
|-----|------|------|------|
| 17 | Robot → jarvis-iot | 完成事件 | status=COMPLETED |
| 18 | ActorRobot | DAPR Publish | TaskUpdateSignal |
| 19 | 多組件並行 | 數據同步 | DB + Redis + WebSocket |
| 20 | notification-pubsub | 客戶端通知 | WebSocket 推送 |

**並行處理**:
- TimescaleDB: UPDATE t_app_task_ledger2 SET status='COMPLETED'
- Redis: 清空 dispatch_tasks
- WebSocket: 推送完成通知給客戶端

---

## 雙任務機制說明

### 為什麼需要雙重任務分配？

```
┌─────────────────────────────────────────────────────────────────────┐
│                      雙任務協作機制                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   配送任務 (68001323)              取件任務 (task_id=2)              │
│   ┌─────────────────┐             ┌─────────────────┐              │
│   │ 分配給 Station  │             │ 分配給 Robot    │              │
│   │ 等待 Robot 取件 │             │ 指示去 Station  │              │
│   └────────┬────────┘             └────────┬────────┘              │
│            │                               │                        │
│            │         Robot 到達            │                        │
│            │◄──────────────────────────────┤                        │
│            │                               │                        │
│            │         取件完成              │                        │
│            │         task_id=2 結束        │                        │
│            │                               ▼                        │
│            │                        ┌─────────────┐                 │
│            │                        │ 任務完成    │                 │
│            │                        └─────────────┘                 │
│            │                                                        │
│            ▼                                                        │
│   ┌─────────────────┐                                              │
│   │ 轉派給 Robot    │ ← TaskUpdateSignal                           │
│   │ 開始配送        │                                              │
│   └─────────────────┘                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 任務 ID 對照表

| 時間 | 事件 | 配送任務 68001323 | 取件任務 task_id=2 |
|-----|------|------------------|-------------------|
| 15:41:35 | 創建 | CREATED (uNone) | - |
| 15:41:59 | 分配 | ASSIGNED (Station) | ASSIGNED (Robot) |
| 15:43:42 | 執行中 | 等待中 (Station) | RUNNING (Robot) |
| 15:43:52 | 轉派 | REASSIGNED (Robot) | COMPLETED |
| 15:44:43 | 配送 | IN_PROGRESS (Robot) | - |
| 完成 | 結束 | COMPLETED | - |

### Log 證據

**Robot 在 Station 內執行取件任務** (15:43:42):
```json
{
  "dispatch_tasks": [{"task_id": 2, "state": "RUNNING"}],
  "task_uuid": "3a39b3bb-5e0d-4886-8d08-46891942cbb7",
  "space_state": "INSIDE_STATION"
}
```

**Robot 離開 Station 執行配送任務** (15:44:43):
```json
{
  "dispatch_tasks": [],
  "task_uuid": "d25d1020-d158-431c-bfd3-106b4ce7618e",
  "space_state": "PLAIN",
  "purpose_state": "GO_POINT"
}
```

---

## 核心通信協議

| 層級 | 協議 | 用途 | 頻率 |
|-----|------|------|-----|
| **雲端調度** | 4G gRPC | 任務分配、狀態同步、事件上報 | 1-5 秒 |
| **本地協作** | 2.4G 無線 | Station-Robot 握手、艙門控制 | 實時 |
| **事件驅動** | DAPR Pub/Sub | 跨服務事件分發 | 毫秒級 |
| **狀態快取** | Redis | UnitConsensus 更新 | 每秒 |
| **持久化** | TimescaleDB | 任務記錄寫入 | 按事件 |

### gRPC 核心方法

| 方法 | 服務 | 用途 |
|-----|------|------|
| `SyncMeteorEvents` | jarvis-iot-v6 | Robot/Station 批量事件上報 |
| `LoginUnit` | jarvis-iot-v6 | 設備登錄認證 |
| `QueryLinkOrders` | jarvis-iot-v6 | 心跳 + 任務拉取 |
| `GetDispatchUnitConfig` | jarvis-iot-v6 | 配置下發 |

### DAPR 事件類型

| 事件 | 觸發時機 | 訂閱者 |
|-----|---------|-------|
| `TaskCreated` | 任務創建 | app-delivery-pubsub |
| `TaskUpdateSignal` | 任務狀態變更 | pubsub, notification, delivery-rpc |
| `unit_ping` | Actor 心跳 | 監控系統 |

---

## 關鍵閉環節點

### 1. 任務下發閉環（雙重）

```
雲平台 ─┬─ (4G) → Station: 配送任務 68001323
        │
        └─ (4G) → Robot: 取件任務 task_id=2
```

### 2. 狀態回報閉環

```
Robot → SyncMeteorEvents → jarvis-iot → ActorRobot → Redis/TimescaleDB
```

### 3. 本地協作閉環

```
Robot ↔ (2.4G) ↔ Station
  │                  │
  └── 身份驗證 ──────┘
  └── 艙門控制 ──────┘
```

### 4. 任務轉派閉環

```
Robot 取件完成 → TaskUpdateSignal → 配送任務轉派 → Robot 開始配送
```

### 5. 事件通知閉環

```
ActorRobot → DAPR Publish → 多訂閱者並行處理
                              ├── app-delivery-rpc (DB 更新)
                              ├── notification-pubsub (客戶端通知)
                              └── 其他訂閱者
```

---

## 2.4G 通訊識別機制

### Station 和 Robot 如何彼此認得？

2.4G 本地通訊使用 **消息頭 (AirMessageHeader)** 中的 `unit_sid` 來識別對方：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     2.4G 消息結構                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   AirMessageHeader (消息頭)                                          │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ message_type: int    # 消息類型 (34=過門, 35=Station-Robot) │  │
│   │ unit_sid: int        # 設備 ID ← 用這個識別對方              │  │
│   │ version: int         # 消息版本號                            │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   SAirSay (空中消息)                                                 │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ msg_type: AIR_SAY_MESSAGE_TYPE   # 消息類型枚舉              │  │
│   │ say: str                          # Base64 編碼的消息內容    │  │
│   │ created_at: int                   # Unix 時間戳 (毫秒)       │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.4G 消息類型

| 類型 | 枚舉值 | 用途 |
|-----|-------|------|
| `PASS_DOOR_ROBOT_SAY` | 34 | 過門狀態（進入/離開 Station） |
| `STATION_ROBOT_SAY` | 35 | Station-Robot 通訊 |
| `CHARGE_ROBOT_SAY` | 37 | 充電狀態 |
| `PLANNING_ROBOT_SAY` | 38 | 規劃狀態 |

### 2.4G 握手流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                     2.4G 握手時序                                    │
└─────────────────────────────────────────────────────────────────────┘

    Station                                          Robot
       │                                               │
       │  1. Station 廣播 2.4G 信號                     │
       │   (包含 Station 的 unit_sid)                  │
       │──────────────────────────────────────────────►│
       │                                               │
       │  2. Robot 發送認證請求                         │
       │   AirMessageHeader:                           │
       │     message_type: STATION_ROBOT_SAY (35)      │
       │     unit_sid: Robot_ID (如 1010020021)        │
       │◄──────────────────────────────────────────────│
       │                                               │
       │  3. Station 驗證 Robot 身份                    │
       │   - 檢查 unit_sid 是否在授權列表               │
       │   - 可能查詢雲平台確認權限                      │
       │                                               │
       │  4. 驗證通過，發送確認                         │
       │   AirMessageHeader:                           │
       │     message_type: STATION_ROBOT_SAY (35)      │
       │     unit_sid: Station_ID (如 1020025026)      │
       │──────────────────────────────────────────────►│
       │                                               │
       │  5. 開啟艙門，Robot 進入                       │
       │◄─────────────── 2.4G 連線建立 ────────────────►│
       │                                               │
```

### Log 證據

**實際 air_says 數據** (來自 `delivery-actor-robot_1542.md`):

```python
air_says=[
    SAirSay(msg_type=CHARGE_ROBOT_SAY, say='JQogDzwA', created_at=1761637348870),
    SAirSay(msg_type=STATION_ROBOT_SAY, say='IwqAMwEQ8GRg...', created_at=1761637422474),
    SAirSay(msg_type=PASS_DOOR_ROBOT_SAY, say='IgowYwEQ...', created_at=1761629219582),
    SAirSay(msg_type=PLANNING_ROBOT_SAY, say='JgqAagAAAA...', created_at=1761637422450)
]

# 設備狀態中的近場通訊標記
unit_status=UnitStatus(
    unit_enabled=False,
    cellular_status=False,
    nearfield_status=False    # ← 2.4G 近場通訊狀態
)
```

### 2.4G 數據上報機制

2.4G 通訊內容會透過 4G 間接上報到雲平台：

```
┌─────────────────────────────────────────────────────────────────────┐
│                          現場 (Local)                                │
│                                                                      │
│   ┌──────────┐     2.4G 無線     ┌──────────┐                       │
│   │ Station  │◄─────────────────►│  Robot   │                       │
│   │          │   (直接通訊)       │          │                       │
│   └──────────┘                   └────┬─────┘                       │
│                                       │                             │
└───────────────────────────────────────┼─────────────────────────────┘
                                        │ 4G (SyncMeteorEvents)
                                        │ 打包上報 air_says
                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          雲平台 (Cloud)                              │
│                                                                      │
│   app-delivery-actor-robot 接收並處理 air_says (Base64 解碼)         │
└─────────────────────────────────────────────────────────────────────┘
```

| 項目 | 說明 |
|-----|------|
| 2.4G 原始封包 | 不會直接傳到雲平台 |
| air_says 消息內容 | ✅ 會傳到雲平台 (Base64 編碼) |
| nearfield_status | ✅ 會傳到雲平台 (True/False) |
| 上報機制 | Robot 透過 4G SyncMeteorEvents 打包上報 |
| 上報頻率 | 每 1-5 秒 (隨心跳一起) |

---

## 注意事項

### 2.4G 無線的重要性

**實驗證據** (2025-10-28 16:32-16:36):
- 拔除 2.4G 電源後，Robot 無法進入 Station
- Station 出現 process_unit_event 錯誤
- 系統自動切換策略，後續任務直接分配給 Robot

**結論**: 2.4G 是不可替代的關鍵組件，斷線會導致 Station-Robot 本地協作失敗

### 容錯機制

1. **自動策略切換**: 2.4G 異常時，繞過 Station 直接分配給 Robot
2. **Cronjob 重試**: 失敗任務每 30 分鐘自動重試
3. **多層數據同步**: 確保數據一致性

---

## 相關文檔

- [系統架構總覽](./01-system-architecture-overview.md)
- [任務閉環流程技術文檔](./02-task-loop-technical-doc.md)
- [API 參考文檔](./03-api-reference.md)
- [DAPR 與 Actor 模式分析](./07-dapr-actor-analysis.md)

---

**文檔維護**: 本文檔基於系統探索成果整理，應隨系統演進持續更新。
