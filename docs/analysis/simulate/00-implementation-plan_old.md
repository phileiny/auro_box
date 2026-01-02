# Station & Robot 模擬器實作規劃

**文檔版本**: v1.0
**創建時間**: 2025-11-18
**目標**: 完整實現 Station 和 Robot 模擬器，模擬任務閉環流程
**狀態**: 📋 規劃中

---

## 📋 目錄

1. [總體目標](#總體目標)
2. [系統架構設計](#系統架構設計)
3. [模組劃分](#模組劃分)
4. [實作階段](#實作階段)
5. [測試計劃](#測試計劃)
6. [可交付成果](#可交付成果)

---

## 總體目標

### 🎯 核心目標

實作兩個模擬程式，模擬真實的 Station 和 Robot 設備：

1. **Station 模擬器** (`simulate_station.py`)
   - 模擬 Station 設備的完整行為
   - 通過 4G gRPC 與雲平台通信
   - 接收任務並轉派給 Robot（模擬 2.4G 本地通信）

2. **Robot 模擬器** (`simulate_robot.py`)
   - 模擬 Robot 設備的完整行為
   - 通過 4G gRPC 與雲平台通信
   - 從 Station 接收任務（模擬 2.4G 本地通信）
   - 執行任務並回報狀態

3. **完整任務閉環**
   ```
   任務創建 → 分配到Station → Station轉派給Robot
   → Robot執行 → 進度回報 → 任務完成
   ```

### 🎨 設計原則

1. **真實性**: 使用真實的 gRPC 端點和 proto 定義
2. **模組化**: 可重用的組件設計
3. **可測試**: 每個階段可獨立測試
4. **可擴展**: 易於添加新功能
5. **可觀察**: 詳細的日誌和狀態輸出

---

## 系統架構設計

### 整體架構圖

```
┌─────────────────────────────────────────────────────────┐
│                    雲平台 (Aurotek Cloud)               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ jarvis-iot-v6│  │app-delivery  │  │ DAPR EventBus│  │
│  │              │  │   -rpc       │  │              │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼─────────┘
          │ 4G gRPC          │ 4G gRPC          │
          │ :443             │ :443             │
          │                  │                  │
    ┌─────▼──────┐      ┌────▼─────┐            │
    │  Station   │      │  Robot   │            │
    │ Simulator  │◄────►│ Simulator│            │
    └────────────┘ 2.4G └──────────┘            │
         │         本地協作       │              │
         │                       │              │
         └───────────────────────┴──────────────┘
              模擬設備層 (本地運行)
```

### 通信層次

#### Layer 1: 雲平台 gRPC 通信 (4G)

**Station ↔ 雲平台**:
- `RSRegisterUnit` - 註冊
- `RSLoginUnit` - 登入
- `StreamUnitControl` - 流式控制（接收任務下發）
- `StreamUnitEvents` - 流式事件（上報狀態）

**Robot ↔ 雲平台**:
- `RSRegisterUnit` - 註冊
- `RSLoginUnit` - 登入
- `StreamUnitControl` - 流式控制（接收任務）
- `StreamUnitEvents` - 流式事件（上報執行進度）

#### Layer 2: 本地協作 (2.4G 模擬)

**Station ↔ Robot**:
- 使用本地事件總線模擬（Queue/Event）
- Station 推送任務給 Robot
- Robot 回報狀態給 Station
- Station 聚合回報給雲平台

---

## 模組劃分

### 共用模組 (`common/`)

#### 1. gRPC 客戶端模組 (`grpc_client.py`)

```python
class GrpcClient:
    """統一的 gRPC 客戶端封裝"""

    def __init__(self, server_url: str):
        self.server_url = server_url
        self.channel = None
        self.stub = None

    def connect(self):
        """建立 TLS 連接"""

    def register_unit(self, unit_sn: str, unit_type: int):
        """註冊設備"""

    def login_unit(self, unit_uuid: str, unit_sn: str, unit_type: int):
        """登入設備"""

    def start_control_stream(self, callback):
        """啟動控制流（接收任務）"""

    def start_event_stream(self, event_generator):
        """啟動事件流（上報狀態）"""
```

#### 2. 狀態機模組 (`state_machine.py`)

```python
class TaskStateMachine:
    """任務狀態機"""

    States = Enum('States', [
        'CREATED', 'ASSIGNED', 'REASSIGNED',
        'EXECUTING', 'COMPLETED', 'FAILED'
    ])

    def transition(self, event: str) -> bool:
        """狀態轉換"""

    def get_current_state(self) -> States:
        """獲取當前狀態"""
```

#### 3. 事件處理模組 (`event_handler.py`)

```python
class EventHandler:
    """事件處理器"""

    def handle_task_assigned(self, task):
        """處理任務分配事件"""

    def handle_task_update(self, task):
        """處理任務更新事件"""

    def emit_event(self, event_type: str, payload: dict):
        """發送事件"""
```

#### 4. 本地消息總線 (`message_bus.py`)

```python
class LocalMessageBus:
    """模擬 2.4G 本地通信的消息總線"""

    def __init__(self):
        self.subscribers = {}
        self.queue = Queue()

    def publish(self, topic: str, message: dict):
        """發布消息"""

    def subscribe(self, topic: str, callback):
        """訂閱消息"""

    def start(self):
        """啟動消息總線"""
```

### Station 模組 (`station/`)

#### 1. Station 主程式 (`station_simulator.py`)

```python
class StationSimulator:
    """Station 模擬器主類"""

    def __init__(self, unit_sn: str):
        self.unit_sn = unit_sn
        self.unit_uuid = None
        self.unit_uid = None
        self.grpc_client = GrpcClient(GRPC_SERVER)
        self.message_bus = LocalMessageBus()
        self.task_queue = []

    async def start(self):
        """啟動 Station"""
        # 1. 註冊
        # 2. 登入
        # 3. 啟動控制流（接收任務）
        # 4. 啟動事件流（上報狀態）
        # 5. 啟動本地消息總線

    def on_task_received(self, task):
        """收到任務回調"""
        # 1. 記錄任務
        # 2. 轉派給 Robot（通過本地總線）
        # 3. 上報任務轉派狀態

    def on_robot_progress(self, robot_id, task_id, progress):
        """收到 Robot 進度回調"""
        # 1. 聚合進度
        # 2. 上報給雲平台
```

#### 2. Station 任務管理 (`station_task_manager.py`)

```python
class StationTaskManager:
    """Station 任務管理器"""

    def assign_task_to_robot(self, task, robot_id):
        """分配任務給 Robot"""

    def track_robot_progress(self, task_id):
        """追蹤 Robot 執行進度"""

    def aggregate_status(self):
        """聚合所有任務狀態"""
```

### Robot 模組 (`robot/`)

#### 1. Robot 主程式 (`robot_simulator.py`)

```python
class RobotSimulator:
    """Robot 模擬器主類"""

    def __init__(self, unit_sn: str):
        self.unit_sn = unit_sn
        self.unit_uuid = None
        self.unit_uid = None
        self.grpc_client = GrpcClient(GRPC_SERVER)
        self.message_bus = LocalMessageBus()
        self.current_task = None

    async def start(self):
        """啟動 Robot"""
        # 1. 註冊
        # 2. 登入
        # 3. 訂閱本地消息總線（接收 Station 任務）
        # 4. 啟動事件流（上報執行進度）

    def on_task_received_from_station(self, task):
        """從 Station 收到任務"""
        # 1. 接受任務
        # 2. 開始執行
        # 3. 定期回報進度

    async def execute_task(self, task):
        """執行任務"""
        # 1. 模擬任務執行（移動、取貨、送貨）
        # 2. 每個階段回報進度
        # 3. 完成後回報結果
```

#### 2. Robot 任務執行器 (`robot_task_executor.py`)

```python
class RobotTaskExecutor:
    """Robot 任務執行器"""

    async def execute_delivery_task(self, task):
        """執行配送任務"""
        # 1. 前往取貨點 (MOVING_TO_PICKUP)
        # 2. 到達取貨點 (ARRIVED_AT_PICKUP)
        # 3. 取貨 (PICKING_UP)
        # 4. 前往送貨點 (MOVING_TO_DELIVERY)
        # 5. 到達送貨點 (ARRIVED_AT_DELIVERY)
        # 6. 送貨 (DELIVERING)
        # 7. 完成 (COMPLETED)

    def simulate_movement(self, from_loc, to_loc):
        """模擬移動"""

    def report_progress(self, status, percentage):
        """回報進度"""
```

---

## 實作階段

### Phase 1: 基礎設施 (Week 1)

**目標**: 建立基礎的 gRPC 通信和認證機制

#### 任務清單

- [ ] **Task 1.1**: 編譯服務器 proto 文件
  - 下載缺失的依賴（google/api/*.proto）
  - 編譯 `proto_from_server/jarvis_iot.proto`
  - 生成 Python pb2 文件
  - 測試導入

- [ ] **Task 1.2**: 實現 GrpcClient 基礎類
  - TLS 連接
  - RSRegisterUnit 方法
  - RSLoginUnit 方法
  - 錯誤處理

- [ ] **Task 1.3**: 實現認證流程
  - 設備註冊
  - 設備登入
  - 保存 unit_uuid 和 unit_uid

- [ ] **Task 1.4**: 單元測試
  - 測試註冊流程
  - 測試登入流程
  - 測試連接穩定性

**可交付成果**:
- `common/grpc_client.py` - 可用的 gRPC 客戶端
- `test_registration.py` - 註冊測試腳本
- 文檔: `docs/simulate/01-grpc-client-guide.md`

---

### Phase 2: Station 模擬器 (Week 2)

**目標**: 實現 Station 基本功能

#### 任務清單

- [ ] **Task 2.1**: 實現 StreamUnitControl
  - 建立流式連接
  - 接收雲端下發的任務
  - 解析任務數據

- [ ] **Task 2.2**: 實現 StreamUnitEvents
  - 建立上行流式連接
  - 發送心跳事件
  - 發送狀態更新事件

- [ ] **Task 2.3**: 實現本地消息總線
  - 發布/訂閱機制
  - Queue 管理
  - 線程安全

- [ ] **Task 2.4**: 實現任務轉派邏輯
  - 接收任務 → 記錄 → 轉派
  - 通過本地總線推送給 Robot
  - 狀態聚合和上報

- [ ] **Task 2.5**: 整合測試
  - 啟動 Station 模擬器
  - 手動觸發任務分配
  - 驗證任務接收

**可交付成果**:
- `station/station_simulator.py` - Station 主程式
- `common/message_bus.py` - 本地消息總線
- `test_station.py` - Station 測試腳本
- 文檔: `docs/simulate/02-station-simulator-guide.md`

---

### Phase 3: Robot 模擬器 (Week 3)

**目標**: 實現 Robot 基本功能

#### 任務清單

- [ ] **Task 3.1**: 實現 Robot gRPC 通信
  - 註冊和登入
  - StreamUnitEvents

- [ ] **Task 3.2**: 實現本地任務接收
  - 訂閱 Station 消息
  - 解析任務數據

- [ ] **Task 3.3**: 實現任務狀態機
  - 定義所有狀態
  - 狀態轉換邏輯
  - 狀態驗證

- [ ] **Task 3.4**: 實現任務執行器
  - 模擬移動
  - 模擬取貨/送貨
  - 進度計算

- [ ] **Task 3.5**: 實現進度回報
  - 每個階段回報
  - 異常處理
  - 完成回報

- [ ] **Task 3.6**: 整合測試
  - 啟動 Robot 模擬器
  - 接收 Station 任務
  - 執行並回報

**可交付成果**:
- `robot/robot_simulator.py` - Robot 主程式
- `robot/task_executor.py` - 任務執行器
- `common/state_machine.py` - 狀態機
- `test_robot.py` - Robot 測試腳本
- 文檔: `docs/simulate/03-robot-simulator-guide.md`

---

### Phase 4: 完整閉環 (Week 4)

**目標**: 整合 Station 和 Robot，實現完整任務閉環

#### 任務清單

- [ ] **Task 4.1**: 集成 Station + Robot
  - 同時啟動兩個模擬器
  - 驗證本地通信
  - 驗證雲端通信

- [ ] **Task 4.2**: 實現完整任務流程
  - 任務創建（手動觸發或 API）
  - Station 接收任務
  - Station 轉派給 Robot
  - Robot 執行並回報
  - Station 聚合並上報
  - 任務完成

- [ ] **Task 4.3**: 實現異常處理
  - 任務超時
  - Robot 離線
  - 任務失敗
  - 重試機制

- [ ] **Task 4.4**: 性能優化
  - 連接池
  - 並發處理
  - 資源清理

- [ ] **Task 4.5**: 端到端測試
  - 完整流程測試
  - 壓力測試
  - 穩定性測試

**可交付成果**:
- `main.py` - 統一啟動腳本
- `test_end_to_end.py` - 端到端測試
- 文檔: `docs/simulate/04-complete-loop-guide.md`

---

### Phase 5: 優化和擴展 (Week 5)

**目標**: 增強功能和可用性

#### 任務清單

- [ ] **Task 5.1**: 添加日誌系統
  - 結構化日誌
  - 日誌級別
  - 日誌持久化

- [ ] **Task 5.2**: 添加監控指標
  - 任務完成率
  - 平均執行時間
  - 錯誤率

- [ ] **Task 5.3**: 添加配置管理
  - YAML 配置文件
  - 環境變量
  - 配置驗證

- [ ] **Task 5.4**: 添加 CLI 工具
  - 啟動/停止命令
  - 狀態查詢
  - 任務觸發

- [ ] **Task 5.5**: 文檔完善
  - API 文檔
  - 使用指南
  - 故障排查

**可交付成果**:
- `config/` - 配置文件目錄
- `cli.py` - CLI 工具
- `docs/simulate/05-usage-guide.md` - 使用指南
- `docs/simulate/06-troubleshooting.md` - 故障排查

---

## 測試計劃

### 單元測試

**目標**: 測試每個模組的功能

```python
# tests/test_grpc_client.py
def test_register_unit():
    """測試設備註冊"""

def test_login_unit():
    """測試設備登入"""

# tests/test_state_machine.py
def test_state_transitions():
    """測試狀態轉換"""

# tests/test_message_bus.py
def test_publish_subscribe():
    """測試發布訂閱"""
```

### 整合測試

**目標**: 測試模組間的交互

```python
# tests/test_station_robot_communication.py
def test_task_relay():
    """測試 Station 轉派任務給 Robot"""

def test_progress_reporting():
    """測試 Robot 進度回報給 Station"""
```

### 端到端測試

**測試場景**:

#### 場景 1: 正常流程
```
1. 啟動 Station 和 Robot
2. 觸發任務創建
3. Station 接收任務
4. Station 轉派給 Robot
5. Robot 執行任務
6. Robot 回報進度 (0% → 25% → 50% → 75% → 100%)
7. Station 聚合回報
8. 任務完成
```

#### 場景 2: Robot 離線
```
1. 啟動 Station
2. Robot 未啟動
3. Station 接收任務
4. Station 嘗試轉派給 Robot
5. 轉派失敗
6. Station 回報錯誤
7. 後續 Robot 上線後重試
```

#### 場景 3: 任務失敗
```
1. 正常啟動
2. Robot 執行任務過程中模擬失敗
3. Robot 回報失敗狀態
4. Station 接收失敗通知
5. Station 上報任務失敗
```

### 壓力測試

**測試指標**:
- 並發任務數: 10 / 50 / 100
- 任務完成率: > 99%
- 平均響應時間: < 5s
- 錯誤率: < 1%

---

## 可交付成果

### 代碼結構

```
k8s_onestaion/
├── simulate/
│   ├── common/
│   │   ├── __init__.py
│   │   ├── grpc_client.py          # gRPC 客戶端封裝
│   │   ├── state_machine.py        # 任務狀態機
│   │   ├── event_handler.py        # 事件處理器
│   │   └── message_bus.py          # 本地消息總線
│   │
│   ├── station/
│   │   ├── __init__.py
│   │   ├── station_simulator.py    # Station 主程式
│   │   └── task_manager.py         # Station 任務管理
│   │
│   ├── robot/
│   │   ├── __init__.py
│   │   ├── robot_simulator.py      # Robot 主程式
│   │   └── task_executor.py        # Robot 任務執行器
│   │
│   ├── config/
│   │   ├── station_config.yaml     # Station 配置
│   │   └── robot_config.yaml       # Robot 配置
│   │
│   ├── tests/
│   │   ├── test_grpc_client.py
│   │   ├── test_state_machine.py
│   │   ├── test_station.py
│   │   ├── test_robot.py
│   │   └── test_end_to_end.py
│   │
│   ├── main.py                      # 統一啟動腳本
│   ├── cli.py                       # CLI 工具
│   └── requirements.txt             # Python 依賴
│
├── docs/simulate/
│   ├── 00-implementation-plan.md    # 本文檔
│   ├── 01-grpc-client-guide.md      # gRPC 客戶端指南
│   ├── 02-station-simulator-guide.md # Station 模擬器指南
│   ├── 03-robot-simulator-guide.md   # Robot 模擬器指南
│   ├── 04-complete-loop-guide.md     # 完整閉環指南
│   ├── 05-usage-guide.md             # 使用指南
│   └── 06-troubleshooting.md         # 故障排查
│
└── proto_from_server/
    └── jarvis_iot.proto             # 服務器 proto 文件
```

### 文檔交付

1. **實作指南** - 每個階段的詳細實作說明
2. **API 文檔** - 所有類和方法的文檔
3. **使用手冊** - 如何啟動和使用模擬器
4. **測試報告** - 測試結果和性能指標
5. **故障排查** - 常見問題和解決方案

---

## 附錄

### 關鍵 API 參考

#### RSRegisterUnit

```python
request = jarvis_iot_pb2.RSRegisterUnitRequest(
    unit_sn="STZJGF4025030067",
    unit_type=2  # UNIT_TYPE_STATION
)
response = stub.RSRegisterUnit(request)
# response.unit_uuid, response.unit_uid
```

#### RSLoginUnit

```python
request = jarvis_iot_pb2.RSLoginUnitRequest(
    unit_uuid="d5ab9c4b-a483-4101-8026-b47237079d6e",
    unit_sn="STZJGF4025030067",
    unit_type=2,
    station_info=StationInfo(hostname="station5-067")
)
response = stub.RSLoginUnit(request)
```

#### StreamUnitControl

```python
def request_generator():
    # 初始心跳
    yield StreamUnitControlRequest(...)

for response in stub.StreamUnitControl(request_generator()):
    # 處理下發的任務
    if response.HasField('task'):
        handle_task(response.task)
```

#### StreamUnitEvents

```python
def event_generator():
    while True:
        event = create_event()
        yield StreamUnitEventsRequest(
            unit_uuid=unit_uuid,
            events=[event]
        )
        time.sleep(5)

stub.StreamUnitEvents(event_generator())
```

### 狀態定義

```python
TaskStatus = {
    'CREATED': 0,
    'ASSIGNED': 1,
    'REASSIGNED': 2,
    'EXECUTING': 3,
    'COMPLETED': 4,
    'FAILED': 5
}

RobotTaskStatus = {
    'IDLE': 'idle',
    'MOVING_TO_PICKUP': 'moving_to_pickup',
    'ARRIVED_AT_PICKUP': 'arrived_at_pickup',
    'PICKING_UP': 'picking_up',
    'MOVING_TO_DELIVERY': 'moving_to_delivery',
    'ARRIVED_AT_DELIVERY': 'arrived_at_delivery',
    'DELIVERING': 'delivering',
    'COMPLETED': 'completed',
    'FAILED': 'failed'
}
```

---

## 下一步行動

### 立即開始

1. **閱讀文檔** - 完整理解系統架構
2. **環境準備** - 安裝依賴，編譯 proto
3. **Phase 1** - 實現基礎的 gRPC 通信

### 每週檢查點

- **Week 1 結束**: 完成 gRPC 客戶端，可以註冊和登入
- **Week 2 結束**: Station 可以接收任務
- **Week 3 結束**: Robot 可以執行任務
- **Week 4 結束**: 完整閉環測試通過
- **Week 5 結束**: 文檔完善，可以交付

---

**創建者**: AI Assistant
**審核者**: 待定
**批准者**: 待定
**版本歷史**:
- v1.0 (2025-11-18): 初始版本，完整規劃
