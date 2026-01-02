# 模擬 Station 與實體 Robot 通訊驗證方案

**文檔版本**: v1.0
**創建時間**: 2025-12-02
**狀態**: 📋 規劃中

---

## 目錄

1. [問題分析](#問題分析)
2. [驗證方案概覽](#驗證方案概覽)
3. [階段 1：純雲端驗證](#階段-1純雲端驗證)
4. [階段 2：雲端中轉驗證](#階段-2雲端中轉驗證)
5. [階段 3：本地通訊驗證](#階段-3本地通訊驗證)
6. [測試代碼實現](#測試代碼實現)
7. [驗證檢查清單](#驗證檢查清單)

---

## 問題分析

### 核心挑戰

模擬 Station 與實體 Robot 通訊面臨的關鍵問題：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           通訊架構                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   模擬 Station (PC)              實體 Robot                             │
│   ┌─────────────┐               ┌─────────────┐                        │
│   │ Python 程式 │               │ 嵌入式系統  │                        │
│   │             │               │             │                        │
│   │ ✅ 4G gRPC  │               │ ✅ 4G gRPC  │                        │
│   │ ❌ 無 2.4G  │               │ ✅ 2.4G 硬體│                        │
│   └──────┬──────┘               └──────┬──────┘                        │
│          │                             │                               │
│          │ 4G                          │ 4G                            │
│          ▼                             ▼                               │
│   ┌─────────────────────────────────────────────────────────┐          │
│   │                    雲平台 (jarvis-iot)                   │          │
│   └─────────────────────────────────────────────────────────┘          │
│                                                                         │
│   ⚠️ 問題：模擬 Station 沒有 2.4G 硬體，無法與實體 Robot 本地通訊        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 通訊層次分析

| 通訊層 | 模擬 Station | 實體 Robot | 是否可驗證 |
|-------|-------------|-----------|-----------|
| **4G gRPC** | ✅ 可實現 | ✅ 已有 | ✅ 可驗證 |
| **2.4G 本地** | ❌ 無硬體 | ✅ 已有 | ⚠️ 需方案 |
| **DAPR 事件** | ✅ 可訂閱 | ✅ 可發布 | ✅ 可驗證 |

---

## 驗證方案概覽

### 三階段驗證路徑

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         驗證路徑                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  階段 1: 純雲端驗證 (無需 2.4G)                                          │
│  ├── 驗證 gRPC 連接 (註冊、登入)                                         │
│  ├── 驗證 StreamUnitControl (接收任務)                                   │
│  ├── 驗證 StreamUnitEvents (上報事件)                                    │
│  └── 驗證與雲平台的基本通訊                                              │
│       │                                                                 │
│       ▼                                                                 │
│  階段 2: 雲端中轉驗證                                                    │
│  ├── 模擬 Station 訂閱任務狀態                                           │
│  ├── 觀察實體 Robot 執行任務                                             │
│  ├── 驗證 DAPR 事件同步                                                  │
│  └── 驗證任務狀態協調                                                    │
│       │                                                                 │
│       ▼                                                                 │
│  階段 3: 本地通訊驗證 (需要硬體)                                          │
│  ├── 獲取 Station 硬體                                                   │
│  ├── 了解 2.4G 模組 API                                                  │
│  └── 在實際硬體上運行模擬軟體                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 方案對比

| 方案 | 描述 | 需要 2.4G | 複雜度 | 可驗證範圍 |
|-----|------|----------|-------|-----------|
| **方案 1** | 純雲端驗證 | ❌ | 低 | 4G 通訊 |
| **方案 2** | 雲端中轉 | ❌ | 中 | 任務協調 |
| **方案 3** | 實際硬體 | ✅ | 高 | 完整流程 |
| **方案 4** | 雲端 2.4G 網關 | ❌ | 很高 | 模擬 2.4G |

---

## 階段 1：純雲端驗證

### 1.1 目標

驗證模擬 Station 與雲平台的 4G gRPC 通訊，不涉及 2.4G。

### 1.2 架構

```
┌──────────────┐     4G gRPC    ┌──────────────┐
│ 模擬 Station │◄──────────────►│   雲平台     │
│              │                │ jarvis-iot   │
│ unit_uid=A   │                │              │
└──────────────┘                └──────────────┘
```

### 1.3 驗證項目

#### 1.3.1 設備註冊

```python
# tests/test_phase1_registration.py

import pytest
from simulate.station import StationSimulator

class TestStationRegistration:
    """測試 Station 註冊流程"""

    @pytest.fixture
    def station(self):
        return StationSimulator(
            unit_sn="SIM_STATION_001",
            grpc_server="robot-grpc.aurotek.com.tw:443"
        )

    async def test_register_new_station(self, station):
        """測試新 Station 註冊"""
        result = await station.register()

        assert result.success, f"註冊失敗: {result.error}"
        assert result.unit_uuid is not None
        assert result.unit_uid > 0

        print(f"✅ 註冊成功:")
        print(f"   unit_uuid: {result.unit_uuid}")
        print(f"   unit_uid: {result.unit_uid}")

    async def test_register_existing_station(self, station):
        """測試已存在的 Station 重複註冊"""
        # 第一次註冊
        result1 = await station.register()

        # 第二次註冊 (應返回相同的 unit_uuid)
        result2 = await station.register()

        assert result1.unit_uuid == result2.unit_uuid
        print(f"✅ 重複註冊返回相同 unit_uuid")
```

#### 1.3.2 設備登入

```python
# tests/test_phase1_login.py

class TestStationLogin:
    """測試 Station 登入流程"""

    async def test_login_after_register(self, station):
        """測試註冊後登入"""
        await station.register()
        result = await station.login()

        assert result.success, f"登入失敗: {result.error}"
        print(f"✅ 登入成功")

    async def test_login_with_station_info(self, station):
        """測試帶 station_info 的登入"""
        await station.register()
        result = await station.login(
            station_info={
                "hostname": "sim-station-001",
                "version": "1.0.0",
                "ip_address": "192.168.1.100"
            }
        )

        assert result.success
        print(f"✅ 帶 station_info 登入成功")
```

#### 1.3.3 StreamUnitControl (接收任務)

```python
# tests/test_phase1_control_stream.py

class TestStreamUnitControl:
    """測試 StreamUnitControl 流"""

    async def test_establish_control_stream(self, station):
        """測試建立控制流"""
        await station.start()

        # 驗證流已建立
        assert station.control_stream_active
        print(f"✅ StreamUnitControl 流已建立")

    async def test_receive_heartbeat_request(self, station):
        """測試接收心跳請求"""
        await station.start()

        # 等待雲端發送心跳請求
        request = await station.wait_for_control_request(timeout=30)

        assert request is not None
        print(f"✅ 收到控制請求: {request.type}")

    async def test_receive_task_assignment(self, station):
        """測試接收任務分配 (需手動觸發)"""
        await station.start()

        print("⏳ 請在雲平台創建任務，分配給此 Station...")
        print(f"   Station unit_uid: {station.unit_uid}")

        task = await station.wait_for_task(timeout=120)

        if task:
            print(f"✅ 收到任務:")
            print(f"   task_id: {task.task_id}")
            print(f"   task_type: {task.task_type}")
        else:
            print("⚠️ 超時未收到任務")
```

#### 1.3.4 StreamUnitEvents (上報事件)

```python
# tests/test_phase1_events_stream.py

class TestStreamUnitEvents:
    """測試 StreamUnitEvents 流"""

    async def test_report_info_event(self, station):
        """測試上報 INFO 事件"""
        await station.start()

        result = await station.report_event(
            event_type="INFO",
            event_name="station_ready",
            event_info={"status": "ready", "version": "1.0.0"}
        )

        assert result.success
        print(f"✅ INFO 事件上報成功")

    async def test_report_error_event(self, station):
        """測試上報 ERROR 事件"""
        await station.start()

        result = await station.report_event(
            event_type="ERROR",
            event_name="connection_failed",
            event_info={"target": "robot_001", "reason": "timeout"},
            event_logs="Failed to connect to robot_001 after 30s"
        )

        assert result.success
        print(f"✅ ERROR 事件上報成功")

    async def test_report_span_event(self, station):
        """測試上報 SPAN 事件 (開始/結束)"""
        await station.start()

        # 開始
        span_uuid = await station.report_span_start(
            span_name="task_processing",
            span_info={"task_id": "12345"}
        )

        # 模擬處理
        await asyncio.sleep(2)

        # 結束
        result = await station.report_span_end(
            span_uuid=span_uuid,
            span_info={"task_id": "12345", "result": "success"}
        )

        assert result.success
        print(f"✅ SPAN 事件上報成功 (duration: 2s)")
```

### 1.4 階段 1 驗證檢查清單

```markdown
## 階段 1 檢查清單

### 設備註冊
- [ ] RSRegisterUnit 調用成功
- [ ] 獲得 unit_uuid
- [ ] 獲得 unit_uid
- [ ] 重複註冊返回相同 unit_uuid

### 設備登入
- [ ] RSLoginUnit 調用成功
- [ ] station_info 正確傳遞
- [ ] 登入後可建立流

### StreamUnitControl
- [ ] 流成功建立
- [ ] 收到雲端心跳請求
- [ ] 能正確回應請求
- [ ] 收到任務分配 (手動觸發)

### StreamUnitEvents
- [ ] 流成功建立
- [ ] INFO 事件上報成功
- [ ] ERROR 事件上報成功
- [ ] SPAN 事件上報成功
- [ ] 雲端確認收到 (processed_offset 更新)
```

---

## 階段 2：雲端中轉驗證

### 2.1 目標

驗證模擬 Station 能夠通過雲平台觀察實體 Robot 的任務執行狀態。

### 2.2 架構

```
┌──────────────┐                              ┌──────────────┐
│ 模擬 Station │                              │ 實體 Robot   │
│              │                              │              │
│ 訂閱任務狀態  │◄──────── 雲端中轉 ──────────►│ 執行任務     │
│              │                              │ 回報狀態     │
└──────────────┘                              └──────────────┘
        │                                            │
        │ 4G                                         │ 4G
        ▼                                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      雲平台                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              任務調度中心                               │  │
│  │                                                        │  │
│  │  • 創建任務                                            │  │
│  │  • 直接分配給 Robot (繞過 Station 2.4G)                │  │
│  │  • 同步任務狀態給 Station                              │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              DAPR EventBus                             │  │
│  │                                                        │  │
│  │  • TaskUpdateSignal 事件                               │  │
│  │  • Station 訂閱                                        │  │
│  │  • Robot 發布                                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 時序圖

```
雲平台                模擬Station              實體Robot
  │                      │                        │
  │ 1.創建任務            │                        │
  │ (指定Robot)          │                        │
  │──────────────────────┼───────────────────────►│
  │                      │                        │ 2.Robot收到任務
  │ 3.通知Station         │                        │
  │ (任務狀態同步)        │                        │
  │─────────────────────►│                        │
  │                      │ 4.Station觀察          │
  │                      │                        │ 5.Robot開始執行
  │                      │                        │
  │◄─────────────────────┼────────────────────────│ 6.Robot回報進度
  │ 7.轉發狀態給Station   │                        │   (20%)
  │─────────────────────►│                        │
  │                      │                        │
  │◄─────────────────────┼────────────────────────│ 8.Robot回報進度
  │ 9.轉發狀態給Station   │                        │   (50%)
  │─────────────────────►│                        │
  │                      │                        │
  │◄─────────────────────┼────────────────────────│ 10.Robot完成
  │ 11.通知Station        │                        │
  │─────────────────────►│                        │
  │                      │ 12.Station確認完成     │
```

### 2.4 驗證項目

#### 2.4.1 訂閱任務更新

```python
# tests/test_phase2_task_subscription.py

class TestTaskSubscription:
    """測試任務狀態訂閱"""

    async def test_subscribe_site_tasks(self, station):
        """測試訂閱站點任務"""
        await station.start()

        # 訂閱站點 504 的任務更新
        result = await station.subscribe_task_updates(site_uid=504)

        assert result.success
        print(f"✅ 已訂閱站點 504 的任務更新")

    async def test_receive_task_update(self, station):
        """測試接收任務更新"""
        await station.start()
        await station.subscribe_task_updates(site_uid=504)

        print("⏳ 等待實體 Robot 執行任務...")

        update = await station.wait_for_task_update(timeout=300)

        if update:
            print(f"✅ 收到任務更新:")
            print(f"   task_id: {update.task_id}")
            print(f"   unit_uid: {update.unit_uid}")
            print(f"   status: {update.status}")
        else:
            print("⚠️ 超時未收到更新")
```

#### 2.4.2 觀察 Robot 執行

```python
# tests/test_phase2_observe_robot.py

class TestObserveRobot:
    """測試觀察實體 Robot 執行"""

    async def test_observe_robot_task_execution(self, station):
        """觀察實體 Robot 執行任務的完整流程"""
        await station.start()
        await station.subscribe_task_updates(site_uid=504)

        print("=" * 60)
        print("觀察實體 Robot 執行任務")
        print("=" * 60)
        print("請確保:")
        print("  1. 實體 Robot 已上線")
        print("  2. 雲平台已創建任務並分配給 Robot")
        print("=" * 60)

        # 收集所有狀態更新
        updates = []
        expected_states = ["ASSIGNED", "EXECUTING", "COMPLETED"]

        async for update in station.task_update_stream(timeout=600):
            updates.append(update)

            timestamp = update.timestamp.strftime("%H:%M:%S")
            print(f"[{timestamp}] Task {update.task_id}: {update.status}")

            if update.status == "COMPLETED":
                break

        # 驗證狀態流轉
        statuses = [u.status for u in updates]
        print(f"\n狀態流轉: {' -> '.join(statuses)}")

        assert len(updates) > 0, "未收到任何更新"
        assert "COMPLETED" in statuses, "任務未完成"

        print(f"\n✅ 成功觀察到任務完成，共 {len(updates)} 次更新")

    async def test_observe_robot_air_says(self, station):
        """觀察 Robot 的 air_says (2.4G 消息)"""
        await station.start()
        await station.subscribe_robot_heartbeat(site_uid=504)

        print("⏳ 觀察 Robot 的 air_says...")

        heartbeats = []
        async for hb in station.robot_heartbeat_stream(timeout=60):
            heartbeats.append(hb)

            print(f"Robot {hb.unit_uid}:")
            print(f"  nearfield_status: {hb.nearfield_status}")
            print(f"  air_says count: {len(hb.air_says)}")

            for air_say in hb.air_says:
                print(f"    - type: {air_say.msg_type}, created_at: {air_say.created_at}")

            if len(heartbeats) >= 5:
                break

        assert len(heartbeats) > 0
        print(f"\n✅ 收到 {len(heartbeats)} 次心跳")
```

#### 2.4.3 Station-Robot 協調

```python
# tests/test_phase2_coordination.py

class TestStationRobotCoordination:
    """測試 Station 與 Robot 的雲端協調"""

    async def test_task_lifecycle_observation(self, station):
        """觀察完整任務生命週期"""
        await station.start()

        # 訂閱任務和 Robot 狀態
        await station.subscribe_task_updates(site_uid=504)
        await station.subscribe_robot_heartbeat(site_uid=504)

        print("=" * 60)
        print("任務生命週期觀察")
        print("=" * 60)

        task_states = {}
        robot_states = {}

        async def collect_updates():
            async for update in station.combined_stream(timeout=600):
                if update.type == "task":
                    task_states[update.task_id] = update.status
                    print(f"[TASK] {update.task_id}: {update.status}")
                elif update.type == "robot":
                    robot_states[update.unit_uid] = {
                        "space_state": update.space_state,
                        "purpose_state": update.purpose_state,
                        "nearfield_status": update.nearfield_status
                    }
                    print(f"[ROBOT] {update.unit_uid}: space={update.space_state}, purpose={update.purpose_state}")

        await collect_updates()

        print("\n任務最終狀態:", task_states)
        print("Robot 最終狀態:", robot_states)

    async def test_station_acknowledge_robot_completion(self, station):
        """測試 Station 確認 Robot 完成任務"""
        await station.start()
        await station.subscribe_task_updates(site_uid=504)

        # 等待任務完成
        update = await station.wait_for_task_status("COMPLETED", timeout=600)

        if update:
            # Station 確認收到完成通知
            await station.report_event(
                event_type="INFO",
                event_name="task_completion_observed",
                event_info={
                    "task_id": update.task_id,
                    "robot_unit_uid": update.unit_uid,
                    "completed_at": update.timestamp.isoformat()
                }
            )

            print(f"✅ Station 確認 Robot 完成任務 {update.task_id}")
```

### 2.5 階段 2 驗證檢查清單

```markdown
## 階段 2 檢查清單

### 任務訂閱
- [ ] 成功訂閱站點任務更新
- [ ] 收到 TaskUpdateSignal 事件
- [ ] 正確解析任務狀態

### Robot 觀察
- [ ] 收到 Robot 心跳數據
- [ ] 能看到 nearfield_status
- [ ] 能看到 air_says 數組
- [ ] 能追蹤 Robot 狀態變化

### 協調驗證
- [ ] 觀察到任務 ASSIGNED 狀態
- [ ] 觀察到任務 EXECUTING 狀態
- [ ] 觀察到任務 COMPLETED 狀態
- [ ] Station 能確認任務完成
```

---

## 階段 3：本地通訊驗證

### 3.1 目標

使用實際 Station 硬體，驗證完整的 2.4G 本地通訊。

### 3.2 前提條件

```
需要準備:
1. 實際 Station 硬體設備
2. Station 硬體的 2.4G 模組 API 文檔
3. Station 嵌入式系統的開發環境
4. 實體 Robot 設備
```

### 3.3 方案選項

#### 方案 A：在 Station 硬體上運行模擬軟體

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     實際 Station 硬體                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    模擬軟體 (Python)                             │   │
│  │                                                                  │   │
│  │   station_simulator.py                                          │   │
│  │       │                                                         │   │
│  │       ├── 調用 4G gRPC API (與雲平台通訊)                        │   │
│  │       │                                                         │   │
│  │       └── 調用 2.4G 硬體 API (與 Robot 通訊)                     │   │
│  │               │                                                 │   │
│  │               ▼                                                 │   │
│  │       ┌─────────────────────┐                                   │   │
│  │       │ 2.4G 硬體抽象層    │                                   │   │
│  │       │ (需要實現)         │                                   │   │
│  │       └─────────────────────┘                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                     │
│  │ 4G 模組     │  │ 2.4G 模組   │  │ 其他硬體    │                     │
│  └─────────────┘  └─────────────┘  └─────────────┘                     │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 方案 B：2.4G 硬體抽象層

```python
# simulate/common/nearfield_hardware.py

from abc import ABC, abstractmethod
from typing import List, Callable

class NearfieldHardwareInterface(ABC):
    """2.4G 硬體抽象介面"""

    @abstractmethod
    def initialize(self) -> bool:
        """初始化 2.4G 硬體"""
        pass

    @abstractmethod
    def send_message(self, target_unit_sid: int, message: bytes) -> bool:
        """發送 2.4G 消息"""
        pass

    @abstractmethod
    def register_callback(self, callback: Callable[[int, bytes], None]):
        """註冊接收消息回調"""
        pass

    @abstractmethod
    def get_connected_units(self) -> List[int]:
        """獲取已連接的設備列表"""
        pass

    @abstractmethod
    def get_nearfield_status(self) -> bool:
        """獲取 2.4G 連線狀態"""
        pass


class SimulatedNearfield(NearfieldHardwareInterface):
    """模擬的 2.4G 硬體 (用於純軟體測試)"""

    def __init__(self, message_bus):
        self.message_bus = message_bus
        self.callback = None

    def initialize(self) -> bool:
        return True

    def send_message(self, target_unit_sid: int, message: bytes) -> bool:
        # 通過軟體消息總線模擬
        return self.message_bus.send(target_unit_sid, message)

    def register_callback(self, callback):
        self.callback = callback
        self.message_bus.subscribe(callback)

    def get_connected_units(self) -> List[int]:
        return self.message_bus.get_connected_units()

    def get_nearfield_status(self) -> bool:
        return len(self.get_connected_units()) > 0


class RealNearfield(NearfieldHardwareInterface):
    """真實的 2.4G 硬體 (需要在 Station 硬體上運行)"""

    def __init__(self, device_path: str = "/dev/ttyUSB0"):
        self.device_path = device_path
        self.serial = None
        self.callback = None

    def initialize(self) -> bool:
        """初始化串口連接到 2.4G 模組"""
        try:
            import serial
            self.serial = serial.Serial(
                self.device_path,
                baudrate=115200,
                timeout=1
            )
            return True
        except Exception as e:
            print(f"初始化 2.4G 硬體失敗: {e}")
            return False

    def send_message(self, target_unit_sid: int, message: bytes) -> bool:
        """通過 2.4G 硬體發送消息"""
        if not self.serial:
            return False

        # 構建硬體協議數據包 (需要根據實際協議)
        packet = self._build_packet(target_unit_sid, message)
        self.serial.write(packet)
        return True

    def _build_packet(self, target: int, payload: bytes) -> bytes:
        """構建 2.4G 協議數據包 (需要根據實際協議實現)"""
        # TODO: 根據實際 2.4G 協議實現
        header = bytes([0xAA, 0x55])
        target_bytes = target.to_bytes(2, 'little')
        length = len(payload).to_bytes(2, 'little')
        return header + target_bytes + length + payload

    # ... 其他方法實現
```

### 3.4 驗證測試

```python
# tests/test_phase3_local_communication.py

class TestLocalCommunication:
    """測試本地 2.4G 通訊 (需要硬體)"""

    @pytest.fixture
    def station_with_hardware(self):
        """創建帶硬體的 Station"""
        from simulate.common.nearfield_hardware import RealNearfield

        nearfield = RealNearfield("/dev/ttyUSB0")
        assert nearfield.initialize(), "2.4G 硬體初始化失敗"

        station = StationSimulator(
            unit_sn="STZJGF4025030067",
            nearfield=nearfield
        )
        return station

    async def test_robot_handshake(self, station_with_hardware):
        """測試與實體 Robot 的 2.4G 握手"""
        station = station_with_hardware
        await station.start()

        print("⏳ 等待 Robot 進入 Station 範圍...")

        # 等待 Robot 連接
        connected = await station.wait_for_robot_connection(timeout=120)

        assert connected
        print(f"✅ Robot 已連接: {station.connected_robots}")

    async def test_send_task_to_robot(self, station_with_hardware):
        """測試通過 2.4G 發送任務給 Robot"""
        station = station_with_hardware
        await station.start()

        # 等待 Robot 連接
        await station.wait_for_robot_connection(timeout=120)

        # 獲取連接的 Robot
        robot_sid = station.connected_robots[0]

        # 發送任務
        result = await station.send_task_to_robot(
            robot_sid=robot_sid,
            task={
                "task_id": "test_001",
                "task_type": "delivery",
                "destination_poi": 15
            }
        )

        assert result.success
        print(f"✅ 任務已發送給 Robot {robot_sid}")

    async def test_receive_robot_status(self, station_with_hardware):
        """測試接收 Robot 的 2.4G 狀態更新"""
        station = station_with_hardware
        await station.start()

        await station.wait_for_robot_connection(timeout=120)

        # 收集 Robot 狀態更新
        updates = []
        async for update in station.robot_status_stream(timeout=60):
            updates.append(update)
            print(f"Robot 狀態: {update}")

            if len(updates) >= 10:
                break

        assert len(updates) > 0
        print(f"✅ 收到 {len(updates)} 次 Robot 狀態更新")
```

### 3.5 階段 3 驗證檢查清單

```markdown
## 階段 3 檢查清單 (需要硬體)

### 硬體準備
- [ ] 獲得 Station 硬體設備
- [ ] 獲得 2.4G 模組 API 文檔
- [ ] 配置開發環境
- [ ] 實現 RealNearfield 類

### 2.4G 通訊
- [ ] 2.4G 硬體初始化成功
- [ ] 檢測到 Robot 連接
- [ ] 成功發送消息給 Robot
- [ ] 成功接收 Robot 狀態

### 完整流程
- [ ] Robot 進入 Station
- [ ] Station 發送任務給 Robot (2.4G)
- [ ] Robot 執行任務
- [ ] Robot 回報狀態 (2.4G)
- [ ] Station 上報給雲平台 (4G)
```

---

## 測試代碼實現

### 目錄結構

```
simulate/
├── tests/
│   ├── __init__.py
│   ├── conftest.py                    # pytest fixtures
│   │
│   ├── phase1/                        # 階段 1 測試
│   │   ├── test_registration.py
│   │   ├── test_login.py
│   │   ├── test_control_stream.py
│   │   └── test_events_stream.py
│   │
│   ├── phase2/                        # 階段 2 測試
│   │   ├── test_task_subscription.py
│   │   ├── test_observe_robot.py
│   │   └── test_coordination.py
│   │
│   └── phase3/                        # 階段 3 測試
│       └── test_local_communication.py
│
├── common/
│   ├── nearfield_hardware.py          # 2.4G 硬體抽象
│   └── ...
│
└── station/
    └── station_simulator.py
```

### conftest.py

```python
# simulate/tests/conftest.py

import pytest
import asyncio

@pytest.fixture(scope="session")
def event_loop():
    """創建事件循環"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture
def station():
    """創建模擬 Station (軟體模擬)"""
    from simulate.station import StationSimulator
    return StationSimulator(
        unit_sn="SIM_STATION_001",
        grpc_server="robot-grpc.aurotek.com.tw:443"
    )

@pytest.fixture
def station_with_nearfield_sim():
    """創建帶軟體模擬 2.4G 的 Station"""
    from simulate.station import StationSimulator
    from simulate.common.nearfield_bus import NearfieldBus
    from simulate.common.nearfield_hardware import SimulatedNearfield

    bus = NearfieldBus()
    nearfield = SimulatedNearfield(bus)

    return StationSimulator(
        unit_sn="SIM_STATION_001",
        grpc_server="robot-grpc.aurotek.com.tw:443",
        nearfield=nearfield
    )

# 階段 3 專用 fixture (需要硬體)
@pytest.fixture
def station_with_hardware():
    """創建帶真實 2.4G 硬體的 Station (需要在 Station 硬體上運行)"""
    from simulate.station import StationSimulator
    from simulate.common.nearfield_hardware import RealNearfield

    nearfield = RealNearfield("/dev/ttyUSB0")
    if not nearfield.initialize():
        pytest.skip("2.4G 硬體不可用")

    return StationSimulator(
        unit_sn="STZJGF4025030067",  # 真實 Station SN
        grpc_server="robot-grpc.aurotek.com.tw:443",
        nearfield=nearfield
    )
```

### 運行測試

```bash
# 運行階段 1 測試 (純雲端)
pytest simulate/tests/phase1/ -v

# 運行階段 2 測試 (雲端中轉)
pytest simulate/tests/phase2/ -v

# 運行階段 3 測試 (需要硬體)
pytest simulate/tests/phase3/ -v --hardware

# 運行所有測試
pytest simulate/tests/ -v

# 生成測試報告
pytest simulate/tests/ -v --html=test_report.html
```

---

## 驗證檢查清單

### 總體檢查清單

```markdown
# 模擬 Station 與實體 Robot 通訊驗證檢查清單

## 階段 1：純雲端驗證 ✅ 可立即開始
- [ ] 設備註冊成功
- [ ] 設備登入成功
- [ ] StreamUnitControl 流建立
- [ ] StreamUnitEvents 流建立
- [ ] 收到雲端任務分配

## 階段 2：雲端中轉驗證 ✅ 可立即開始
- [ ] 訂閱站點任務更新
- [ ] 收到 Robot 心跳數據
- [ ] 觀察到任務狀態流轉
- [ ] Station 確認 Robot 完成

## 階段 3：本地通訊驗證 ⚠️ 需要硬體
- [ ] 獲得 Station 硬體
- [ ] 實現 2.4G 硬體介面
- [ ] Robot 2.4G 握手成功
- [ ] 2.4G 任務發送成功
- [ ] 2.4G 狀態接收成功

## 最終驗證
- [ ] 完整任務閉環測試通過
- [ ] 性能測試通過
- [ ] 穩定性測試通過
```

---

## 附錄

### A. 相關文檔

| 文檔 | 路徑 | 說明 |
|-----|------|------|
| 實作計劃 | `00-implementation-plan.md` | 總體實作規劃 |
| Phase 1 指南 | `01-phase1-infrastructure.md` | 基礎設施實作 |
| 2.4G 模擬增強 | `02-2.4g-simulation-enhancement.md` | 2.4G 消息模擬 |
| Proto 定義 | `proto_from_server/jarvis_iot.proto` | gRPC API 定義 |

### B. 實際 Log 參考

驗證時可參考實際設備的 Log 格式：

```
# 來源: clue/delivery-actor-robot_1542.md

# Robot 心跳數據 (包含 air_says)
robot_say=PassDoorRobotAirSay(...)
air_says=[
    SAirSay(msg_type=STATION_ROBOT_SAY, say='IwqAMwEQ...', created_at=1761637422474),
    SAirSay(msg_type=PASS_DOOR_ROBOT_SAY, say='IgowYwEQ...', created_at=1761629219582),
]
nearfield_status=True
```

### C. 常見問題

**Q: 沒有 2.4G 硬體，能驗證什麼？**

A: 可以完成階段 1 和階段 2，驗證：
- 4G gRPC 通訊
- 雲平台任務協調
- DAPR 事件訂閱
- 任務狀態觀察

**Q: 如何獲得 2.4G 硬體 API？**

A: 需要聯繫硬體團隊獲取：
- 2.4G 模組技術文檔
- 串口通訊協議
- SDK 或示例代碼

**Q: 階段 2 能驗證到什麼程度？**

A: 可以驗證模擬 Station 能夠：
- 觀察 Robot 執行任務
- 收到任務狀態更新
- 與雲平台正確協調

唯一無法驗證的是「模擬 Station 直接控制 Robot」，這需要 2.4G 通訊。

---

**文檔版本**: v1.0
**創建者**: AI Assistant
**相關文件**:
- `00-implementation-plan.md`
- `02-2.4g-simulation-enhancement.md`
- `clue/*1542.md` (實際 log)
