# Station & Robot 模擬器實作規劃 (C# 版本)

**文檔版本**: v3.1
**創建時間**: 2025-11-18
**更新時間**: 2025-12-03
**開發語言**: C# (.NET 8.0)
**目標**: 完整實現 Station 和 Robot 模擬器，模擬任務閉環流程，包含 2.4G 通訊優化
**狀態**: ⚠️ 實作中（Phase 1 完成，Phase 2 規劃中）

---

## 📋 目錄

1. [總體目標](#總體目標)
2. [系統架構設計](#系統架構設計)
3. [技術棧](#技術棧)
4. [Stage 切分策略](#stage-切分策略)
5. [詳細 Stage 規劃](#詳細-stage-規劃)
6. [測試策略](#測試策略)
7. [可交付成果](#可交付成果)

---

## 總體目標

### 🎯 核心目標

使用 **C#** 實作兩個模擬程式，模擬真實的 Station 和 Robot 設備：

1. **Station 模擬器** (`StationSimulator`)
   - 模擬 Station 設備的完整行為
   - 通過 4G gRPC 與雲平台通信
   - 接收任務並轉派給 Robot（模擬 2.4G 本地通信）

2. **Robot 模擬器** (`RobotSimulator`)
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
3. **可測試**: 每個 RPC 動作都有對應的單元測試
4. **小步快跑**: 每個 Stage 控制在 4 小時內可完成
5. **可觀察**: 詳細的日誌和狀態輸出

---

## 系統架構設計

### 整體架構圖

```
┌─────────────────────────────────────────────────────────┐
│                    雲平台 (Aurotek Cloud)                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ jarvis-iot-v6│  │app-delivery  │  │ DAPR EventBus│  │
│  │              │  │   -rpc       │  │              │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼─────────┘
          │ 4G gRPC          │ 4G gRPC          │
          │ :443             │ :443             │
          │                  │                  │
    ┌─────▼──────┐      ┌────▼─────┐           │
    │  Station   │      │  Robot   │           │
    │ Simulator  │◄────►│ Simulator│           │
    │   (C#)     │ 2.4G │   (C#)   │           │
    └────────────┘ 本地  └──────────┘           │
         │         協作      │                  │
         │                   │                  │
         └───────────────────┴──────────────────┘
              模擬設備層 (本地運行)
```

### gRPC API 清單

#### jarvis-iot-v6 提供的 RPC

| RPC 方法 | 類型 | 用途 | 優先級 |
|---------|------|------|--------|
| `RSRegisterUnit` | Unary | 註冊設備 | 🔴 高 |
| `RSLoginUnit` | Unary | 登入設備 | 🔴 高 |
| `StreamUnitControl` | Server Streaming | 接收任務下發 | 🔴 高 |
| `StreamUnitEvents` | Client Streaming | 上報狀態事件 | 🔴 高 |
| `SyncMeteorEvents` | Unary | 高頻事件同步 | 🟡 中 |
| `QueryLinkOrders` | Unary | 查詢關聯訂單 | 🟢 低 |
| `GetDispatchUnitConfig` | Unary | 獲取配置 | 🟢 低 |

---

## 技術棧

### 開發技術

| 技術 | 版本 | 用途 |
|------|------|------|
| **.NET SDK** | 8.0 | 開發平台 |
| **C#** | 12 | 開發語言 |
| **Grpc.Net.Client** | 最新穩定版 | gRPC 客戶端 |
| **Google.Protobuf** | 最新穩定版 | Protobuf 序列化 |
| **Grpc.Tools** | 最新穩定版 | Proto 編譯工具 |
| **xUnit** | 最新穩定版 | 單元測試框架 |
| **Moq** | 最新穩定版 | Mock 框架 |
| **Microsoft.Extensions.Logging** | 8.0 | 日誌框架 |
| **Microsoft.Extensions.Configuration** | 8.0 | 配置管理 |

### 專案結構

```
YogoCloudSimulate/
├── YogoCloudSimulate.sln              # 解決方案
│
├── src/
│   ├── Common/                         # 共用類別庫
│   │   ├── Protos/                     # Proto 文件
│   │   │   └── jarvis_iot.proto
│   │   ├── GrpcClient.cs               # gRPC 客戶端基礎類
│   │   ├── StateMachine.cs             # 任務狀態機
│   │   └── MessageBus.cs               # 本地消息總線（2.4G 模擬）
│   │
│   ├── StationSimulator/               # Station 模擬器
│   │   ├── Program.cs
│   │   ├── StationSimulator.cs
│   │   └── TaskManager.cs
│   │
│   └── RobotSimulator/                 # Robot 模擬器
│       ├── Program.cs
│       ├── RobotSimulator.cs
│       └── TaskExecutor.cs
│
└── tests/
    ├── Common.Tests/                   # 共用模組測試
    │   ├── GrpcClientTests.cs
    │   ├── StateMachineTests.cs
    │   └── MessageBusTests.cs
    │
    ├── StationSimulator.Tests/         # Station 測試
    │   └── StationSimulatorTests.cs
    │
    ├── RobotSimulator.Tests/           # Robot 測試
    │   └── RobotSimulatorTests.cs
    │
    └── Integration.Tests/              # 整合測試
        └── EndToEndTests.cs
```

---

## Stage 切分策略

### 切分原則

1. **時間控制**: 每個 Stage 實作時間控制在 **4 小時內**
2. **獨立完整**: 每個 Stage 都有明確的輸入、輸出和驗收標準
3. **測試先行**: 每個 Stage 都包含對應的單元測試
4. **小步快跑**: 優先實作核心功能，逐步擴展

### Stage 組織

**總共 26 個 Stage**，分為 5 個 Phase：

- **Phase 1: 基礎設施** (Stage 1-6) - 建立 gRPC 通信基礎
- **Phase 2: Station 模擬器** (Stage 7-11) - 實作 Station 功能
- **Phase 3: Robot 模擬器** (Stage 12-16) - 實作 Robot 功能
- **Phase 4: 完整閉環** (Stage 17-18) - 整合測試
- **Phase 5: 2.4G 通訊優化與部署** (Stage 19-26) - 2.4G 增強、驗證、監控和文檔

---

## 詳細 Stage 規劃

### Phase 1: 基礎設施 (Stage 1-6)

**目標**: 建立 gRPC 通信和認證機制

---

#### **Stage 1: 專案初始化和 Proto 編譯** ⏱️ 2 小時 ✅ 已完成

**目標**: 建立 C# 專案結構並編譯 Proto 文件

**任務清單**:
- [x] 建立解決方案 `YogoCloudSimulate.sln`
- [x] 建立專案 `Common`、`StationSimulator`、`RobotSimulator`
- [x] 建立測試專案 `Common.Tests`、`Integration.Tests`
- [x] 安裝 NuGet 套件（Grpc.Net.Client、Google.Protobuf、Grpc.Tools）
- [x] 複製 `jarvis_iot.proto` 到 `Common/Protos/`
- [x] 配置 Proto 編譯（.csproj 設定）
- [x] 編譯並驗證生成的 C# 類別

**驗收標準**:
- ✅ 解決方案可正常編譯
- ✅ Proto 編譯成功，生成 `JarvisIot.cs` 和 `JarvisIotGrpc.cs`
- ✅ 可以 `using` 生成的命名空間

**交付成果**:
- 專案結構
- 編譯後的 Proto 類別
- `docs/implementation/stage-01-project-setup.md`

---

#### **Stage 2: gRPC 基礎連接** ⏱️ 3 小時 ✅ 已完成

**目標**: 實作基礎的 gRPC TLS 連接

**任務清單**:
- [x] 建立 `GrpcClientBase` 類別
- [x] 實作 `ConnectAsync()` 方法（TLS 連接到 `robot-rpc.aurotek.com:443`）
- [x] 實作連接狀態檢查
- [x] 實作錯誤處理和重試機制
- [x] 實作 `Dispose()` 資源清理
- [x] 撰寫單元測試 `GrpcClientBaseTests.cs`

**C# 實作範例**:
```csharp
/// <summary>
/// gRPC 客戶端基礎類別
/// </summary>
public class GrpcClientBase : IDisposable
{
    private readonly string _serverUrl;
    private GrpcChannel? _channel;
    private readonly ILogger<GrpcClientBase> _logger;

    /// <summary>
    /// 建立 gRPC 連接
    /// </summary>
    public async Task<bool> ConnectAsync(CancellationToken cancellationToken = default)
    {
        // 建立 SSL 憑證
        // 建立 gRPC Channel
        // 等待連接就緒
    }

    public void Dispose()
    {
        _channel?.Dispose();
    }
}
```

**單元測試**:
```csharp
public class GrpcClientBaseTests
{
    [Fact]
    public async Task ConnectAsync_ShouldEstablishConnection()
    {
        // Arrange
        var client = new GrpcClientBase("robot-rpc.aurotek.com:443");

        // Act
        var result = await client.ConnectAsync();

        // Assert
        Assert.True(result);
    }
}
```

**驗收標準**:
- ✅ 可成功連接到 `robot-rpc.aurotek.com:443`
- ✅ 連接失敗時能正確處理錯誤
- ✅ 單元測試通過率 100%

**交付成果**:
- `GrpcClientBase.cs`
- `GrpcClientBaseTests.cs`
- `docs/implementation/stage-02-grpc-connection.md`

---

#### **Stage 3: RSRegisterUnit 註冊功能** ⏱️ 3 小時 ✅ 已完成

**目標**: 實作設備註冊功能

**任務清單**:
- [x] 建立 `UnitRegistration` 類別
- [x] 實作 `RegisterUnitAsync()` 方法
- [x] 處理註冊請求和響應
- [x] 保存 `unit_uuid` 和 `unit_uid`
- [x] 處理註冊錯誤（已註冊、參數錯誤等）
- [x] 撰寫單元測試 `UnitRegistrationTests.cs`

**C# 實作範例**:
```csharp
/// <summary>
/// 設備註冊結果
/// </summary>
public record UnitRegistrationResult
{
    public bool Success { get; init; }
    public string? UnitUuid { get; init; }
    public long UnitUid { get; init; }
    public string? Hostname { get; init; }
    public string? ErrorMessage { get; init; }
}

/// <summary>
/// 設備註冊服務
/// </summary>
public class UnitRegistration
{
    /// <summary>
    /// 註冊設備
    /// </summary>
    /// <param name="unitSn">設備序號</param>
    /// <param name="unitType">設備類型 (1=Robot, 2=Station)</param>
    public async Task<UnitRegistrationResult> RegisterUnitAsync(
        string unitSn,
        UnitType unitType,
        CancellationToken cancellationToken = default)
    {
        // 建立請求
        // 調用 RSRegisterUnit RPC
        // 處理響應
        // 返回結果
    }
}
```

**單元測試**:
```csharp
public class UnitRegistrationTests
{
    [Fact]
    public async Task RegisterUnitAsync_WithValidData_ShouldReturnSuccess()
    {
        // Arrange
        var registration = new UnitRegistration(mockClient.Object);

        // Act
        var result = await registration.RegisterUnitAsync("TEST001", UnitType.Station);

        // Assert
        Assert.True(result.Success);
        Assert.NotNull(result.UnitUuid);
    }

    [Fact]
    public async Task RegisterUnitAsync_WithInvalidSN_ShouldReturnError()
    {
        // 測試錯誤處理
    }
}
```

**驗收標準**:
- ✅ 可成功註冊設備
- ✅ 正確保存 `unit_uuid` 和 `unit_uid`
- ✅ 錯誤情況能正確處理
- ✅ 單元測試通過率 100%

**交付成果**:
- `UnitRegistration.cs`
- `UnitRegistrationTests.cs`
- `docs/implementation/stage-03-register-unit.md`

---

#### **Stage 4: RSLoginUnit 登入功能** ⏱️ 3 小時 ✅ 已完成

**目標**: 實作設備登入功能

**任務清單**:
- [x] 建立 `UnitLogin` 類別
- [x] 實作 `LoginUnitAsync()` 方法
- [x] 處理登入請求和響應
- [x] 保存 token
- [x] 處理登入錯誤（未註冊、認證失敗等）
- [x] 撰寫單元測試 `UnitLoginTests.cs`

**C# 實作範例**:
```csharp
/// <summary>
/// 設備登入結果
/// </summary>
public record UnitLoginResult
{
    public bool Success { get; init; }
    public string? Token { get; init; }
    public string? ErrorMessage { get; init; }
}

/// <summary>
/// 設備登入服務
/// </summary>
public class UnitLogin
{
    /// <summary>
    /// 登入設備
    /// </summary>
    public async Task<UnitLoginResult> LoginUnitAsync(
        string unitUuid,
        string unitSn,
        UnitType unitType,
        CancellationToken cancellationToken = default)
    {
        // 建立請求
        // 調用 RSLoginUnit RPC
        // 處理響應
        // 返回結果
    }
}
```

**單元測試**:
```csharp
public class UnitLoginTests
{
    [Fact]
    public async Task LoginUnitAsync_WithValidCredentials_ShouldReturnSuccess()
    {
        // Arrange & Act & Assert
    }

    [Fact]
    public async Task LoginUnitAsync_WithInvalidUuid_ShouldReturnError()
    {
        // 測試錯誤處理
    }
}
```

**驗收標準**:
- ✅ 可成功登入設備
- ✅ 正確保存 token
- ✅ 錯誤情況能正確處理
- ✅ 單元測試通過率 100%

**交付成果**:
- `UnitLogin.cs`
- `UnitLoginTests.cs`
- `docs/implementation/stage-04-login-unit.md`

---

#### **Stage 5: 任務狀態機** ⏱️ 3 小時 ✅ 已完成

**目標**: 實作任務狀態機

**任務清單**:
- [x] 定義 `TaskState` 枚舉
- [x] 建立 `TaskStateMachine` 類別
- [x] 實作狀態轉換邏輯
- [x] 實作狀態轉換驗證
- [x] 實作狀態變更事件
- [x] 撰寫單元測試 `TaskStateMachineTests.cs`

**C# 實作範例**:
```csharp
/// <summary>
/// 任務狀態
/// </summary>
public enum TaskState
{
    Created,      // 已創建
    Assigned,     // 已分配給 Station
    Reassigned,   // Station 已分配給 Robot
    Executing,    // Robot 執行中
    Completed,    // 已完成
    Failed        // 失敗
}

/// <summary>
/// 任務狀態機
/// </summary>
public class TaskStateMachine
{
    private TaskState _currentState;
    private readonly Dictionary<TaskState, List<TaskState>> _allowedTransitions;

    /// <summary>
    /// 狀態轉換
    /// </summary>
    public bool TryTransition(TaskState newState, out string? errorMessage)
    {
        // 檢查轉換是否合法
        // 執行轉換
        // 觸發事件
    }

    /// <summary>
    /// 狀態變更事件
    /// </summary>
    public event EventHandler<StateChangedEventArgs>? StateChanged;
}
```

**單元測試**:
```csharp
public class TaskStateMachineTests
{
    [Fact]
    public void TryTransition_FromCreatedToAssigned_ShouldSucceed()
    {
        // Arrange
        var sm = new TaskStateMachine();

        // Act
        var result = sm.TryTransition(TaskState.Assigned, out _);

        // Assert
        Assert.True(result);
        Assert.Equal(TaskState.Assigned, sm.CurrentState);
    }

    [Fact]
    public void TryTransition_InvalidTransition_ShouldFail()
    {
        // 測試無效的狀態轉換
    }
}
```

**驗收標準**:
- ✅ 所有合法的狀態轉換都能成功
- ✅ 無效的狀態轉換會被拒絕
- ✅ 狀態變更事件正常觸發
- ✅ 單元測試通過率 100%

**交付成果**:
- `TaskStateMachine.cs`
- `TaskStateMachineTests.cs`
- `docs/implementation/stage-05-state-machine.md`

---

#### **Stage 6: 本地消息總線（2.4G 模擬）** ⏱️ 4 小時 ✅ 已完成

**目標**: 實作本地消息總線，模擬 2.4G 通信

**任務清單**:
- [x] 建立 `LocalMessageBus` 類別
- [x] 實作發布/訂閱機制
- [x] 實作主題（Topic）管理
- [x] 實作消息佇列
- [x] 實作線程安全
- [x] 撰寫單元測試 `LocalMessageBusTests.cs`

**C# 實作範例**:
```csharp
/// <summary>
/// 本地消息
/// </summary>
public record LocalMessage
{
    public string Topic { get; init; } = string.Empty;
    public string SenderId { get; init; } = string.Empty;
    public string ReceiverId { get; init; } = string.Empty;
    public object Payload { get; init; } = new();
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
}

/// <summary>
/// 本地消息總線（模擬 2.4G 通信）
/// </summary>
public class LocalMessageBus : IDisposable
{
    private readonly ConcurrentDictionary<string, List<Action<LocalMessage>>> _subscribers;
    private readonly Channel<LocalMessage> _messageQueue;

    /// <summary>
    /// 訂閱主題
    /// </summary>
    public void Subscribe(string topic, string subscriberId, Action<LocalMessage> handler)
    {
        // 添加訂閱者
    }

    /// <summary>
    /// 發布消息
    /// </summary>
    public async Task PublishAsync(LocalMessage message, CancellationToken cancellationToken = default)
    {
        // 將消息放入佇列
        // 異步處理消息分發
    }

    /// <summary>
    /// 啟動消息總線
    /// </summary>
    public Task StartAsync(CancellationToken cancellationToken = default)
    {
        // 啟動消息處理循環
    }
}
```

**單元測試**:
```csharp
public class LocalMessageBusTests
{
    [Fact]
    public async Task PublishAsync_ShouldDeliverToSubscribers()
    {
        // Arrange
        var bus = new LocalMessageBus();
        var received = false;

        bus.Subscribe("test.topic", "subscriber1", msg => received = true);
        await bus.StartAsync();

        // Act
        await bus.PublishAsync(new LocalMessage
        {
            Topic = "test.topic",
            SenderId = "sender1",
            ReceiverId = "subscriber1"
        });

        await Task.Delay(100); // 等待消息處理

        // Assert
        Assert.True(received);
    }

    [Fact]
    public async Task Subscribe_MultipleSubscribers_ShouldAllReceive()
    {
        // 測試多訂閱者
    }
}
```

**驗收標準**:
- ✅ 消息能正確發布和訂閱
- ✅ 支持多個訂閱者
- ✅ 線程安全
- ✅ 單元測試通過率 100%

**交付成果**:
- `LocalMessageBus.cs`
- `LocalMessageBusTests.cs`
- `docs/implementation/stage-06-message-bus.md`

---

### Phase 2: Station 模擬器 (Stage 7-11)

**目標**: 實作 Station 基本功能

---

#### **Stage 7: StreamUnitControl 接收任務** ⏱️ 4 小時 📋 規劃完成

**目標**: 實作接收雲端任務的流式 RPC

**規劃文檔**: `docs/planning/phase-2-stage-7-stream-unit-control.md` ✅

**任務清單**:
- [ ] 建立 `UnitControlStream` 類別
- [ ] 實作 `StreamUnitControl` 客戶端流式調用
- [ ] 實作心跳發送
- [ ] 實作任務接收回調
- [ ] 處理流式連接錯誤和重連
- [ ] 撰寫單元測試 `UnitControlStreamTests.cs`

**驗收標準**:
- ✅ 可建立流式連接
- ✅ 能接收任務下發
- ✅ 心跳正常發送
- ✅ 單元測試通過率 100%

**交付成果**:
- `UnitControlStream.cs`
- `UnitControlStreamTests.cs`
- `docs/implementation/stage-07-control-stream.md`

---

#### **Stage 8: StreamUnitEvents 上報事件** ⏱️ 4 小時 📋 規劃完成

**目標**: 實作上報事件的流式 RPC

**規劃文檔**: `docs/planning/phase-2-stage-8-stream-unit-events.md` ✅

**任務清單**:
- [ ] 建立 `UnitEventStream` 類別
- [ ] 實作 `StreamUnitEvents` 客戶端流式調用
- [ ] 實作事件發送佇列
- [ ] 實作批次發送機制
- [ ] 處理流式連接錯誤和重連
- [ ] 撰寫單元測試 `UnitEventStreamTests.cs`

**驗收標準**:
- ✅ 可建立流式連接
- ✅ 能上報事件
- ✅ 批次發送正常工作
- ✅ 單元測試通過率 100%

**交付成果**:
- `UnitEventStream.cs`
- `UnitEventStreamTests.cs`
- `docs/implementation/stage-08-event-stream.md`

---

#### **Stage 9: Station 主程式框架** ⏱️ 3 小時

**目標**: 建立 Station 模擬器主程式

**任務清單**:
- [ ] 建立 `StationSimulator` 類別
- [ ] 整合註冊、登入功能
- [ ] 整合控制流和事件流
- [ ] 實作啟動和停止流程
- [ ] 實作配置管理
- [ ] 撰寫單元測試 `StationSimulatorTests.cs`

**驗收標準**:
- ✅ Station 可正常啟動和停止
- ✅ 註冊登入流程完整
- ✅ 控制流和事件流正常工作
- ✅ 單元測試通過率 100%

**交付成果**:
- `StationSimulator.cs`
- `appsettings.json`
- `StationSimulatorTests.cs`
- `docs/implementation/stage-09-station-framework.md`

---

#### **Stage 10: Station 任務接收和記錄** ⏱️ 3 小時

**目標**: 實作任務接收和記錄功能

**任務清單**:
- [ ] 實作任務接收回調
- [ ] 實作任務資料解析
- [ ] 實作任務佇列管理
- [ ] 實作任務日誌記錄
- [ ] 撰寫單元測試

**驗收標準**:
- ✅ 能接收並解析任務
- ✅ 任務正確加入佇列
- ✅ 日誌完整記錄
- ✅ 單元測試通過率 100%

**交付成果**:
- 更新 `StationSimulator.cs`
- `TaskQueue.cs`
- 測試程式碼
- `docs/implementation/stage-10-task-reception.md`

---

#### **Stage 11: Station 任務轉派** ⏱️ 4 小時

**目標**: 實作任務轉派給 Robot 的功能

**任務清單**:
- [ ] 實作任務轉派邏輯
- [ ] 通過本地消息總線發送任務
- [ ] 實作轉派狀態追蹤
- [ ] 實作轉派失敗處理
- [ ] 撰寫單元測試

**驗收標準**:
- ✅ 能將任務轉派給 Robot
- ✅ 轉派狀態正確追蹤
- ✅ 失敗能正確處理
- ✅ 單元測試通過率 100%

**交付成果**:
- 更新 `StationSimulator.cs`
- `TaskDispatcher.cs`
- 測試程式碼
- `docs/implementation/stage-11-task-dispatch.md`

---

### Phase 3: Robot 模擬器 (Stage 12-16)

**目標**: 實作 Robot 基本功能

---

#### **Stage 12: Robot 主程式框架** ⏱️ 3 小時

**目標**: 建立 Robot 模擬器主程式

**任務清單**:
- [ ] 建立 `RobotSimulator` 類別
- [ ] 整合註冊、登入功能
- [ ] 整合事件流
- [ ] 訂閱本地消息總線
- [ ] 實作啟動和停止流程
- [ ] 撰寫單元測試

**驗收標準**:
- ✅ Robot 可正常啟動和停止
- ✅ 註冊登入流程完整
- ✅ 能接收本地消息
- ✅ 單元測試通過率 100%

**交付成果**:
- `RobotSimulator.cs`
- `appsettings.json`
- `RobotSimulatorTests.cs`
- `docs/implementation/stage-12-robot-framework.md`

---

#### **Stage 13: Robot 接收 Station 任務** ⏱️ 3 小時

**目標**: 實作從 Station 接收任務

**任務清單**:
- [ ] 實作本地消息訂閱
- [ ] 實作任務接收回調
- [ ] 實作任務資料解析
- [ ] 實作任務確認回覆
- [ ] 撰寫單元測試

**驗收標準**:
- ✅ 能從 Station 接收任務
- ✅ 任務正確解析
- ✅ 確認回覆正常發送
- ✅ 單元測試通過率 100%

**交付成果**:
- 更新 `RobotSimulator.cs`
- 測試程式碼
- `docs/implementation/stage-13-task-reception.md`

---

#### **Stage 14: 任務執行器** ⏱️ 4 小時

**目標**: 實作任務執行器

**任務清單**:
- [ ] 建立 `TaskExecutor` 類別
- [ ] 實作任務執行流程（移動、取貨、送貨）
- [ ] 實作進度計算
- [ ] 實作模擬延遲
- [ ] 撰寫單元測試

**驗收標準**:
- ✅ 任務執行流程完整
- ✅ 進度計算正確
- ✅ 模擬時間合理
- ✅ 單元測試通過率 100%

**交付成果**:
- `TaskExecutor.cs`
- `TaskExecutorTests.cs`
- `docs/implementation/stage-14-task-executor.md`

---

#### **Stage 15: Robot 進度回報** ⏱️ 3 小時

**目標**: 實作進度回報功能

**任務清單**:
- [ ] 實作進度事件生成
- [ ] 通過事件流上報進度
- [ ] 實作高頻同步（SyncMeteorEvents）
- [ ] 實作進度批次發送
- [ ] 撰寫單元測試

**驗收標準**:
- ✅ 進度正確上報
- ✅ 高頻同步正常工作
- ✅ 批次發送正常
- ✅ 單元測試通過率 100%

**交付成果**:
- 更新 `RobotSimulator.cs`
- `ProgressReporter.cs`
- 測試程式碼
- `docs/implementation/stage-15-progress-report.md`

---

#### **Stage 16: Robot 任務完成處理** ⏱️ 3 小時

**目標**: 實作任務完成和異常處理

**任務清單**:
- [ ] 實作任務完成通知
- [ ] 實作任務失敗處理
- [ ] 實作異常狀態上報
- [ ] 實作重試機制
- [ ] 撰寫單元測試

**驗收標準**:
- ✅ 完成通知正確發送
- ✅ 失敗能正確處理
- ✅ 異常狀態正確上報
- ✅ 單元測試通過率 100%

**交付成果**:
- 更新 `RobotSimulator.cs`
- 測試程式碼
- `docs/implementation/stage-16-task-completion.md`

---

### Phase 4: 完整閉環 (Stage 17-18)

**目標**: 整合測試和優化

---

#### **Stage 17: Station-Robot 整合測試** ⏱️ 4 小時

**目標**: 整合 Station 和 Robot，測試完整流程

**任務清單**:
- [ ] 同時啟動 Station 和 Robot
- [ ] 測試任務完整流程
- [ ] 測試異常場景（Robot 離線、任務失敗）
- [ ] 撰寫整合測試 `EndToEndTests.cs`
- [ ] 記錄測試結果

**驗收標準**:
- ✅ 完整任務流程能正常執行
- ✅ 異常場景能正確處理
- ✅ 整合測試通過率 100%

**交付成果**:
- `EndToEndTests.cs`
- 測試報告
- `docs/validation/stage-17-integration-test.md`

---

#### **Stage 18: 性能測試和優化** ⏱️ 4 小時

**目標**: 性能測試和優化

**任務清單**:
- [ ] 壓力測試（10/50/100 並發任務）
- [ ] 性能指標收集（延遲、吞吐量、記憶體）
- [ ] 識別性能瓶頸
- [ ] 優化關鍵路徑
- [ ] 記錄優化結果

**驗收標準**:
- ✅ 通過壓力測試
- ✅ 性能指標符合預期
- ✅ 無明顯性能瓶頸

**交付成果**:
- 性能測試報告
- 優化建議
- `docs/validation/stage-18-performance-test.md`

---

### Phase 5: 2.4G 通訊優化與部署 (Stage 19-26)

**目標**: 增強 2.4G 通訊模擬真實性、完成驗證、監控和文檔

**背景說明**:
根據實際 Log 分析（參考 `02-2.4g-simulation-enhancement.md`、`03-2.4g-clues.md`），
2.4G 通訊數據會透過 Robot 4G 上報傳到雲平台。為了更真實地模擬系統行為，
需要增強 2.4G 模擬模組，使其符合實際的消息格式和上報機制。

---

#### **Stage 19: 2.4G 消息結構和類型定義** ⏱️ 3 小時

**目標**: 定義符合真實系統的 2.4G 消息結構

**任務清單**:
- [ ] 定義 `AirSayMessageType` 枚舉（PASS_DOOR_ROBOT_SAY=34, STATION_ROBOT_SAY=35, CHARGE_ROBOT_SAY=37, PLANNING_ROBOT_SAY=38）
- [ ] 實作 `SAirSay` 類別（msg_type, say, created_at）
- [ ] 實作 `Position` 類別（map_sid, zone_id, x, y, theta）
- [ ] 實作 `PassDoorPosition` 枚舉（Unknown, Outside, Inside, Passing）
- [ ] 實作 `UnitStatus` 類別（unit_enabled, cellular_status, nearfield_status）
- [ ] 實作 `AirMessageHeader` 類別（message_type, unit_sid, version）
- [ ] 實作 `PassDoorRobotSay` 類別
- [ ] 撰寫單元測試

**C# 實作範例**:
```csharp
/// <summary>
/// 2.4G 空中消息類型
/// </summary>
public enum AirSayMessageType
{
    PassDoorRobotSay = 34,    // 過門狀態
    StationRobotSay = 35,     // Station-Robot 通訊
    ChargeRobotSay = 37,      // 充電狀態
    PlanningRobotSay = 38     // 規劃狀態
}

/// <summary>
/// 2.4G 空中消息
/// </summary>
public record SAirSay
{
    public AirSayMessageType MsgType { get; init; }
    public string Say { get; init; } = string.Empty;  // Base64 編碼的消息內容
    public long CreatedAt { get; init; }              // Unix timestamp (毫秒)

    public static SAirSay Create(AirSayMessageType msgType, byte[] payload)
    {
        return new SAirSay
        {
            MsgType = msgType,
            Say = Convert.ToBase64String(payload),
            CreatedAt = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
        };
    }
}

/// <summary>
/// 位置資訊
/// </summary>
public record Position
{
    public int MapSid { get; init; }
    public int ZoneId { get; init; }
    public int X { get; init; }
    public int Y { get; init; }
    public int Theta { get; init; }
}
```

**驗收標準**:
- ✅ 所有 2.4G 消息類型都已定義
- ✅ 消息結構符合實際 Log 中的格式
- ✅ Base64 編碼/解碼正常工作
- ✅ 單元測試通過率 100%

**交付成果**:
- `AirSayTypes.cs`
- `AirSayTypesTests.cs`
- `docs/implementation/stage-19-air-say-types.md`

---

#### **Stage 20: AirSayGenerator 消息生成器** ⏱️ 3 小時

**目標**: 實作 2.4G 消息生成器

**任務清單**:
- [ ] 建立 `AirSayGenerator` 類別
- [ ] 實作 `GenerateStationRobotSay()` 方法
- [ ] 實作 `GeneratePassDoorRobotSay()` 方法
- [ ] 實作 `GenerateChargeRobotSay()` 方法
- [ ] 實作 `GeneratePlanningRobotSay()` 方法
- [ ] 實作消息編碼邏輯（struct pack 格式）
- [ ] 撰寫單元測試

**C# 實作範例**:
```csharp
/// <summary>
/// 2.4G 空中消息生成器
/// </summary>
public class AirSayGenerator
{
    private readonly int _unitSid;
    private int _version;

    public AirSayGenerator(int unitSid)
    {
        _unitSid = unitSid;
        _version = 0;
    }

    /// <summary>
    /// 生成 Station-Robot 通訊消息
    /// </summary>
    public SAirSay GenerateStationRobotSay(Position position, int destinationPoi)
    {
        _version++;
        var payload = EncodeStationRobotSay(position, destinationPoi, _version);
        return SAirSay.Create(AirSayMessageType.StationRobotSay, payload);
    }

    /// <summary>
    /// 生成過門狀態消息
    /// </summary>
    public SAirSay GeneratePassDoorRobotSay(
        Position position,
        PassDoorPosition passDoorPosition,
        bool nearfieldStatus)
    {
        _version++;
        var payload = EncodePassDoorRobotSay(position, passDoorPosition, nearfieldStatus, _version);
        return SAirSay.Create(AirSayMessageType.PassDoorRobotSay, payload);
    }

    /// <summary>
    /// 生成充電狀態消息
    /// </summary>
    public SAirSay GenerateChargeRobotSay(int batterySoc, bool isCharging)
    {
        var payload = EncodeChargeRobotSay(batterySoc, isCharging);
        return SAirSay.Create(AirSayMessageType.ChargeRobotSay, payload);
    }

    private byte[] EncodeStationRobotSay(Position position, int destinationPoi, int version)
    {
        using var ms = new MemoryStream();
        using var writer = new BinaryWriter(ms);
        writer.Write((byte)35);           // message_type
        writer.Write((ushort)_unitSid);   // unit_sid
        writer.Write((ushort)version);    // version
        writer.Write((ushort)position.X);
        writer.Write((ushort)position.Y);
        writer.Write((ushort)position.Theta);
        writer.Write(destinationPoi);     // destination_poi
        return ms.ToArray();
    }

    // ... 其他編碼方法
}
```

**驗收標準**:
- ✅ 各類型消息都能正確生成
- ✅ 編碼格式符合真實協議
- ✅ 版本號正確遞增
- ✅ 單元測試通過率 100%

**交付成果**:
- `AirSayGenerator.cs`
- `AirSayGeneratorTests.cs`
- `docs/implementation/stage-20-air-say-generator.md`

---

#### **Stage 21: NearfieldBus 增強實作** ⏱️ 4 小時

**目標**: 替換原有的 LocalMessageBus，實作增強版 2.4G 近場通訊總線

**任務清單**:
- [ ] 建立 `NearfieldBus` 類別（替換 `LocalMessageBus`）
- [ ] 實作設備註冊和連線管理
- [ ] 實作 `nearfield_status` 狀態追蹤
- [ ] 實作消息緩存（供 4G 上報使用）
- [ ] 實作 `GetAirSaysForUpload()` 方法
- [ ] 實作線程安全的發布/訂閱機制
- [ ] 撰寫單元測試

**C# 實作範例**:
```csharp
/// <summary>
/// 模擬 2.4G 近場通訊總線
///
/// 特點：
/// 1. 模擬真實的 2.4G 訊息交換
/// 2. 維護 nearfield_status 狀態
/// 3. 生成符合真實格式的 air_says
/// </summary>
public class NearfieldBus : IDisposable
{
    private readonly ConcurrentDictionary<string, List<Action<SAirSay, object>>> _subscribers;
    private readonly Channel<NearfieldMessage> _messageQueue;

    // 2.4G 狀態
    private bool _nearfieldStatus;
    private readonly ConcurrentDictionary<int, bool> _connectedUnits;

    // 消息生成器
    private readonly ConcurrentDictionary<int, AirSayGenerator> _airSayGenerators;

    // 消息歷史 (用於上報)
    private readonly ConcurrentDictionary<int, ConcurrentQueue<SAirSay>> _airSaysBuffer;

    /// <summary>
    /// 註冊設備
    /// </summary>
    public void RegisterUnit(int unitSid)
    {
        _connectedUnits[unitSid] = false;
        _airSayGenerators[unitSid] = new AirSayGenerator(unitSid);
        _airSaysBuffer[unitSid] = new ConcurrentQueue<SAirSay>();
    }

    /// <summary>
    /// 設備連線 (模擬 2.4G 握手成功)
    /// </summary>
    public void ConnectUnit(int unitSid)
    {
        _connectedUnits[unitSid] = true;
        _nearfieldStatus = true;
        _logger.LogInformation("[2.4G] Unit {UnitSid} connected, nearfield_status=True", unitSid);
    }

    /// <summary>
    /// 獲取待上報的 air_says (供 4G 心跳使用)
    /// </summary>
    public IReadOnlyList<SAirSay> GetAirSaysForUpload(int unitSid)
    {
        var messages = new List<SAirSay>();
        if (_airSaysBuffer.TryGetValue(unitSid, out var buffer))
        {
            while (buffer.TryDequeue(out var msg))
            {
                messages.Add(msg);
            }
        }
        return messages;
    }

    /// <summary>
    /// 獲取 2.4G 連線狀態
    /// </summary>
    public bool GetNearfieldStatus() => _nearfieldStatus;
}
```

**驗收標準**:
- ✅ 消息能正確發布和訂閱
- ✅ `nearfield_status` 狀態正確追蹤
- ✅ 消息緩存供 4G 上報使用
- ✅ 線程安全
- ✅ 單元測試通過率 100%

**交付成果**:
- `NearfieldBus.cs`（替換 `LocalMessageBus.cs`）
- `NearfieldBusTests.cs`
- `docs/implementation/stage-21-nearfield-bus.md`

---

#### **Stage 22: Robot 模擬器 2.4G 優化** ⏱️ 4 小時

**目標**: 增強 Robot 模擬器，支援真實 2.4G 消息格式

**任務清單**:
- [ ] 整合 `NearfieldBus` 到 `RobotSimulator`
- [ ] 實作進入 Station 時的 2.4G 握手邏輯
- [ ] 實作離開 Station 時的斷線邏輯
- [ ] 修改心跳循環，包含 `air_says` 上報
- [ ] 實作 `_buildRobotSay()` 方法
- [ ] 更新狀態機配合空間狀態（INSIDE_STATION, PLAIN）
- [ ] 撰寫單元測試

**C# 實作範例**:
```csharp
public class RobotSimulator
{
    private readonly NearfieldBus _nearfieldBus;
    private Position _position;
    private SpaceState _spaceState = SpaceState.Plain;
    private PurposeState _purposeState = PurposeState.Idle;

    /// <summary>
    /// 進入 Station (觸發 2.4G 握手)
    /// </summary>
    public async Task EnterStationAsync(int stationSid)
    {
        // 模擬 2.4G 握手
        _nearfieldBus.ConnectUnit(_unitSid);

        // 更新狀態
        _spaceState = SpaceState.InsideStation;
        _purposeState = PurposeState.WaitingStation;

        // 發送過門消息
        await _nearfieldBus.SendMessageAsync(
            fromUnit: _unitSid,
            toUnit: stationSid,
            msgType: AirSayMessageType.PassDoorRobotSay,
            payload: new { Position = _position, PassDoorPosition = PassDoorPosition.Inside }
        );
    }

    /// <summary>
    /// 4G 心跳循環 (包含 2.4G 消息上報)
    /// </summary>
    private async Task HeartbeatLoopAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            // 收集 2.4G 消息
            var airSays = _nearfieldBus.GetAirSaysForUpload(_unitSid);
            var nearfieldStatus = _nearfieldBus.GetNearfieldStatus();

            // 構建心跳數據
            var pingData = new RobotPingData
            {
                RobotSay = BuildRobotSay(nearfieldStatus),
                AirSays = airSays.ToList(),
                Heartbeat = BuildHeartbeat(),
                NearfieldStatus = nearfieldStatus
            };

            // 通過 4G 上報
            await _grpcClient.SendMeteorEventsAsync(pingData);

            await Task.Delay(1000, cancellationToken);
        }
    }
}
```

**驗收標準**:
- ✅ 進入/離開 Station 時正確觸發 2.4G 握手/斷線
- ✅ `nearfield_status` 正確反映連線狀態
- ✅ 心跳數據包含 `air_says` 陣列
- ✅ 單元測試通過率 100%

**交付成果**:
- 更新 `RobotSimulator.cs`
- `RobotSimulatorNearfieldTests.cs`
- `docs/implementation/stage-22-robot-nearfield.md`

---

#### **Stage 23: Station 模擬器 2.4G 優化** ⏱️ 4 小時

**目標**: 增強 Station 模擬器，支援 2.4G 消息處理

**任務清單**:
- [ ] 整合 `NearfieldBus` 到 `StationSimulator`
- [ ] 實作接收 Robot 2.4G 消息的回調
- [ ] 實作發送任務給 Robot（透過 2.4G）
- [ ] 實作 Robot 連線/斷線事件處理
- [ ] 實作 2.4G 硬體抽象介面（`INearfieldHardware`）
- [ ] 撰寫單元測試

**C# 實作範例**:
```csharp
/// <summary>
/// 2.4G 硬體抽象介面
/// </summary>
public interface INearfieldHardware
{
    bool Initialize();
    bool SendMessage(int targetUnitSid, byte[] message);
    void RegisterCallback(Action<int, byte[]> callback);
    IReadOnlyList<int> GetConnectedUnits();
    bool GetNearfieldStatus();
}

/// <summary>
/// 模擬的 2.4G 硬體 (用於純軟體測試)
/// </summary>
public class SimulatedNearfield : INearfieldHardware
{
    private readonly NearfieldBus _messageBus;

    public bool Initialize() => true;

    public bool SendMessage(int targetUnitSid, byte[] message)
        => _messageBus.SendRaw(targetUnitSid, message);

    // ... 其他實作
}

/// <summary>
/// 真實的 2.4G 硬體 (需要在 Station 硬體上運行)
/// </summary>
public class RealNearfield : INearfieldHardware
{
    private readonly string _devicePath;
    private SerialPort? _serial;

    public bool Initialize()
    {
        try
        {
            _serial = new SerialPort(_devicePath, 115200);
            _serial.Open();
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "初始化 2.4G 硬體失敗");
            return false;
        }
    }

    // ... 其他實作
}
```

**驗收標準**:
- ✅ Station 能接收 Robot 的 2.4G 消息
- ✅ Station 能發送任務給 Robot
- ✅ 硬體抽象層允許切換模擬/真實硬體
- ✅ 單元測試通過率 100%

**交付成果**:
- 更新 `StationSimulator.cs`
- `INearfieldHardware.cs`
- `SimulatedNearfield.cs`
- `RealNearfield.cs`
- `StationSimulatorNearfieldTests.cs`
- `docs/implementation/stage-23-station-nearfield.md`

---

#### **Stage 24: 2.4G 通訊驗證** ⏱️ 4 小時

**目標**: 執行階段驗證（純雲端 + 雲端中轉）

**參考文檔**: `04-station-robot-verification.md`

**任務清單**:
- [ ] 執行階段 1 驗證（純雲端）
  - [ ] 設備註冊/登入成功
  - [ ] StreamUnitControl 流建立
  - [ ] StreamUnitEvents 流建立
  - [ ] 收到雲端任務分配
- [ ] 執行階段 2 驗證（雲端中轉）
  - [ ] 訂閱站點任務更新
  - [ ] 收到 Robot 心跳數據
  - [ ] 觀察到任務狀態流轉
  - [ ] Station 確認 Robot 完成
- [ ] 撰寫驗證報告

**驗收標準**:
- ✅ 階段 1 所有測試項目通過
- ✅ 階段 2 所有測試項目通過
- ✅ 驗證報告完整

**交付成果**:
- 驗證測試程式碼
- `docs/validation/stage-24-nearfield-verification.md`

---

#### **Stage 25: 日誌和監控** ⏱️ 4 小時

**目標**: 完善日誌和監控功能

**任務清單**:
- [ ] 實作結構化日誌（Serilog 或 Microsoft.Extensions.Logging）
- [ ] 實作日誌級別控制
- [ ] 實作 2.4G 通訊日誌（air_says 記錄）
- [ ] 實作性能指標收集（心跳延遲、消息吞吐量）
- [ ] 實作健康檢查端點
- [ ] 撰寫文檔

**驗收標準**:
- ✅ 日誌完整且結構化
- ✅ 2.4G 消息可追蹤
- ✅ 性能指標可觀測
- ✅ 健康檢查正常工作

**交付成果**:
- 日誌配置
- 監控指標定義
- `docs/implementation/stage-25-logging-monitoring.md`

---

#### **Stage 26: 文檔和部署** ⏱️ 4 小時

**目標**: 完善文檔和部署指南

**任務清單**:
- [ ] 撰寫使用指南（含 2.4G 通訊說明）
- [ ] 撰寫 API 文檔
- [ ] 撰寫故障排查指南
- [ ] 建立部署腳本
- [ ] 建立 Docker 映像（選用）
- [ ] 撰寫 2.4G 硬體整合指南

**驗收標準**:
- ✅ 文檔完整且易懂
- ✅ 2.4G 通訊流程有說明
- ✅ 部署流程清晰
- ✅ 故障排查指南實用

**交付成果**:
- 使用指南
- API 文檔
- 故障排查指南
- 2.4G 硬體整合指南
- 部署腳本
- `docs/implementation/stage-26-documentation.md`

---

## 測試策略

### 測試金字塔

```
        ┌─────────────┐
        │ E2E 測試    │ 10%  (整合測試)
        ├─────────────┤
        │ 整合測試    │ 20%  (模組間測試)
        ├─────────────┤
        │ 單元測試    │ 70%  (每個方法測試)
        └─────────────┘
```

### 測試覆蓋率目標

- **單元測試覆蓋率**: ≥ 80%
- **分支覆蓋率**: ≥ 70%
- **整合測試覆蓋率**: 100% 關鍵流程
- **E2E 測試覆蓋率**: 100% 主要場景

### 每個 Stage 的測試要求

1. **單元測試**: 每個公開方法都要有對應的單元測試
2. **Mock 使用**: 使用 Moq 框架 Mock 外部依賴
3. **測試場景**:
   - ✅ 正常場景（Happy Path）
   - ✅ 錯誤場景（Error Cases）
   - ✅ 邊界條件（Edge Cases）
4. **測試命名**: 使用 `MethodName_Scenario_ExpectedResult` 格式

---

## 可交付成果

### 程式碼結構

```
YogoCloudSimulate/
├── YogoCloudSimulate.sln
│
├── src/
│   ├── Common/
│   │   ├── Common.csproj
│   │   ├── Protos/
│   │   │   └── jarvis_iot.proto
│   │   ├── GrpcClientBase.cs
│   │   ├── UnitRegistration.cs
│   │   ├── UnitLogin.cs
│   │   ├── UnitControlStream.cs
│   │   ├── UnitEventStream.cs
│   │   ├── Tasks/
│   │   │   └── TaskStateMachine.cs
│   │   │
│   │   └── Nearfield/                    # 2.4G 通訊模組 (Phase 5)
│   │       ├── AirSayTypes.cs            # 2.4G 消息類型定義
│   │       ├── AirSayGenerator.cs        # 消息生成器
│   │       ├── NearfieldBus.cs           # 2.4G 近場通訊總線
│   │       ├── INearfieldHardware.cs     # 硬體抽象介面
│   │       ├── SimulatedNearfield.cs     # 軟體模擬實作
│   │       └── RealNearfield.cs          # 真實硬體實作
│   │
│   ├── StationSimulator/
│   │   ├── StationSimulator.csproj
│   │   ├── Program.cs
│   │   ├── StationSimulator.cs
│   │   ├── TaskQueue.cs
│   │   ├── TaskDispatcher.cs
│   │   └── appsettings.json
│   │
│   └── RobotSimulator/
│       ├── RobotSimulator.csproj
│       ├── Program.cs
│       ├── RobotSimulator.cs
│       ├── TaskExecutor.cs
│       ├── ProgressReporter.cs
│       └── appsettings.json
│
└── tests/
    ├── Common.Tests/
    │   ├── Common.Tests.csproj
    │   ├── GrpcClientBaseTests.cs
    │   ├── UnitRegistrationTests.cs
    │   ├── UnitLoginTests.cs
    │   ├── Tasks/
    │   │   └── TaskStateMachineTests.cs
    │   │
    │   └── Nearfield/                    # 2.4G 模組測試 (Phase 5)
    │       ├── AirSayTypesTests.cs
    │       ├── AirSayGeneratorTests.cs
    │       └── NearfieldBusTests.cs
    │
    ├── StationSimulator.Tests/
    │   ├── StationSimulator.Tests.csproj
    │   ├── StationSimulatorTests.cs
    │   └── StationSimulatorNearfieldTests.cs   # 2.4G 測試 (Phase 5)
    │
    ├── RobotSimulator.Tests/
    │   ├── RobotSimulator.Tests.csproj
    │   ├── RobotSimulatorTests.cs
    │   ├── TaskExecutorTests.cs
    │   └── RobotSimulatorNearfieldTests.cs     # 2.4G 測試 (Phase 5)
    │
    └── Integration.Tests/
        ├── Integration.Tests.csproj
        ├── EndToEndTests.cs
        └── NearfieldVerificationTests.cs       # 2.4G 驗證測試 (Phase 5)
```

### 文檔結構

```
docs/
├── planning/
│   └── phase-1-grpc-infrastructure.md
│
├── implementation/
│   ├── stage-01-project-setup.md
│   ├── stage-02-grpc-connection.md
│   ├── stage-03-register-unit.md
│   ├── ...
│   ├── stage-18-performance-test.md
│   ├── stage-19-air-say-types.md           # 2.4G 消息類型
│   ├── stage-20-air-say-generator.md       # 消息生成器
│   ├── stage-21-nearfield-bus.md           # 近場通訊總線
│   ├── stage-22-robot-nearfield.md         # Robot 2.4G 優化
│   ├── stage-23-station-nearfield.md       # Station 2.4G 優化
│   ├── stage-25-logging-monitoring.md      # 日誌和監控
│   └── stage-26-documentation.md           # 文檔和部署
│
├── validation/
│   ├── stage-17-integration-test.md
│   ├── stage-18-performance-test.md
│   ├── stage-24-nearfield-verification.md  # 2.4G 通訊驗證
│   └── test-results/
│
└── simulate/                               # 系統理解文檔
    ├── 00-implementation-plan.md           # 本文檔
    ├── 01-phase1-infrastructure.md
    ├── 02-2.4g-simulation-enhancement.md   # 2.4G 增強方案
    ├── 03-2.4g-clues.md                    # 2.4G 線索分析
    ├── 04-station-robot-verification.md    # 驗證方案
    └── 05-feasibility-evaluation.md        # 可行性評估
```

---

## 時間估算

### 總時間

- **26 個 Stage** × 平均 **3.6 小時** = **94 小時**
- **預計工作天數**: 94 小時 ÷ 4 小時/天 = **24 天**

### 各 Phase 時間

| Phase | Stage 數 | 預估時間 |
|-------|---------|---------|
| Phase 1: 基礎設施 | 6 | 18 小時 |
| Phase 2: Station 模擬器 | 5 | 18 小時 |
| Phase 3: Robot 模擬器 | 5 | 16 小時 |
| Phase 4: 完整閉環 | 2 | 8 小時 |
| Phase 5: 2.4G 通訊優化與部署 | 8 | 34 小時 |
| **總計** | **26** | **94 小時** |

---

## 下一步行動

### 當前進度 ✅

1. ✅ **Phase 1 完成** (2025-12-03) - 基礎設施已完成，包括：
   - Stage 1: 專案初始化和 Proto 編譯 ✅
   - Stage 2: gRPC 基礎連接 ✅
   - Stage 3: RSRegisterUnit 註冊功能 ✅
   - Stage 4: RSLoginUnit 登入功能 ✅
   - Stage 5: 任務狀態機 ✅
   - Stage 6: 本地消息總線 ✅

2. 📋 **Phase 2 規劃中** - Stage 7-8 規劃完成：
   - Stage 7: StreamUnitControl 接收任務 📋 規劃完成
   - Stage 8: StreamUnitEvents 上報事件 📋 規劃完成

### 立即行動

1. 📅 **開始 Stage 7 實作** - StreamUnitControl 接收任務
2. 📅 **開始 Stage 8 實作** - StreamUnitEvents 上報事件

### 開發檢查點

- ✅ **Stage 1-6 完成**: Phase 1 基礎設施完成，可進行基礎 gRPC 通信
- **Stage 7-11 完成**: Station 模擬器完成，可接收和轉派任務
- **Stage 12-16 完成**: Robot 模擬器完成，可執行任務並回報
- **Stage 17-18 完成**: 完整閉環測試通過，系統可運行
- **Stage 19-21 完成**: 2.4G 消息結構和 NearfieldBus 完成，可模擬真實 2.4G 通訊
- **Stage 22-23 完成**: Robot/Station 2.4G 優化完成，air_says 可正確生成和上報
- **Stage 24 完成**: 2.4G 通訊驗證完成，純雲端和雲端中轉驗證通過
- **Stage 25-26 完成**: 日誌監控和文檔完善，系統可交付

---

## 附錄

### 參考資料

- **系統架構**: `docs/01-system-architecture-overview.md`
- **任務閉環流程**: `docs/02-task-loop-technical-doc.md`
- **API 參考**: `docs/03-api-reference.md`
- **開發規則**: `claude.md`

### 技術文檔

- [.NET 官方文檔](https://learn.microsoft.com/dotnet/)
- [gRPC for .NET](https://grpc.io/docs/languages/csharp/)
- [Protocol Buffers](https://protobuf.dev/)
- [xUnit 文檔](https://xunit.net/)

---

**創建者**: AI Assistant
**審核者**: 待定
**批准者**: 待定
**版本歷史**:
- v3.1 (2025-12-03): 更新進度狀態
  - Phase 1 Stage 1-6 全部完成 ✅
  - Phase 2 Stage 7-8 規劃完成 📋
  - 新增規劃文檔連結
  - 更新「下一步行動」章節
- v3.0 (2025-12-02): 擴展 Phase 5 為「2.4G 通訊優化與部署」，新增 Stage 19-26 共 8 個 Stage
  - 整合 `02-2.4g-simulation-enhancement.md` 的 2.4G 消息格式
  - 整合 `04-station-robot-verification.md` 的驗證方案
  - 整合 `05-feasibility-evaluation.md` 的可行性評估
  - 新增 AirSayTypes、AirSayGenerator、NearfieldBus 等模組
  - 新增 INearfieldHardware 硬體抽象層
  - 總 Stage 數從 20 增加至 26
- v2.0 (2025-11-24): 改用 C# 實作，重新規劃 20 個 Stage，每個 Stage 4 小時內完成
- v1.0 (2025-11-18): 初始版本（Python 實作）
