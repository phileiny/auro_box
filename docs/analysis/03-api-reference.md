# API參考文檔

**文檔版本**: v1.0
**最後更新**: 2025-10-31
**API基礎URL**: `https://robot-api.aurotek.com`
**gRPC基礎URL**: `robot-rpc.aurotek.com:443`
**狀態**: ✅ 已完成

---

## 目錄

1. [認證機制](#認證機制)
2. [gRPC服務](#grpc服務)
3. [HTTP RESTful API](#http-restful-api)
4. [DAPR事件](#dapr事件)
5. [數據模型](#數據模型)
6. [錯誤碼](#錯誤碼)

---

## 認證機制

### JWT認證

所有HTTP API和gRPC調用都需要JWT Token認證。

**獲取Token**：
```http
POST https://robot-api.aurotek.com/auth/login
Content-Type: application/json

{
  "username": "user@example.com",
  "password": "password"
}
```

**響應**：
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**使用Token**：

HTTP:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

gRPC:
```python
metadata = [('authorization', 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...')]
response = stub.Method(request, metadata=metadata)
```

---

## gRPC服務

### 1. app-delivery-v2.AppDeliveryV2

**服務端點**: `robot-rpc.aurotek.com/app_delivery_v2.AppDeliveryV2`
**後端服務**: `app-delivery-rpc-svc:5000` (application namespace)
**協議**: h2c (HTTP/2 Cleartext)

#### 1.1 AssignTask

分配任務給Robot或Station。

**方法簽名**（推導）：
```protobuf
rpc AssignTask(AssignTaskRequest) returns (AssignTaskResponse);

message AssignTaskRequest {
  int64 task_id = 1;
  int64 unit_uid = 2;          // Robot或Station的UID
  string task_type = 3;         // DELIVERY, PATROL, MAINTENANCE
  google.protobuf.Timestamp deadline = 4;
  map<string, string> metadata = 5;
}

message AssignTaskResponse {
  bool success = 1;
  string message = 2;
  int64 task_id = 3;
  string task_uuid = 4;
}
```

**使用範例**（Python）：
```python
import grpc
from app_delivery import app_delivery_pb2, app_delivery_pb2_grpc

channel = grpc.insecure_channel('robot-rpc.aurotek.com:443')
stub = app_delivery_pb2_grpc.AppDeliveryV2Stub(channel)

request = app_delivery_pb2.AssignTaskRequest(
    task_id=67985120,
    unit_uid=1010020021,
    task_type='DELIVERY',
    metadata={
        'priority': 'HIGH',
        'estimated_duration': '600'
    }
)

metadata = [('authorization', f'Bearer {jwt_token}')]
response = stub.AssignTask(request, metadata=metadata)

print(f"Success: {response.success}")
print(f"Task UUID: {response.task_uuid}")
```

#### 1.2 GetDispatchTaskBoard

查詢待分派的任務看板。

**方法簽名**（推導）：
```protobuf
rpc GetDispatchTaskBoard(GetTaskBoardRequest) returns (TaskBoardResponse);

message GetTaskBoardRequest {
  int64 site_uid = 1;
  int64 unit_uid = 2;           // 可選，過濾特定設備的任務
  repeated string status = 3;   // 可選，過濾狀態
  int32 limit = 4;              // 可選，限制返回數量
}

message TaskBoardResponse {
  repeated TaskInfo tasks = 1;
  int32 total_count = 2;
}

message TaskInfo {
  int64 task_id = 1;
  string task_uuid = 2;
  int64 unit_uid = 3;
  string status = 4;
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp assigned_at = 6;
  string task_type = 7;
  google.protobuf.Any metadata = 8;
}
```

**使用範例**：
```python
request = app_delivery_pb2.GetTaskBoardRequest(
    site_uid=504,
    unit_uid=1020025026,  # Station ID
    status=['ASSIGNED', 'IN_PROGRESS'],
    limit=10
)

response = stub.GetDispatchTaskBoard(request, metadata=metadata)

for task in response.tasks:
    print(f"Task {task.task_id}: {task.status}")
```

#### 1.3 QueryLinkOrders

查詢關聯訂單。

**方法簽名**（推導）：
```protobuf
rpc QueryLinkOrders(QueryLinkOrdersRequest) returns (QueryLinkOrdersResponse);

message QueryLinkOrdersRequest {
  int64 unit_uid = 1;
  repeated int64 task_ids = 2;  // 可選，查詢特定任務
}

message QueryLinkOrdersResponse {
  repeated OrderInfo orders = 1;
}

message OrderInfo {
  int64 order_id = 1;
  int64 task_id = 2;
  string product_name = 3;
  string customer_phone = 4;
  string delivery_address = 5;
  google.protobuf.Timestamp order_time = 6;
}
```

**實際使用頻率**（從log分析）：
- Robot每分鐘調用一次（15:41:59, 15:42:59, 15:43:59, 15:44:59）

#### 1.4 SyncStorage

同步Station存儲狀態。

**方法簽名**（推導）：
```protobuf
rpc SyncStorage(SyncStorageRequest) returns (SyncStorageResponse);

message SyncStorageRequest {
  int64 site_uid = 1;
  int64 unit_uid = 2;          // Station UID
  repeated CellInfo cells = 3;
}

message CellInfo {
  int32 cell_idx = 1;
  int32 cell_size = 2;
  string box_type = 3;         // BOX_TYPE_DELIVERY, BOX_TYPE_RETURN
  int32 box_capacity = 4;
  bool is_occupied = 5;
  int64 current_task_id = 6;   // 可選，當前存放的任務ID
}

message SyncStorageResponse {
  bool success = 1;
  string message = 2;
}
```

**實際log範例**（clue/delivery_rpc_1542.md）：
```python
# Station 1020025026同步存儲狀態
request = SyncStorageRequest(
    site_uid=504,
    unit_uid=1020025026,
    cells=[
        CellInfo(
            cell_idx=4,
            cell_size=4,
            box_type='BOX_TYPE_DELIVERY',
            box_capacity=1,
            is_occupied=True,
            current_task_id=68001323
        )
    ]
)
# 響應時間: ~60ms
```

---

### 2. jarvis_iot.JarvisIot

**服務端點**: `robot-rpc.aurotek.com/jarvis_iot.JarvisIot`
**後端服務**: `jarvis-iot-v6-svc:5000` (spf namespace)
**協議**: h2c (HTTP/2 Cleartext)

#### 2.1 LoginUnit

設備登錄。

**方法簽名**（推導）：
```protobuf
rpc LoginUnit(LoginUnitRequest) returns (LoginUnitResponse);

message LoginUnitRequest {
  int64 unit_uid = 1;
  string unit_type = 2;        // ROBOT, STATION, LOCKER
  string firmware_version = 3;
  string ip_address = 4;
  map<string, string> device_info = 5;
}

message LoginUnitResponse {
  bool success = 1;
  string message = 2;
  string session_token = 3;
  UnitConfig config = 4;       // 下發的配置
}

message UnitConfig {
  int64 heartbeat_interval_ms = 1;  // 心跳間隔（毫秒）
  int64 sync_interval_ms = 2;       // 同步間隔（毫秒）
  map<string, string> parameters = 3;
}
```

**使用範例**：
```python
# Robot登錄
request = jarvis_iot_pb2.LoginUnitRequest(
    unit_uid=1010020021,
    unit_type='ROBOT',
    firmware_version='v2.3.5',
    ip_address='10.0.5.21',
    device_info={
        'model': 'kago5-1393',
        'battery_level': '85',
        'hardware_version': 'v1.2'
    }
)

response = stub.LoginUnit(request)
print(f"Login success: {response.success}")
print(f"Heartbeat interval: {response.config.heartbeat_interval_ms}ms")
```

#### 2.2 RSLoginUnit

Robot專用增強登錄。

**方法簽名**（推導）：
```protobuf
rpc RSLoginUnit(RSLoginUnitRequest) returns (RSLoginUnitResponse);

message RSLoginUnitRequest {
  LoginUnitRequest base_request = 1;
  RobotCapabilities capabilities = 2;
  RobotLocation initial_location = 3;
}

message RobotCapabilities {
  int32 max_payload_kg = 1;
  int32 max_speed_cm_s = 2;
  repeated string supported_tasks = 3;  // DELIVERY, PATROL, CLEANING
  bool has_camera = 4;
  bool has_speaker = 5;
}

message RobotLocation {
  double latitude = 1;
  double longitude = 2;
  int32 poi_sid = 3;           // Point of Interest ID
  int32 floor = 4;
}

message RSLoginUnitResponse {
  LoginUnitResponse base_response = 1;
  repeated int64 pending_task_ids = 2;  // 待執行的任務列表
}
```

#### 2.3 SyncMeteorEvents ⭐重要

高頻同步Robot/Station事件（每1-5秒）。

**方法簽名**（推導）：
```protobuf
rpc SyncMeteorEvents(SyncMeteorEventsRequest) returns (SyncMeteorEventsResponse);

message SyncMeteorEventsRequest {
  int64 unit_uid = 1;
  repeated UnitEvent events = 2;
}

message UnitEvent {
  string event_uuid = 1;
  string event_type = 2;       // 見下方事件類型表
  google.protobuf.Timestamp timestamp = 3;
  google.protobuf.Any payload = 4;
}

message SyncMeteorEventsResponse {
  bool success = 1;
  int32 processed_count = 2;
  repeated string failed_event_uuids = 3;
}
```

**支持的事件類型**：

| event_type | 說明 | payload類型 |
|-----------|------|-----------|
| `LOCATION_UPDATE` | 位置更新 | LocationPayload |
| `BATTERY_UPDATE` | 電量更新 | BatteryPayload |
| `TASK_PROGRESS` | 任務進度 | TaskProgressPayload |
| `TASK_COMPLETED` | 任務完成 | TaskCompletionPayload |
| `TASK_FAILED` | 任務失敗 | TaskFailurePayload |
| `OBSTACLE_DETECTED` | 障礙物檢測 | ObstaclePayload |
| `DOOR_STATE_CHANGE` | 艙門狀態變化 | DoorStatePayload |
| `ERROR_OCCURRED` | 錯誤發生 | ErrorPayload |

**Payload定義**：
```protobuf
message LocationPayload {
  double latitude = 1;
  double longitude = 2;
  int32 poi_sid = 3;
  int32 floor = 4;
  double speed_cm_s = 5;
  double heading_degree = 6;
}

message BatteryPayload {
  int32 level_percent = 1;
  int32 voltage_mv = 2;
  int32 current_ma = 3;
  bool is_charging = 4;
  int32 estimated_minutes_remaining = 5;
}

message TaskProgressPayload {
  int64 task_id = 1;
  int32 progress_percent = 2;
  string current_stage = 3;    // PICKING_UP, IN_TRANSIT, DELIVERING
  double distance_remaining_m = 4;
  int32 eta_seconds = 5;
}

message TaskCompletionPayload {
  int64 task_id = 1;
  string result = 2;           // SUCCESS, PARTIAL_SUCCESS
  google.protobuf.Timestamp completion_time = 3;
  string evidence_photo_url = 4;
  string customer_signature = 5;
}

message TaskFailurePayload {
  int64 task_id = 1;
  string failure_type = 2;     // ROBOT_MALFUNCTION, NAVIGATION_FAILED, CUSTOMER_ABSENT, TIMEOUT
  string error_details = 3;
  google.protobuf.Timestamp failure_time = 4;
}
```

**使用範例**：
```python
# Robot批量上報事件
events = [
    jarvis_iot_pb2.UnitEvent(
        event_uuid='evt-001',
        event_type='LOCATION_UPDATE',
        timestamp=Timestamp(seconds=int(time.time())),
        payload=Any().Pack(LocationPayload(
            latitude=25.0330,
            longitude=121.5654,
            poi_sid=19,
            floor=5,
            speed_cm_s=50.0,
            heading_degree=180.0
        ))
    ),
    jarvis_iot_pb2.UnitEvent(
        event_uuid='evt-002',
        event_type='TASK_PROGRESS',
        timestamp=Timestamp(seconds=int(time.time())),
        payload=Any().Pack(TaskProgressPayload(
            task_id=67985120,
            progress_percent=60,
            current_stage='IN_TRANSIT',
            distance_remaining_m=150.0,
            eta_seconds=120
        ))
    )
]

request = jarvis_iot_pb2.SyncMeteorEventsRequest(
    unit_uid=1010020021,
    events=events
)

response = stub.SyncMeteorEvents(request)
print(f"Processed: {response.processed_count} events")
```

#### 2.4 GetDispatchUnitConfig

獲取設備配置。

**方法簽名**（推導）：
```protobuf
rpc GetDispatchUnitConfig(GetConfigRequest) returns (ConfigResponse);

message GetConfigRequest {
  int64 unit_uid = 1;
  repeated string config_keys = 2;  // 可選，僅獲取特定配置
}

message ConfigResponse {
  map<string, string> config_values = 1;
  google.protobuf.Timestamp last_updated = 2;
}
```

**常見配置鍵**：
- `heartbeat_interval_ms`：心跳間隔（毫秒）
- `sync_interval_ms`：事件同步間隔（毫秒）
- `max_task_count`：最大並發任務數
- `navigation_speed_cm_s`：導航速度（厘米/秒）
- `battery_low_threshold`：低電量閾值（百分比）

#### 2.5 CreateResource

創建資源（任務相關）。

**方法簽名**（推導）：
```protobuf
rpc CreateResource(CreateResourceRequest) returns (CreateResourceResponse);

message CreateResourceRequest {
  string resource_type = 1;    // TASK_EVIDENCE, ROBOT_LOG, ERROR_REPORT
  int64 unit_uid = 2;
  bytes resource_data = 3;
  map<string, string> metadata = 4;
}

message CreateResourceResponse {
  bool success = 1;
  string resource_id = 2;
  string resource_url = 3;     // OSS URL
}
```

**使用範例**：
```python
# Robot上傳任務完成證據照片
with open('delivery_photo.jpg', 'rb') as f:
    photo_data = f.read()

request = jarvis_iot_pb2.CreateResourceRequest(
    resource_type='TASK_EVIDENCE',
    unit_uid=1010020021,
    resource_data=photo_data,
    metadata={
        'task_id': '67985120',
        'photo_type': 'DELIVERY_CONFIRMATION',
        'timestamp': '2025-10-28T15:45:30+08:00'
    }
)

response = stub.CreateResource(request)
print(f"Photo URL: {response.resource_url}")
```

---

### 3. app_shop.AppTask

**服務端點**: `robot-rpc.aurotek.com/app_shop.AppTask`
**後端服務**: `app-task-rpc-svc:5000` (application namespace)

#### 3.1 CreateOrder（推導）

創建訂單並產生任務。

**方法簽名**：
```protobuf
rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);

message CreateOrderRequest {
  int64 customer_id = 1;
  int64 product_id = 2;
  int32 quantity = 3;
  DeliveryAddress address = 4;
  string customer_phone = 5;
  map<string, string> metadata = 6;
}

message DeliveryAddress {
  string building = 1;
  int32 floor = 2;
  string room = 3;
  string detailed_address = 4;
  double latitude = 5;
  double longitude = 6;
}

message CreateOrderResponse {
  bool success = 1;
  int64 order_id = 2;
  int64 task_id = 3;
  string task_uuid = 4;
  google.protobuf.Timestamp estimated_delivery_time = 5;
}
```

---

## HTTP RESTful API

### 1. 訂單管理

#### 1.1 創建訂單

**HTTP端點**（實際）：
```http
POST https://robot-api.aurotek.com/app-shop/customer/order2/new
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
```

**請求Body**：
```json
{
  "product_id": 12345,
  "quantity": 1,
  "delivery_location": {
    "building": "A棟",
    "floor": 5,
    "room": "501",
    "detailed_address": "靠近電梯口"
  },
  "customer_phone": "0912345678",
  "delivery_time_preference": "ASAP",
  "notes": "請按門鈴"
}
```

**響應**：
```json
{
  "success": true,
  "order_id": 1234567,
  "task_id": 67985120,
  "task_uuid": "d25d1020-d158-431c-bfd3-106b4ce7618e",
  "estimated_delivery_time": "2025-10-28T16:00:00+08:00",
  "robot_assigned": "kago5-1393"
}
```

**HTTP狀態碼**：
- `200 OK`：訂單創建成功
- `400 Bad Request`：請求參數錯誤
- `401 Unauthorized`：JWT Token無效或過期
- `500 Internal Server Error`：服務器內部錯誤

#### 1.2 查詢訂單狀態

```http
GET https://robot-api.aurotek.com/app-shop/customer/order/{order_id}
Authorization: Bearer {JWT_TOKEN}
```

**響應**：
```json
{
  "order_id": 1234567,
  "status": "IN_TRANSIT",
  "task_id": 67985120,
  "task_status": "EXECUTING",
  "progress_percent": 60,
  "robot_name": "kago5-1393",
  "robot_location": {
    "building": "B棟",
    "floor": 3,
    "poi_name": "電梯口"
  },
  "estimated_arrival": "2025-10-28T15:55:00+08:00",
  "created_at": "2025-10-28T15:30:00+08:00",
  "updated_at": "2025-10-28T15:45:00+08:00"
}
```

### 2. 任務管理

#### 2.1 查詢任務詳情

```http
GET https://robot-api.aurotek.com/app-delivery/tasks/{task_id}
Authorization: Bearer {JWT_TOKEN}
```

**響應**：
```json
{
  "task_id": 67985120,
  "task_uuid": "d25d1020-d158-431c-bfd3-106b4ce7618e",
  "status": "EXECUTING",
  "task_type": "DELIVERY",
  "assigned_unit": {
    "unit_uid": 1010020021,
    "unit_type": "ROBOT",
    "name": "kago5-1393"
  },
  "pickup_location": {
    "station_id": 1020025026,
    "station_name": "station5-067",
    "poi_id": 5
  },
  "delivery_location": {
    "building": "A棟",
    "floor": 5,
    "room": "501"
  },
  "progress": {
    "current_stage": "IN_TRANSIT",
    "progress_percent": 60,
    "distance_remaining_m": 150,
    "eta_seconds": 120
  },
  "timeline": {
    "created_at": "2025-10-28T15:30:00+08:00",
    "assigned_at": "2025-10-28T15:30:15+08:00",
    "started_at": "2025-10-28T15:31:00+08:00",
    "estimated_completion": "2025-10-28T15:55:00+08:00"
  }
}
```

#### 2.2 查詢任務列表

```http
GET https://robot-api.aurotek.com/app-delivery/tasks?site_uid=504&status=EXECUTING&limit=10
Authorization: Bearer {JWT_TOKEN}
```

**查詢參數**：
- `site_uid`：站點ID（必填）
- `status`：任務狀態（可選，逗號分隔多個狀態）
- `unit_uid`：設備ID（可選，過濾特定Robot/Station）
- `task_type`：任務類型（可選）
- `from_date`：起始日期（可選，ISO 8601格式）
- `to_date`：結束日期（可選）
- `limit`：返回數量限制（可選，默認20，最大100）
- `offset`：偏移量（可選，用於分頁）

**響應**：
```json
{
  "tasks": [
    {
      "task_id": 67985120,
      "task_uuid": "d25d1020-...",
      "status": "EXECUTING",
      "unit_uid": 1010020021,
      "progress_percent": 60,
      "created_at": "2025-10-28T15:30:00+08:00"
    },
    {
      "task_id": 68001323,
      "task_uuid": "3a39b3bb-...",
      "status": "EXECUTING",
      "unit_uid": 1010020021,
      "progress_percent": 40,
      "created_at": "2025-10-28T15:41:35+08:00"
    }
  ],
  "total_count": 25,
  "limit": 10,
  "offset": 0
}
```

### 3. 設備管理

#### 3.1 查詢Robot狀態

```http
GET https://robot-api.aurotek.com/jarvis-iot/units/{unit_uid}
Authorization: Bearer {JWT_TOKEN}
```

**響應**：
```json
{
  "unit_uid": 1010020021,
  "unit_type": "ROBOT",
  "name": "kago5-1393",
  "is_online": true,
  "status": {
    "work_status": "BUSYNESS",
    "space": "PLAIN",
    "purpose": "GO_POINT",
    "mission_type": "GO_POINT"
  },
  "location": {
    "building": "B棟",
    "floor": 3,
    "poi_sid": 19,
    "latitude": 25.0330,
    "longitude": 121.5654
  },
  "battery": {
    "level_percent": 85,
    "is_charging": false,
    "estimated_minutes_remaining": 240
  },
  "current_tasks": [67985120],
  "last_heartbeat": "2025-10-28T15:45:00+08:00",
  "firmware_version": "v2.3.5"
}
```

#### 3.2 查詢Station狀態

```http
GET https://robot-api.aurotek.com/jarvis-iot/stations/{station_uid}
Authorization: Bearer {JWT_TOKEN}
```

**響應**：
```json
{
  "unit_uid": 1020025026,
  "unit_type": "STATION",
  "name": "station5-067",
  "is_online": true,
  "location": {
    "building": "A棟",
    "floor": 5,
    "poi_sid": 5
  },
  "storage": {
    "total_cells": 12,
    "occupied_cells": 3,
    "available_cells": 9,
    "cells": [
      {
        "cell_idx": 4,
        "cell_size": 4,
        "box_type": "BOX_TYPE_DELIVERY",
        "is_occupied": true,
        "current_task_id": 68001323
      }
    ]
  },
  "last_heartbeat": "2025-10-28T15:44:00+08:00"
}
```

### 4. 監控和統計

#### 4.1 任務統計

```http
GET https://robot-api.aurotek.com/app-delivery/stats/tasks?site_uid=504&date=2025-10-28
Authorization: Bearer {JWT_TOKEN}
```

**響應**：
```json
{
  "date": "2025-10-28",
  "site_uid": 504,
  "total_tasks": 1250,
  "completed_tasks": 1180,
  "failed_tasks": 15,
  "in_progress_tasks": 55,
  "completion_rate": 94.4,
  "average_duration_seconds": 580,
  "peak_hour": {
    "hour": 14,
    "task_count": 85
  },
  "by_task_type": {
    "DELIVERY": 1100,
    "PATROL": 100,
    "MAINTENANCE": 50
  }
}
```

#### 4.2 Robot效能統計

```http
GET https://robot-api.aurotek.com/jarvis-iot/stats/robots?site_uid=504&from_date=2025-10-21&to_date=2025-10-28
Authorization: Bearer {JWT_TOKEN}
```

**響應**：
```json
{
  "robots": [
    {
      "unit_uid": 1010020021,
      "name": "kago5-1393",
      "total_tasks": 250,
      "completed_tasks": 242,
      "failed_tasks": 3,
      "completion_rate": 96.8,
      "average_task_duration_seconds": 520,
      "total_distance_km": 45.5,
      "average_battery_usage_percent": 35,
      "online_hours": 156.5
    }
  ],
  "summary": {
    "total_robots": 15,
    "average_completion_rate": 95.2,
    "total_distance_km": 680
  }
}
```

---

## DAPR事件

### Pubsub事件格式

**Topic名稱格式**：
```
s{site_uid}.k{key_id}.t{task_id}.u{unit_uid}.p{pickup_id}
```

**範例**：
```
s504.k67985120.t67985120.u1010020021.p0
│    │          │          │           └─ pickup_id
│    │          │          └───────────── unit_uid (NULL時為"None")
│    │          └──────────────────────── task_id
│    └─────────────────────────────────── key_id (通常等於task_id)
└──────────────────────────────────────── site_uid
```

### TaskUpdateSignal事件

**Event Type**: `T_TaskUpdateSignal`

**Payload結構**：
```json
{
  "event_id": 11961,
  "task_id": 67985120,
  "task_uuid": "d25d1020-d158-431c-bfd3-106b4ce7618e",
  "site_uid": 504,
  "unit_uid": 1010020021,
  "status": "COMPLETED",
  "previous_status": "EXECUTING",
  "action": "STATUS_UPDATE",
  "metadata": {
    "progress_percent": 100,
    "completion_time": "2025-10-28T15:45:30+08:00"
  },
  "timestamp": "2025-10-28T15:45:30+08:00"
}
```

**Action類型**：
- `STATUS_UPDATE`：狀態更新
- `REASSIGN`：任務轉派
- `PROGRESS_UPDATE`：進度更新
- `FAILURE`：任務失敗

### 訂閱TaskUpdateSignal

**Python範例（DAPR SDK）**：
```python
from dapr.ext.grpc import App

app = App()

@app.subscribe(pubsub_name='pubsub', topic='task_update_signal')
def task_update_handler(event_data: dict):
    """處理任務更新事件"""
    task_id = event_data['task_id']
    status = event_data['status']
    action = event_data.get('action', 'STATUS_UPDATE')

    print(f"Task {task_id} {action}: {status}")

    if status == 'COMPLETED':
        # 處理任務完成邏輯
        send_customer_notification(task_id)
    elif status == 'FAILED':
        # 處理任務失敗邏輯
        schedule_retry(task_id)

    return {'success': True}

app.run(50051)  # gRPC端口
```

---

## 數據模型

### Task（任務）

```typescript
interface Task {
  task_id: number;
  task_uuid: string;
  site_uid: number;
  unit_uid: number | null;        // NULL表示未分配
  task_type: TaskType;
  status: TaskStatus;
  pickup_location: Location;
  delivery_location: Location;
  metadata: Record<string, any>;
  created_at: string;             // ISO 8601
  assigned_at: string | null;
  started_at: string | null;
  completed_at: string | null;
  updated_at: string;
}

enum TaskType {
  DELIVERY = 'DELIVERY',
  PATROL = 'PATROL',
  MAINTENANCE = 'MAINTENANCE',
  CLEANING = 'CLEANING'
}

enum TaskStatus {
  CREATED = 'CREATED',
  ASSIGNED = 'ASSIGNED',
  REASSIGNED = 'REASSIGNED',
  EXECUTING = 'EXECUTING',
  IN_PROGRESS = 'IN_PROGRESS',
  COMPLETED = 'COMPLETED',
  FAILED = 'FAILED',
  TIMEOUT = 'TIMEOUT',
  CANCELLED = 'CANCELLED'
}

interface Location {
  station_id?: number;
  poi_id?: number;
  building?: string;
  floor?: number;
  room?: string;
  latitude?: number;
  longitude?: number;
}
```

### Unit（設備）

```typescript
interface Unit {
  unit_uid: number;
  unit_type: UnitType;
  name: string;
  site_uid: number;
  is_online: boolean;
  status: UnitStatus;
  location: Location;
  metadata: Record<string, any>;
  last_heartbeat: string;
  firmware_version: string;
}

enum UnitType {
  ROBOT = 'ROBOT',
  STATION = 'STATION',
  LOCKER = 'LOCKER'
}

interface UnitStatus {
  work_status: WorkStatus;
  space: SpaceState;
  purpose: PurposeState;
  mission_type: MissionType;
}

enum WorkStatus {
  IDLE = 'IDLE',
  BUSYNESS = 'BUSYNESS',
  FIND_WORK = 'FIND_WORK',
  CHARGING = 'CHARGING',
  MAINTENANCE = 'MAINTENANCE',
  ERROR = 'ERROR'
}

enum SpaceState {
  PLAIN = 'PLAIN',
  INSIDE_STATION = 'INSIDE_STATION',
  IN_ELEVATOR = 'IN_ELEVATOR'
}

enum PurposeState {
  WAITING_STATION = 'WAITING_STATION',
  GO_POINT = 'GO_POINT',
  RETURN_HOME = 'RETURN_HOME'
}

enum MissionType {
  STANDBY = 'STANDBY',
  GO_POINT = 'GO_POINT',
  DELIVERY = 'DELIVERY',
  PATROL = 'PATROL'
}
```

### Order（訂單）

```typescript
interface Order {
  order_id: number;
  customer_id: number;
  product_id: number;
  quantity: number;
  task_id: number;
  status: OrderStatus;
  delivery_address: DeliveryAddress;
  customer_phone: string;
  total_amount: number;
  payment_status: PaymentStatus;
  order_time: string;
  estimated_delivery_time: string;
  actual_delivery_time: string | null;
}

enum OrderStatus {
  PENDING = 'PENDING',
  CONFIRMED = 'CONFIRMED',
  IN_PREPARATION = 'IN_PREPARATION',
  IN_TRANSIT = 'IN_TRANSIT',
  DELIVERED = 'DELIVERED',
  CANCELLED = 'CANCELLED'
}

enum PaymentStatus {
  UNPAID = 'UNPAID',
  PAID = 'PAID',
  REFUNDED = 'REFUNDED'
}

interface DeliveryAddress {
  building: string;
  floor: number;
  room: string;
  detailed_address: string;
  latitude: number;
  longitude: number;
}
```

---

## 錯誤碼

### HTTP錯誤碼

| 狀態碼 | 說明 | 範例 |
|-------|------|------|
| `200 OK` | 成功 | 查詢成功 |
| `201 Created` | 創建成功 | 訂單創建成功 |
| `400 Bad Request` | 請求參數錯誤 | 缺少必填字段 |
| `401 Unauthorized` | 未授權 | JWT Token無效 |
| `403 Forbidden` | 無權限 | 無權訪問該資源 |
| `404 Not Found` | 資源不存在 | 任務ID不存在 |
| `409 Conflict` | 衝突 | 任務已分配 |
| `422 Unprocessable Entity` | 業務邏輯錯誤 | Robot電量不足 |
| `500 Internal Server Error` | 服務器錯誤 | 數據庫連接失敗 |
| `503 Service Unavailable` | 服務不可用 | 服務維護中 |

### 業務錯誤碼

**錯誤響應格式**：
```json
{
  "success": false,
  "error_code": "ROBOT_NOT_AVAILABLE",
  "error_message": "No robots available for assignment",
  "details": {
    "site_uid": 504,
    "requested_task_type": "DELIVERY",
    "available_robots_count": 0
  },
  "timestamp": "2025-10-28T15:45:30+08:00"
}
```

**常見業務錯誤碼**：

| 錯誤碼 | HTTP狀態碼 | 說明 |
|-------|-----------|------|
| `TASK_NOT_FOUND` | 404 | 任務不存在 |
| `ROBOT_NOT_AVAILABLE` | 422 | 無可用Robot |
| `ROBOT_OFFLINE` | 422 | Robot離線 |
| `ROBOT_LOW_BATTERY` | 422 | Robot電量不足 |
| `STATION_FULL` | 422 | Station存儲已滿 |
| `INVALID_LOCATION` | 400 | 無效的位置資訊 |
| `TASK_ALREADY_ASSIGNED` | 409 | 任務已分配 |
| `TASK_ALREADY_COMPLETED` | 409 | 任務已完成 |
| `CONCURRENT_MODIFICATION` | 409 | 並發修改衝突 |
| `AUTH_TOKEN_EXPIRED` | 401 | Token已過期 |
| `PERMISSION_DENIED` | 403 | 權限不足 |
| `RATE_LIMIT_EXCEEDED` | 429 | 請求頻率超限 |

---

## 使用範例

### 完整任務流程（Python）

```python
import grpc
import requests
from datetime import datetime

# 1. 獲取JWT Token
auth_response = requests.post(
    'https://robot-api.aurotek.com/auth/login',
    json={'username': 'user@example.com', 'password': 'password'}
)
jwt_token = auth_response.json()['access_token']

# 2. 創建訂單（HTTP API）
order_response = requests.post(
    'https://robot-api.aurotek.com/app-shop/customer/order2/new',
    headers={'Authorization': f'Bearer {jwt_token}'},
    json={
        'product_id': 12345,
        'quantity': 1,
        'delivery_location': {
            'building': 'A棟',
            'floor': 5,
            'room': '501'
        },
        'customer_phone': '0912345678'
    }
)
task_id = order_response.json()['task_id']
print(f"Task created: {task_id}")

# 3. 查詢任務狀態（HTTP API）
while True:
    status_response = requests.get(
        f'https://robot-api.aurotek.com/app-delivery/tasks/{task_id}',
        headers={'Authorization': f'Bearer {jwt_token}'}
    )
    task_status = status_response.json()

    print(f"Status: {task_status['status']}, Progress: {task_status['progress']['progress_percent']}%")

    if task_status['status'] in ['COMPLETED', 'FAILED']:
        break

    time.sleep(5)  # 5秒查詢一次

# 4. gRPC查詢詳細資訊（可選）
channel = grpc.secure_channel(
    'robot-rpc.aurotek.com:443',
    grpc.ssl_channel_credentials()
)
stub = app_delivery_pb2_grpc.AppDeliveryV2Stub(channel)

metadata = [('authorization', f'Bearer {jwt_token}')]
request = app_delivery_pb2.QueryLinkOrdersRequest(
    unit_uid=task_status['assigned_unit']['unit_uid'],
    task_ids=[task_id]
)
orders = stub.QueryLinkOrders(request, metadata=metadata)
print(f"Order details: {orders}")
```

---

## 相關文檔

- [系統架構總覽](./01-system-architecture-overview.md)
- [任務閉環流程技術文檔](./02-task-loop-technical-doc.md)
- [運維和故障排查手冊](./04-operations-troubleshooting.md)
- [最佳實踐和改進建議](./05-best-practices-improvements.md)

---

**API文檔維護**: 本文檔基於2025-10-28至2025-10-30期間的系統探索成果編寫，部分gRPC定義為推導結果。如需確認實際Proto定義，請訪問內部代碼倉庫。
