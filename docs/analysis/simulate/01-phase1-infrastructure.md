# Phase 1: 基礎設施實作指南

## 📋 概述

本文檔詳細說明 Phase 1 的實作步驟，包括 proto 編譯、gRPC 客戶端實作和認證機制。

## 🎯 Phase 1 目標

建立基礎的 gRPC 通信和認證機制，為後續的 Station 和 Robot 模擬器奠定基礎。

---

## 1. Proto 文件處理

### 1.1 當前狀態

**已有文件:**
- `proto_from_server/jarvis_iot.proto` (86KB) - 從 Pod 提取的完整 proto
- `proto_from_server/jarvis_iot.protoset` - 預編譯的 protoset 文件

**依賴問題:**
```protobuf
import "google/api/annotations.proto";
import "protoc-gen-swagger/options/annotations.proto";
```

### 1.2 解決方案選項

#### 選項 A: 使用 protoset (推薦)

**優點:**
- 無需解決依賴問題
- protoset 包含所有編譯後的定義
- 可用於 grpcurl 測試

**實作方式:**

1. **使用 Python 動態加載 protoset**

```python
from google.protobuf import descriptor_pool
from google.protobuf import symbol_database
from google.protobuf.descriptor_pb2 import FileDescriptorSet

# 讀取 protoset
with open('proto_from_server/jarvis_iot.protoset', 'rb') as f:
    file_descriptor_set = FileDescriptorSet()
    file_descriptor_set.ParseFromString(f.read())

# 註冊到 descriptor pool
pool = descriptor_pool.DescriptorPool()
for file_descriptor in file_descriptor_set.file:
    pool.Add(file_descriptor)

# 獲取消息描述符
db = symbol_database.Default()
message_descriptor = pool.FindMessageTypeByName('jarvis_iot.RSLoginUnitRequest')
```

2. **或使用 grpcio-reflection**

```python
import grpc
from grpc_reflection.v1alpha import reflection

# 動態創建請求
```

#### 選項 B: 下載依賴並編譯

**步驟:**

```bash
cd proto_from_server

# 創建目錄結構
mkdir -p google/api
mkdir -p protoc-gen-swagger/options

# 下載 Google API proto
curl -o google/api/annotations.proto \
  https://raw.githubusercontent.com/googleapis/googleapis/master/google/api/annotations.proto

curl -o google/api/http.proto \
  https://raw.githubusercontent.com/googleapis/googleapis/master/google/api/http.proto

# 下載 grpc-gateway proto
curl -o protoc-gen-swagger/options/annotations.proto \
  https://raw.githubusercontent.com/grpc-ecosystem/grpc-gateway/master/protoc-gen-swagger/options/annotations.proto

# 編譯
python -m grpc_tools.protoc -I. \
  --python_out=. \
  --grpc_python_out=. \
  jarvis_iot.proto
```

**生成文件:**
- `jarvis_iot_pb2.py` - 消息定義
- `jarvis_iot_pb2_grpc.py` - 服務定義

#### 選項 C: 簡化版 proto (快速原型)

創建簡化版本，移除不需要的 HTTP 註解：

**`simulate/proto/jarvis_iot_simple.proto`:**

```protobuf
syntax = "proto3";

package jarvis_iot;
option go_package = ".;jarvis_iot";

// 只保留必要的消息定義
enum ErrorType {
  ERROR_TYPE_INVALID = 0;
  ERROR_TYPE_UNKWN_ERROR = 1;
  ERROR_TYPE_SYSTEM_ERROR = 2;
  ERROR_TYPE_USER_ERROR = 3;
}

enum ErrorCode {
  ERROR_CODE_OK = 0;
  ERROR_CODE_UNKWN_ERROR_OCCURRED = 1;
  ERROR_CODE_NOT_FOUND = 10;
  ERROR_CODE_INVALID_REQUEST_DATA = 11;
  ERROR_CODE_LOGIN_FAILED = 13;
  ERROR_CODE_REGISTER_FAILED = 14;
}

enum UnitType {
  UNIT_TYPE_UNKNOWN = 0;
  UNIT_TYPE_ROBOT = 1;
  UNIT_TYPE_STATION = 2;
}

message Error {
  ErrorType type = 1;
  ErrorCode code = 2;
  string description = 3;
  uint32 http_status_code = 4;
}

// 註冊請求
message RSRegisterUnitRequest {
  string unit_sn = 2;
  UnitType unit_type = 4;
}

message RSRegisterUnitResponse {
  Error error = 1;
  string unit_uuid = 2;
  int64 unit_uid = 3;
  string unit_hostname = 4;
}

// 登入請求
message RSLoginUnitRequest {
  string unit_uuid = 1;
  string unit_sn = 2;
  UnitType unit_type = 4;
}

message RSLoginUnitResponse {
  Error error = 1;
  string token = 2;
}

// 任務相關 (待完善)
message Task {
  string task_id = 1;
  string task_type = 2;
  int32 status = 3;
}

// 服務定義
service JarvisIot {
  rpc RSRegisterUnit(RSRegisterUnitRequest) returns (RSRegisterUnitResponse);
  rpc RSLoginUnit(RSLoginUnitRequest) returns (RSLoginUnitResponse);
}
```

**編譯簡化版:**

```bash
cd simulate/proto
python -m grpc_tools.protoc -I. \
  --python_out=. \
  --grpc_python_out=. \
  jarvis_iot_simple.proto
```

---

## 2. GrpcClient 基礎類實作

### 2.1 設計目標

統一的 gRPC 客戶端封裝，提供：
- TLS 連接管理
- 註冊和登入
- 錯誤處理
- 重連機制

### 2.2 實作骨架

**`simulate/common/grpc_client.py`:**

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
統一的 gRPC 客戶端封裝
"""

import grpc
import logging
from typing import Optional, Callable
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class UnitInfo:
    """設備信息"""
    unit_sn: str
    unit_type: int  # 1=Robot, 2=Station
    unit_uuid: Optional[str] = None
    unit_uid: Optional[int] = None
    unit_hostname: Optional[str] = None
    token: Optional[str] = None


class GrpcClient:
    """
    gRPC 客戶端基礎類

    功能:
    - 建立 TLS 連接
    - 設備註冊 (RSRegisterUnit)
    - 設備登入 (RSLoginUnit)
    - 錯誤處理和重連
    """

    def __init__(self, server: str, unit_info: UnitInfo):
        """
        Args:
            server: gRPC 服務器地址 (如 robot-rpc.aurotek.com:443)
            unit_info: 設備信息
        """
        self.server = server
        self.unit_info = unit_info
        self.channel: Optional[grpc.Channel] = None
        self.stub = None

    def connect(self) -> bool:
        """
        建立 TLS 連接

        Returns:
            是否成功連接
        """
        try:
            logger.info(f"Connecting to {self.server}...")

            # 創建 TLS 憑證 (不需要客戶端證書)
            credentials = grpc.ssl_channel_credentials()

            # 建立安全通道
            self.channel = grpc.secure_channel(self.server, credentials)

            # 等待連接就緒
            future = grpc.channel_ready_future(self.channel)
            future.result(timeout=10)

            # 創建 stub (需要根據選擇的 proto 方案調整)
            # self.stub = jarvis_iot_pb2_grpc.JarvisIotStub(self.channel)

            logger.info("✅ TLS connection established")
            return True

        except Exception as e:
            logger.error(f"❌ Connection failed: {e}")
            return False

    def register_unit(self) -> bool:
        """
        註冊設備 (RSRegisterUnit)

        Returns:
            是否註冊成功
        """
        try:
            logger.info(f"Registering unit: {self.unit_info.unit_sn}")

            # 構建請求
            # request = jarvis_iot_pb2.RSRegisterUnitRequest(
            #     unit_sn=self.unit_info.unit_sn,
            #     unit_type=self.unit_info.unit_type
            # )

            # 調用 RPC
            # response = self.stub.RSRegisterUnit(request, timeout=10)

            # 檢查錯誤
            # if response.error.code == 0:  # ERROR_CODE_OK
            #     self.unit_info.unit_uuid = response.unit_uuid
            #     self.unit_info.unit_uid = response.unit_uid
            #     self.unit_info.unit_hostname = response.unit_hostname
            #     logger.info(f"✅ Registered: UUID={response.unit_uuid}")
            #     return True
            # else:
            #     logger.error(f"❌ Register failed: {response.error.description}")
            #     return False

            # 暫時返回 False，等待 proto 編譯完成
            return False

        except grpc.RpcError as e:
            logger.error(f"❌ RPC error: {e.code()} - {e.details()}")
            return False

    def login_unit(self) -> bool:
        """
        登入設備 (RSLoginUnit)

        Returns:
            是否登入成功
        """
        try:
            if not self.unit_info.unit_uuid:
                logger.error("❌ unit_uuid not set, please register first")
                return False

            logger.info(f"Logging in unit: {self.unit_info.unit_uuid}")

            # 構建請求
            # request = jarvis_iot_pb2.RSLoginUnitRequest(
            #     unit_uuid=self.unit_info.unit_uuid,
            #     unit_sn=self.unit_info.unit_sn,
            #     unit_type=self.unit_info.unit_type
            # )

            # 調用 RPC
            # response = self.stub.RSLoginUnit(request, timeout=10)

            # 檢查錯誤
            # if response.error.code == 0:
            #     self.unit_info.token = response.token
            #     logger.info(f"✅ Login successful")
            #     return True
            # else:
            #     logger.error(f"❌ Login failed: {response.error.description}")
            #     return False

            # 暫時返回 False，等待 proto 編譯完成
            return False

        except grpc.RpcError as e:
            logger.error(f"❌ RPC error: {e.code()} - {e.details()}")
            return False

    def close(self):
        """關閉連接"""
        if self.channel:
            self.channel.close()
            logger.info("Connection closed")


# 使用示例
if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)

    # Station 示例
    station_info = UnitInfo(
        unit_sn="STZJGF4025030067",
        unit_type=2  # UNIT_TYPE_STATION
    )

    client = GrpcClient("robot-rpc.aurotek.com:443", station_info)

    try:
        # 連接
        if not client.connect():
            exit(1)

        # 註冊
        if not client.register_unit():
            logger.warning("Registration failed, trying login with existing UUID")
            station_info.unit_uuid = "d5ab9c4b-a483-4101-8026-b47237079d6e"

        # 登入
        client.login_unit()

    finally:
        client.close()
```

---

## 3. 任務狀態機

### 3.1 設計

基於文檔中的任務狀態流程：
```
CREATED → ASSIGNED → REASSIGNED → EXECUTING → COMPLETED/FAILED
```

### 3.2 實作骨架

**`simulate/common/state_machine.py`:**

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
任務狀態機
"""

from enum import Enum, auto
from typing import Optional, Callable
from dataclasses import dataclass
from datetime import datetime
import logging

logger = logging.getLogger(__name__)


class TaskState(Enum):
    """任務狀態"""
    CREATED = auto()      # 已創建
    ASSIGNED = auto()     # 已分配給 Station
    REASSIGNED = auto()   # Station 已分配給 Robot
    EXECUTING = auto()    # Robot 執行中
    COMPLETED = auto()    # 已完成
    FAILED = auto()       # 失敗


@dataclass
class Task:
    """任務數據"""
    task_id: str
    task_type: str
    state: TaskState = TaskState.CREATED

    # 分配信息
    station_id: Optional[str] = None
    robot_id: Optional[str] = None

    # 進度
    progress_percentage: int = 0

    # 時間戳
    created_at: datetime = None
    assigned_at: datetime = None
    reassigned_at: datetime = None
    executing_at: datetime = None
    completed_at: datetime = None

    # 結果
    error_message: Optional[str] = None

    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()


class TaskStateMachine:
    """
    任務狀態機

    管理任務狀態轉換和事件通知
    """

    # 允許的狀態轉換
    ALLOWED_TRANSITIONS = {
        TaskState.CREATED: [TaskState.ASSIGNED],
        TaskState.ASSIGNED: [TaskState.REASSIGNED, TaskState.FAILED],
        TaskState.REASSIGNED: [TaskState.EXECUTING, TaskState.FAILED],
        TaskState.EXECUTING: [TaskState.COMPLETED, TaskState.FAILED],
        TaskState.COMPLETED: [],
        TaskState.FAILED: []
    }

    def __init__(self, task: Task):
        self.task = task
        self.callbacks = {}

    def transition(self, new_state: TaskState, **kwargs) -> bool:
        """
        狀態轉換

        Args:
            new_state: 新狀態
            **kwargs: 額外參數 (如 station_id, robot_id, error_message)

        Returns:
            是否轉換成功
        """
        current_state = self.task.state

        # 檢查轉換是否合法
        if new_state not in self.ALLOWED_TRANSITIONS[current_state]:
            logger.error(
                f"Invalid transition: {current_state.name} -> {new_state.name}"
            )
            return False

        # 更新狀態
        old_state = self.task.state
        self.task.state = new_state

        # 更新時間戳和相關信息
        now = datetime.now()

        if new_state == TaskState.ASSIGNED:
            self.task.assigned_at = now
            self.task.station_id = kwargs.get('station_id')

        elif new_state == TaskState.REASSIGNED:
            self.task.reassigned_at = now
            self.task.robot_id = kwargs.get('robot_id')

        elif new_state == TaskState.EXECUTING:
            self.task.executing_at = now

        elif new_state == TaskState.COMPLETED:
            self.task.completed_at = now
            self.task.progress_percentage = 100

        elif new_state == TaskState.FAILED:
            self.task.completed_at = now
            self.task.error_message = kwargs.get('error_message', 'Unknown error')

        logger.info(
            f"Task {self.task.task_id}: {old_state.name} -> {new_state.name}"
        )

        # 觸發回調
        self._trigger_callback(new_state, old_state)

        return True

    def update_progress(self, percentage: int):
        """更新進度"""
        self.task.progress_percentage = min(100, max(0, percentage))
        logger.debug(f"Task {self.task.task_id}: progress {percentage}%")

    def on_state_change(self, state: TaskState, callback: Callable):
        """註冊狀態變化回調"""
        self.callbacks[state] = callback

    def _trigger_callback(self, new_state: TaskState, old_state: TaskState):
        """觸發回調"""
        if new_state in self.callbacks:
            try:
                self.callbacks[new_state](self.task, old_state)
            except Exception as e:
                logger.error(f"Callback error: {e}")


# 使用示例
if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)

    # 創建任務
    task = Task(
        task_id="task-001",
        task_type="delivery"
    )

    sm = TaskStateMachine(task)

    # 註冊回調
    def on_assigned(task, old_state):
        print(f"📋 Task assigned to station: {task.station_id}")

    def on_completed(task, old_state):
        print(f"✅ Task completed!")

    sm.on_state_change(TaskState.ASSIGNED, on_assigned)
    sm.on_state_change(TaskState.COMPLETED, on_completed)

    # 模擬狀態轉換
    sm.transition(TaskState.ASSIGNED, station_id="station-001")
    sm.transition(TaskState.REASSIGNED, robot_id="robot-001")
    sm.transition(TaskState.EXECUTING)
    sm.update_progress(50)
    sm.transition(TaskState.COMPLETED)

    print(f"\nFinal state: {task}")
```

---

## 4. 本地消息總線 (2.4G 模擬)

### 4.1 設計

模擬 Station 和 Robot 之間的 2.4G 無線通信。

### 4.2 實作骨架

**`simulate/common/message_bus.py`:**

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
本地消息總線 - 模擬 2.4G 無線通信
"""

import asyncio
from typing import Dict, List, Callable, Any
from dataclasses import dataclass
from datetime import datetime
import logging
import json

logger = logging.getLogger(__name__)


@dataclass
class Message:
    """消息"""
    topic: str           # 主題 (如 "task.assign", "task.progress")
    sender_id: str       # 發送者 ID
    receiver_id: str     # 接收者 ID
    payload: Dict        # 消息內容
    timestamp: datetime = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()

    def to_json(self) -> str:
        """轉為 JSON"""
        return json.dumps({
            'topic': self.topic,
            'sender_id': self.sender_id,
            'receiver_id': self.receiver_id,
            'payload': self.payload,
            'timestamp': self.timestamp.isoformat()
        })


class LocalMessageBus:
    """
    本地消息總線

    模擬 Station 和 Robot 之間的 2.4G 無線通信
    使用內存隊列實現消息傳遞
    """

    def __init__(self):
        # 訂閱者: topic -> [(subscriber_id, callback)]
        self.subscribers: Dict[str, List[tuple]] = {}

        # 消息隊列
        self.message_queue: asyncio.Queue = asyncio.Queue()

        # 是否運行
        self.running = False

    def subscribe(self, topic: str, subscriber_id: str, callback: Callable):
        """
        訂閱主題

        Args:
            topic: 主題 (支持通配符 "task.*")
            subscriber_id: 訂閱者 ID
            callback: 回調函數 callback(message: Message)
        """
        if topic not in self.subscribers:
            self.subscribers[topic] = []

        self.subscribers[topic].append((subscriber_id, callback))
        logger.info(f"📡 {subscriber_id} subscribed to '{topic}'")

    def unsubscribe(self, topic: str, subscriber_id: str):
        """取消訂閱"""
        if topic in self.subscribers:
            self.subscribers[topic] = [
                (sid, cb) for sid, cb in self.subscribers[topic]
                if sid != subscriber_id
            ]

    async def publish(self, message: Message):
        """
        發布消息

        Args:
            message: 消息對象
        """
        logger.info(
            f"📤 {message.sender_id} -> {message.receiver_id}: "
            f"{message.topic}"
        )

        await self.message_queue.put(message)

    async def _process_messages(self):
        """處理消息隊列"""
        while self.running:
            try:
                message = await asyncio.wait_for(
                    self.message_queue.get(),
                    timeout=1.0
                )

                # 查找訂閱者
                await self._deliver_message(message)

            except asyncio.TimeoutError:
                continue
            except Exception as e:
                logger.error(f"Message processing error: {e}")

    async def _deliver_message(self, message: Message):
        """投遞消息到訂閱者"""
        delivered = False

        for topic, subscribers in self.subscribers.items():
            # 簡單的主題匹配 (可以擴展為通配符支持)
            if self._topic_matches(message.topic, topic):
                for subscriber_id, callback in subscribers:
                    # 檢查是否是目標接收者 (或廣播)
                    if (message.receiver_id == subscriber_id or
                        message.receiver_id == "*"):
                        try:
                            logger.info(
                                f"📥 {subscriber_id} received: {message.topic}"
                            )

                            # 調用回調
                            if asyncio.iscoroutinefunction(callback):
                                await callback(message)
                            else:
                                callback(message)

                            delivered = True

                        except Exception as e:
                            logger.error(
                                f"Callback error for {subscriber_id}: {e}"
                            )

        if not delivered:
            logger.warning(
                f"⚠️  No subscriber for {message.topic} "
                f"(receiver: {message.receiver_id})"
            )

    def _topic_matches(self, message_topic: str, subscribe_topic: str) -> bool:
        """
        主題匹配

        支持簡單的通配符:
        - "task.*" 匹配 "task.assign", "task.progress"
        - "task.assign" 只匹配精確的 "task.assign"
        """
        if subscribe_topic == message_topic:
            return True

        if subscribe_topic.endswith(".*"):
            prefix = subscribe_topic[:-2]
            return message_topic.startswith(prefix + ".")

        return False

    async def start(self):
        """啟動消息總線"""
        self.running = True
        logger.info("🚀 Message bus started")
        await self._process_messages()

    async def stop(self):
        """停止消息總線"""
        self.running = False
        logger.info("🛑 Message bus stopped")


# 使用示例
async def main():
    logging.basicConfig(level=logging.INFO)

    bus = LocalMessageBus()

    # Station 訂閱
    def on_robot_progress(msg: Message):
        print(f"[Station] Robot progress: {msg.payload}")

    bus.subscribe("task.progress", "station-001", on_robot_progress)

    # Robot 訂閱
    async def on_task_assign(msg: Message):
        print(f"[Robot] Received task: {msg.payload}")

        # 模擬執行並回報進度
        await asyncio.sleep(1)
        await bus.publish(Message(
            topic="task.progress",
            sender_id="robot-001",
            receiver_id="station-001",
            payload={"task_id": msg.payload['task_id'], "progress": 50}
        ))

    bus.subscribe("task.assign", "robot-001", on_task_assign)

    # 啟動總線
    bus_task = asyncio.create_task(bus.start())

    # 模擬 Station 發送任務
    await asyncio.sleep(0.5)
    await bus.publish(Message(
        topic="task.assign",
        sender_id="station-001",
        receiver_id="robot-001",
        payload={"task_id": "task-001", "type": "delivery"}
    ))

    # 運行 3 秒
    await asyncio.sleep(3)
    await bus.stop()
    bus_task.cancel()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 5. 下一步行動

### 5.1 Proto 編譯 (三選一)

**方案 A - 簡化版 proto (推薦快速開始):**
1. 創建 `simulate/proto/jarvis_iot_simple.proto`
2. 編譯生成 Python 代碼
3. 更新 `grpc_client.py` 使用生成的 stub

**方案 B - 下載依賴並編譯:**
1. 下載 `google/api/*.proto` 和 `protoc-gen-swagger/*.proto`
2. 完整編譯 `jarvis_iot.proto`
3. 更新 `grpc_client.py`

**方案 C - 動態加載 protoset:**
1. 使用 Python 的 descriptor_pool 加載 protoset
2. 動態創建消息和 stub
3. 更新 `grpc_client.py`

### 5.2 測試驗證

創建測試腳本驗證基礎功能:

```bash
# 測試 gRPC 連接
python simulate/common/grpc_client.py

# 測試狀態機
python simulate/common/state_machine.py

# 測試消息總線
python simulate/common/message_bus.py
```

### 5.3 完成 Phase 1 的標誌

- ✅ Proto 文件已編譯或可用
- ✅ GrpcClient 可以註冊和登入
- ✅ 狀態機正常工作
- ✅ 消息總線可以傳遞消息

---

## 📝 文件清單

**已創建:**
- `docs/simulate/00-implementation-plan.md` - 總體規劃
- `docs/simulate/01-phase1-infrastructure.md` - 本文檔

**待創建 (Phase 1):**
- `simulate/common/grpc_client.py` - gRPC 客戶端 ⚠️ 骨架已提供
- `simulate/common/state_machine.py` - 狀態機 ⚠️ 骨架已提供
- `simulate/common/message_bus.py` - 消息總線 ⚠️ 骨架已提供
- `simulate/proto/` - Proto 文件和編譯產物
- `simulate/test_phase1.py` - Phase 1 測試腳本

**待創建 (Phase 2+):**
- `simulate/station/` - Station 模擬器 (Phase 2)
- `simulate/robot/` - Robot 模擬器 (Phase 3)

---

**文檔版本**: 1.0
**創建時間**: 2025-11-18
**狀態**: 📝 待實作 - 骨架代碼已提供
**下次更新**: Proto 編譯完成後更新實際代碼
