# Station 模擬器可行性評估報告

**文檔版本**: v1.0
**創建時間**: 2025-12-02
**狀態**: ✅ 評估完成

---

## 目錄

1. [評估結論](#評估結論)
2. [詳細分析](#詳細分析)
3. [關鍵風險與挑戰](#關鍵風險與挑戰)
4. [實作路線圖](#實作路線圖)
5. [結論與建議](#結論與建議)

---

## 評估結論

### 總體結論：✅ 可行，但需要完成額外實作

| 通訊層 | 可行性 | 需要實作 | 複雜度 |
|-------|-------|---------|-------|
| **4G gRPC (與雲平台)** | ✅ 完全可行 | GrpcClient 完善 | 低 |
| **2.4G (與實體 Robot)** | ✅ 可行 (有硬體) | NearfieldHardwareInterface | 中 |
| **完整任務閉環** | ✅ 可行 | 整合 + 測試 | 中高 |

---

## 詳細分析

### 1. 雲平台通訊 (4G gRPC) - ✅ 已有完整設計

**現有設計**：
- `01-phase1-infrastructure.md` 定義了 `GrpcClient` 類
- 支援 `RSRegisterUnit`, `RSLoginUnit`, `StreamUnitControl`, `StreamUnitEvents`
- TLS 連接已驗證可行 (參考 `test_rs_login.py`)

**缺口**：
- `GrpcClient` 目前是骨架代碼，需要完成實作
- 需要選擇 proto 編譯方案並生成 Python 代碼

**預估工作量**：1-2 天

---

### 2. 2.4G 硬體通訊 - ⚠️ 需要硬體介面實作

**現有設計 (`02-2.4g-simulation-enhancement.md`)**：

```python
# 已定義的消息類型
class AIR_SAY_MESSAGE_TYPE(IntEnum):
    PASS_DOOR_ROBOT_SAY = 34    # 過門狀態
    STATION_ROBOT_SAY = 35      # Station-Robot 通訊
    CHARGE_ROBOT_SAY = 37       # 充電狀態
    PLANNING_ROBOT_SAY = 38     # 規劃狀態
```

**硬體抽象層 (`04-station-robot-verification.md`)**：

```python
class NearfieldHardwareInterface(ABC):
    def initialize(self) -> bool: ...
    def send_message(self, target_unit_sid: int, message: bytes) -> bool: ...
    def register_callback(self, callback: Callable[[int, bytes], None]): ...
    def get_connected_units(self) -> List[int]: ...
    def get_nearfield_status(self) -> bool: ...
```

**缺口**：
- 需要 2.4G 硬體模組的 API 文檔
- 需要實作 `RealNearfield` 類
- 需要了解實際的串口通訊協議

**預估工作量**：3-5 天 (取決於硬體文檔的完整性)

---

### 3. 任務閉環整合

**完整任務流程**：

```
雲平台 (jarvis-iot)
    ↓ StreamUnitControl (4G gRPC)
Station 模擬器
    ↓ send_message() (2.4G 硬體)
實體 Robot
    ↓ 執行任務
    ↓ 回報狀態 (2.4G)
Station 模擬器
    ↓ StreamUnitEvents (4G gRPC)
雲平台
```

**現有設計覆蓋情況**：

| 步驟 | 文檔覆蓋 | 代碼骨架 | 需要完成 |
|-----|---------|---------|---------|
| 接收雲平台任務 | ✅ | ✅ | 實作 |
| 解析任務數據 | ✅ | ⚠️ 部分 | 完善 |
| 2.4G 發送任務 | ✅ | ✅ 抽象層 | 實作硬體介面 |
| 接收 Robot 狀態 | ✅ | ✅ 抽象層 | 實作硬體介面 |
| 4G 上報 air_says | ✅ | ✅ | 實作 |
| 狀態機管理 | ✅ | ✅ | 測試 |

---

## 關鍵風險與挑戰

### 風險 1：2.4G 硬體協議未知 ⚠️ 高風險

**現狀**：文檔假設使用串口通訊，但實際協議未知

**緩解措施**：
- 需要獲取硬體團隊的技術文檔
- 或分析現有 Station 固件的通訊代碼

---

### 風險 2：Station 硬體環境 ⚠️ 中風險

**現狀**：模擬器設計為 Python 程式，需在 Station 硬體上運行

**緩解措施**：
- 確認 Station 硬體支援 Python 3.x 環境
- 確認 gRPC 相關依賴可安裝
- 考慮使用 Docker 容器化部署

---

### 風險 3：消息格式相容性 ⚠️ 中風險

**現狀**：`02-2.4g-simulation-enhancement.md` 定義的 SAirSay 格式基於 log 分析

**緩解措施**：
- 與實際 Robot 進行通訊測試
- 比對發送/接收的 Base64 編碼數據
- 必要時調整編碼邏輯

---

## 實作路線圖

```
Phase 1: 完成 GrpcClient (1-2 天)
├── 編譯 proto 文件
├── 實作 register/login
└── 實作 StreamUnitControl/Events

Phase 2: 實作 2.4G 硬體介面 (3-5 天)
├── 獲取硬體 API 文檔
├── 實作 RealNearfield 類
└── 測試串口通訊

Phase 3: 整合 Station 模擬器 (2-3 天)
├── 整合 GrpcClient + NearfieldHardware
├── 實作任務接收和轉發邏輯
└── 實作狀態上報

Phase 4: 驗證測試 (3-5 天)
├── 階段 1 測試 (純雲端)
├── 階段 2 測試 (雲端中轉)
└── 階段 3 測試 (2.4G 硬體)
```

### 甘特圖

```
Week 1          Week 2          Week 3
|----|----|----|----|----|----|----|----|
[Phase 1  ]
     [Phase 2        ]
               [Phase 3   ]
                    [Phase 4        ]
```

---

## 結論與建議

### ✅ 結論：技術上可行

Station 模擬器**可以**在有 2.4G 硬體的情況下與雲平台和實體 Robot 正常通訊，完成任務。

現有文檔已提供：
- 完整的架構設計
- 4G gRPC 通訊骨架
- 2.4G 消息格式定義
- 硬體抽象層介面
- 三階段驗證方案

---

### 📋 建議優先順序

| 優先級 | 任務 | 狀態 | 前置條件 |
|-------|-----|------|---------|
| 1 | 完成 GrpcClient 實作 | 立即可做 | 無 |
| 2 | 驗證雲平台通訊 (階段 1) | 立即可做 | 任務 1 |
| 3 | 獲取 2.4G 硬體 API 文檔 | 需要協調 | 硬體團隊 |
| 4 | 實作 RealNearfield 類 | 待文檔 | 任務 3 |
| 5 | 在 Station 硬體上測試 | 需要硬體 | 任務 4 |
| 6 | 端到端驗證 | 最終測試 | 所有任務 |

---

### ⚠️ 關鍵前提條件

#### 必須獲取

1. **2.4G 硬體 API 文檔**
   - 串口通訊協議
   - 消息格式定義
   - SDK 或示例代碼

2. **Station 硬體訪問**
   - 確認支援 Python 3.x
   - 確認可安裝 gRPC 依賴
   - 確認 2.4G 模組串口路徑

3. **測試用 Robot**
   - 需要一台實體 Robot 進行驗證
   - Robot 需在同一站點 (site_uid)

---

### 📊 可行性矩陣

| 場景 | 雲平台 | 2.4G 硬體 | 實體 Robot | 可行性 |
|-----|-------|----------|-----------|-------|
| 純軟體模擬 | ✅ | ❌ 模擬 | ❌ 模擬 | ✅ 可驗證邏輯 |
| 雲端驗證 | ✅ | ❌ | ✅ | ✅ 可觀察 Robot |
| 完整閉環 | ✅ | ✅ | ✅ | ✅ **目標場景** |

---

## 附錄

### A. 相關文檔索引

| 編號 | 文檔 | 說明 |
|-----|------|------|
| 00 | `00-implementation-plan.md` | 總體實作規劃 |
| 01 | `01-phase1-infrastructure.md` | 基礎設施實作 |
| 02 | `02-2.4g-simulation-enhancement.md` | 2.4G 模擬增強 |
| 04 | `04-station-robot-verification.md` | 驗證方案 |
| 05 | `05-feasibility-evaluation.md` | 本文檔 |

### B. 評估依據

本評估基於以下資料：

1. **設計文檔分析**
   - `simulate/` 目錄下所有規劃文檔
   - `proto_from_server/jarvis_iot.proto` 服務定義

2. **實際 Log 分析**
   - `clue/*1542.md` 系列 log 文件
   - Robot 心跳數據結構
   - air_says 消息格式

3. **已驗證的技術**
   - gRPC TLS 連接 (`test_rs_login.py`)
   - RSRegisterUnit / RSLoginUnit 調用

---

**創建者**: AI Assistant
**審核者**: 待定
**版本歷史**:
- v1.0 (2025-12-02): 初始評估報告
