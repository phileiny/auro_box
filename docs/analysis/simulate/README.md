# 模擬器實作文檔

## 📚 文檔索引

### 規劃文檔

| 文檔 | 說明 | 狀態 |
|------|------|------|
| [00-implementation_plan_v2.md](00-implementation_plan_v2.md) | 總體實作計劃 v2 (33 Steps) ⭐ 使用中 | 🚧 Phase 1 Step 6 |
| [00-implementation-plan.md](00-implementation-plan.md) | 總體實作計劃 v1 (已棄用) | ✅ 完成 |
| [01-phase1-infrastructure.md](01-phase1-infrastructure.md) | Phase 1: 基礎設施實作指南 | ✅ 完成 |
| [02-2.4g-simulation-enhancement.md](02-2.4g-simulation-enhancement.md) | 2.4G 通訊模擬增強方案 | ✅ 完成 |
| [04-station-robot-verification.md](04-station-robot-verification.md) | 模擬 Station 與實體 Robot 通訊驗證 | ✅ 完成 |
| [05-feasibility-evaluation.md](05-feasibility-evaluation.md) | 2.4G 硬體可行性評估報告 | ✅ 完成 |
| [06-csharp-implementation-plan.md](06-csharp-implementation-plan.md) | C# 實作計劃 ⭐NEW | ✅ 完成 |

### 未來文檔 (待創建)

- `03-phase2-station.md` - Phase 2: Station 模擬器實作
- `05-phase3-robot.md` - Phase 3: Robot 模擬器實作
- `06-phase4-integration.md` - Phase 4: 完整任務閉環
- `07-phase5-enhancement.md` - Phase 5: 優化和監控

---

## 🎯 項目目標

實作 **Station 模擬器** 和 **Robot 模擬器**，完成完整的任務閉環：

```
雲平台 (Jarvis IoT)
    ↓ 4G gRPC
Station 模擬器
    ↓ 2.4G 模擬 (本地消息總線)
Robot 模擬器
    ↓ 執行任務
完成並回報
```

---

## 📋 實作階段

### Phase 1: 基礎設施 ⚠️ 進行中

**目標**: 建立 gRPC 通信和認證機制

**關鍵組件**:
- ✅ Proto 文件處理 (3 種方案)
- ✅ GrpcClient 基礎類 (骨架)
- ✅ TaskStateMachine 狀態機 (骨架)
- ✅ LocalMessageBus 消息總線 (骨架)

**詳見**: [01-phase1-infrastructure.md](01-phase1-infrastructure.md)

### Phase 2: Station 模擬器 📅 待開始

**目標**: 實作 Station 模擬器

**關鍵功能**:
- 從雲平台接收任務 (StreamUnitControl)
- 分配任務給 Robot (via 2.4G)
- 收集 Robot 進度並回報雲平台 (StreamUnitEvents)

### Phase 3: Robot 模擬器 📅 待開始

**目標**: 實作 Robot 模擬器

**關鍵功能**:
- 從 Station 接收任務 (via 2.4G)
- 執行任務 (模擬移動、取貨、送貨)
- 回報進度給 Station

### Phase 4: 完整閉環 📅 待開始

**目標**: 整合 Station 和 Robot

**驗證**:
- 端到端任務流程
- 異常處理
- 性能優化

### Phase 5: 增強功能 📅 待開始

**目標**: 提升可用性

**增強項目**:
- 日誌和監控
- 配置管理
- CLI 工具
- 文檔完善

---

## 🏗️ 目錄結構

```
simulate/
├── common/                    # 共享組件
│   ├── grpc_client.py        # gRPC 客戶端封裝
│   ├── state_machine.py      # 任務狀態機
│   └── message_bus.py        # 本地消息總線 (2.4G 模擬)
│
├── station/                   # Station 模擬器
│   ├── station_simulator.py  # 主程序
│   └── task_manager.py       # 任務管理器
│
├── robot/                     # Robot 模擬器
│   ├── robot_simulator.py    # 主程序
│   └── task_executor.py      # 任務執行器
│
├── proto/                     # Proto 文件
│   ├── jarvis_iot_simple.proto  # 簡化版 proto
│   ├── jarvis_iot_pb2.py        # 生成的消息定義
│   └── jarvis_iot_pb2_grpc.py   # 生成的服務定義
│
└── tests/                     # 測試
    ├── test_phase1.py
    ├── test_station.py
    └── test_robot.py
```

---

## 🚀 快速開始

### 環境準備

```bash
# 安裝依賴
pip install grpcio grpcio-tools protobuf

# 進入項目目錄
cd D:\sides\k8s_onestaion
```

### Phase 1 - 測試基礎組件

**1. 編譯 Proto (選擇一種方案)**

```bash
# 方案 A: 簡化版 proto (推薦)
cd simulate/proto
python -m grpc_tools.protoc -I. \
  --python_out=. \
  --grpc_python_out=. \
  jarvis_iot_simple.proto
```

**2. 測試 gRPC 客戶端**

```bash
python simulate/common/grpc_client.py
```

**3. 測試狀態機**

```bash
python simulate/common/state_machine.py
```

**4. 測試消息總線**

```bash
python simulate/common/message_bus.py
```

---

## 📖 關鍵概念

### 1. 任務流程

```
雲平台                 Station                Robot
  |                      |                      |
  |-- 任務分配 --------->|                      |
  |   (StreamUnitControl)|                      |
  |                      |-- 轉發任務 -------->|
  |                      |   (2.4G 消息)       |
  |                      |                      |-- 執行
  |                      |<-- 進度回報 --------|
  |                      |   (2.4G 消息)       |
  |<-- 狀態更新 ---------|                      |
  |   (StreamUnitEvents) |                      |
```

### 2. 任務狀態

```
CREATED     - 雲平台創建
    ↓
ASSIGNED    - 分配給 Station
    ↓
REASSIGNED  - Station 分配給 Robot
    ↓
EXECUTING   - Robot 執行中
    ↓
COMPLETED   - 完成
或
FAILED      - 失敗
```

### 3. 通信方式

**4G gRPC (雲平台 ↔ Station/Robot)**:
- 註冊: `RSRegisterUnit`
- 登入: `RSLoginUnit`
- 接收控制: `StreamUnitControl` (雙向流)
- 發送事件: `StreamUnitEvents` (雙向流)

**2.4G 模擬 (Station ↔ Robot)**:
- 使用 `NearfieldBus` 模擬無線通信 (增強版)
- 基於主題的發布/訂閱模式
- 支持任務分配、進度回報
- ⭐ **新增**: 真實 `SAirSay` 消息格式 (Base64 編碼)
- ⭐ **新增**: `nearfield_status` 狀態追蹤
- ⭐ **新增**: 4G 上報時包含 `air_says` 陣列
- 詳見: [02-2.4g-simulation-enhancement.md](02-2.4g-simulation-enhancement.md)

---

## 🔍 參考文檔

### 項目相關

- `FINAL_SOLUTION.md` - gRPC 連接問題解決方案
- `docs/02-task-loop-technical-doc.md` - 任務閉環技術文檔
- `docs/03-api-reference.md` - API 參考

### Proto 文件

- `proto_from_server/jarvis_iot.proto` - 服務器真實 proto
- `proto_from_server/jarvis_iot.protoset` - 預編譯 protoset

### 測試腳本

- `test_rs_login.py` - RSLoginUnit 測試示例

---

## ⚠️ 注意事項

### Proto 編譯

**依賴問題**: 服務器的 proto 依賴 `google/api/annotations.proto` 和 `protoc-gen-swagger/options/annotations.proto`

**解決方案** (見 [01-phase1-infrastructure.md](01-phase1-infrastructure.md)):
1. 使用簡化版 proto (快速開始)
2. 下載依賴並完整編譯 (完整功能)
3. 動態加載 protoset (高級用法)

### gRPC 連接

**證書**: 不需要客戶端證書，只需 TLS 連接

```python
credentials = grpc.ssl_channel_credentials()  # 足夠了
```

**正確的 RPC 方法**:
- ✅ `RSRegisterUnit` - 註冊設備
- ✅ `RSLoginUnit` - 登入設備
- ❌ `LoginUnit` - 已棄用/不存在

---

## 📞 後續支持

### 下班前的狀態

**已完成**:
- ✅ 總體實作計劃 (5 階段)
- ✅ Phase 1 詳細指南 (含 3 種 proto 方案)
- ✅ 3 個基礎組件骨架代碼:
  - `grpc_client.py`
  - `state_machine.py`
  - `message_bus.py`

**待完成** (下次開始):
1. 選擇並執行 proto 編譯方案
2. 完善 `grpc_client.py` 實作 (連接註冊/登入方法)
3. 創建 Phase 1 整合測試
4. 驗證基礎功能正常

### 啟動清單

下次開始時執行:

```bash
# 1. 查看待辦事項
cat docs/simulate/README.md

# 2. 選擇 proto 方案並編譯
# (見 01-phase1-infrastructure.md 第 1 節)

# 3. 測試基礎組件
python simulate/common/state_machine.py
python simulate/common/message_bus.py

# 4. 完善並測試 grpc_client.py
python simulate/common/grpc_client.py
```

---

**文檔版本**: 1.0
**創建時間**: 2025-11-18
**狀態**: ✅ 文檔已整理 - 可以下班了！
**下次重點**: Proto 編譯 → 完善 gRPC 客戶端 → 整合測試
