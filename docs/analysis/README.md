# 一站式系統技術文檔

**文檔版本**: v1.0
**最後更新**: 2025-10-31
**維護團隊**: 技術團隊
**狀態**: ✅ 完成

---

## 📚 文檔總覽

本文檔集基於2025-10-28至2025-10-30期間對K8s一站式智慧配送系統的完整探索成果編寫，包含實際log驗證和物理實驗數據。

### 核心文檔（7篇）

| # | 文檔名稱 | 說明 | 目標讀者 | 字數 |
|---|---------|------|---------|------|
| 1 | [系統架構總覽](./01-system-architecture-overview.md) | 系統整體架構、核心服務層、通信架構 | 架構師、技術Lead、新成員 | ~8,000 |
| 2 | [任務閉環流程技術文檔](./02-task-loop-technical-doc.md) | 任務從創建到完成的完整流程和實際案例 | 開發工程師、架構師 | ~12,000 |
| 3 | [API參考文檔](./03-api-reference.md) | gRPC服務、HTTP API、數據模型完整參考 | 開發工程師、前端團隊 | ~10,000 |
| 4 | [運維和故障排查手冊](./04-operations-troubleshooting.md) | 日常運維、故障排查、應急響應 | 運維工程師、DevOps | ~7,000 |
| 5 | [最佳實踐和改進建議](./05-best-practices-improvements.md) | 設計亮點、短中長期改進建議、開發規範 | 技術Lead、架構師、開發團隊 | ~9,000 |
| 6 | [JWT認證指南](./06-jwt-authentication-guide.md) | JWT認證機制和使用方式 | 開發工程師 | ~3,000 |
| 7 | [DAPR與Actor分析報告](./07-dapr-actor-analysis.md) | DAPR架構、Actor模式、事件驅動機制深度分析 | 架構師、開發工程師 | ~5,000 |

**總計**: ~54,000字，約180頁A4文檔

---

## 🎯 快速導航

### 我是新成員，想了解系統
👉 從這裡開始：
1. [系統架構總覽](./01-system-architecture-overview.md) - 了解整體架構
2. [任務閉環流程](./02-task-loop-technical-doc.md) - 理解核心業務流程
3. [API參考](./03-api-reference.md) - 熟悉API接口

### 我是開發工程師
👉 你需要：
1. [任務閉環流程](./02-task-loop-technical-doc.md) - 理解業務邏輯
2. [API參考](./03-api-reference.md) - 查閱API定義
3. [最佳實踐](./05-best-practices-improvements.md) - 遵循開發規範

### 我是運維工程師
👉 你需要：
1. [運維和故障排查手冊](./04-operations-troubleshooting.md) - 日常運維和故障排查
2. [系統架構總覽](./01-system-architecture-overview.md) - 了解系統組件
3. [最佳實踐](./05-best-practices-improvements.md) - 查看改進建議

### 我是架構師/技術Lead
👉 你需要：
1. [系統架構總覽](./01-system-architecture-overview.md) - 審視整體架構
2. [最佳實踐和改進建議](./05-best-practices-improvements.md) - 規劃技術演進
3. [任務閉環流程](./02-task-loop-technical-doc.md) - 理解核心流程

---

## 📖 文檔詳細介紹

### 1. 系統架構總覽

**文件**: `01-system-architecture-overview.md`

**內容概要**:
- 📌 系統整體架構圖
- 📌 三層命名空間架構（Application / SPF / Infrastructure）
- 📌 核心服務層詳解
- 📌 雙層通信架構（4G + 2.4G）⭐⭐⭐
- 📌 數據流向和技術棧
- 📌 部署架構

**核心發現**:
- ✅ 雙層通信架構：4G雲端調度 + 2.4G本地協作
- ✅ 事件驅動設計：DAPR + TaskUpdateSignal
- ✅ Actor模式：每個Robot/Station獨立Actor實例
- ✅ 多層數據同步：TimescaleDB + Redis + DAPR EventBus

**適合場景**:
- 新成員onboarding
- 架構審查
- 技術分享

### 2. 任務閉環流程技術文檔

**文件**: `02-task-loop-technical-doc.md`

**內容概要**:
- 📌 任務生命週期和狀態機
- 📌 階段1：任務創建（訂單 → 任務）
- 📌 階段2：任務分配（雲端 → Robot/Station）
- 📌 階段3：任務執行（Robot在Station內執行）
- 📌 階段4：任務回報（進度更新機制）
- 📌 階段5：任務完成（多層數據同步）
- 📌 異常處理流程
- 📌 實際案例分析（含實際log）⭐

**核心發現**:
- ✅ SyncMeteorEvents：Robot高頻事件同步（每1-5秒）
- ✅ TaskUpdateSignal：DAPR事件驅動任務狀態更新
- ✅ 兩種任務分配模式：直接分配 vs Station轉派
- ✅ 2.4G本地通信的關鍵作用（經物理實驗驗證）

**適合場景**:
- 理解業務邏輯
- 故障排查
- 新功能開發

### 3. API參考文檔

**文件**: `03-api-reference.md`

**內容概要**:
- 📌 認證機制（JWT）
- 📌 gRPC服務完整參考
  - app-delivery-v2.AppDeliveryV2
  - jarvis_iot.JarvisIot
  - app_shop.AppTask
- 📌 HTTP RESTful API
- 📌 DAPR事件規範
- 📌 數據模型（Task、Unit、Order）
- 📌 錯誤碼定義

**核心內容**:
- ✅ 完整的gRPC Proto定義（含推導）
- ✅ HTTP API端點和請求/響應範例
- ✅ DAPR事件格式和訂閱方式
- ✅ Python使用範例

**適合場景**:
- 開發時查閱API
- 前後端對接
- 第三方系統整合

### 4. 運維和故障排查手冊

**文件**: `04-operations-troubleshooting.md`

**內容概要**:
- 📌 日常運維（K8s管理、服務重啟、資源監控）
- 📌 監控和告警（Grafana、DAPR Metrics）
- 📌 常見故障排查
  - 任務分配失敗
  - Robot無法登錄
  - Robot無法進入Station ⭐
  - 任務無法完成
  - 數據庫連接池耗盡
- 📌 日誌查詢指南（阿里雲Log Service + K8s）
- 📌 應急響應（P0-P4事件等級）
- 📌 定期維護清單

**核心內容**:
- ✅ 實用的kubectl命令
- ✅ 詳細的故障排查步驟
- ✅ 日誌查詢SQL範例
- ✅ 應急響應流程

**適合場景**:
- 日常運維
- 故障排查
- oncall值班

### 5. 最佳實踐和改進建議

**文件**: `05-best-practices-improvements.md`

**內容概要**:
- 📌 系統設計最佳實踐（現有架構亮點）
- 📌 短期改進建議（1-2週）
  - 整合分散式追蹤 🔴高優先
  - 增加日誌保留期 🟡中優先
  - 部署Prometheus Server 🟡中優先
- 📌 中期改進建議（1-2個月）
  - 配置死信佇列 🟡中優先
  - 實作告警系統 🟡中優先
- 📌 長期改進建議（3-6個月）
  - 遷移OpenTelemetry 🟡中優先
  - 實施SLO/SLI監控 🟢低優先
- 📌 性能優化建議
- 📌 安全加固建議
- 📌 開發規範

**核心內容**:
- ✅ 優先級矩陣和時間表
- ✅ 詳細的實施步驟和代碼範例
- ✅ 工作量和投資回報評估
- ✅ 快速勝利（Quick Wins）建議

**適合場景**:
- 技術規劃
- 季度OKR制定
- 技術債務管理

### 7. DAPR與Actor分析報告 ⭐NEW

**文件**: `07-dapr-actor-analysis.md`

**內容概要**:
- 📌 分析來源與方法說明
- 📌 DAPR 架構分析（版本、Sidecar注入、系統組件）
- 📌 Actor 模式分析（類型、ID命名規則、Loop週期）
- 📌 事件驅動機制（DAPR Publish/Subscribe）
- 📌 任務訂閱模式（格式解析、訂閱者分析）
- 📌 狀態同步機制（UnitConsensus、Robot心跳）
- 📌 README驗證結論

**分析來源**:
- ✅ `clue/*1542.md` - 4個服務的實際log
- ✅ `apps/dapr_yaml/*.yaml` - DAPR系統配置
- ✅ `apps/app_yaml/app-delivery-actor-*.yaml` - Actor服務部署配置

**核心發現**:
- ✅ Robot和Station都會獨立訂閱任務
- ✅ DAPR 1.8.2 版本，Sidecar自動注入
- ✅ Actor Loop週期約10-20ms
- ✅ UnitConsensus每秒同步到Redis
- ✅ 毫秒級事件分發（發布和訂閱在同一秒內完成）

**適合場景**:
- 理解DAPR架構
- 深入Actor模式
- 事件驅動設計參考

---

## 🔍 核心發現速覽

### 雙層通信架構 ⭐⭐⭐⭐⭐

這是系統最重要的架構特徵之一：

```
雲端層（4G gRPC）：
  ├─ 任務調度和分配
  ├─ 狀態同步（每秒）
  └─ 事件流處理

本地層（2.4G）：
  ├─ Station-Robot握手
  ├─ 身份驗證
  ├─ 艙門控制
  └─ 實時協作
```

**物理實驗證據**（2025-10-28）:
- 拔除2.4G電源線 → Robot無法進入Station
- 證明2.4G對本地協作至關重要
- 詳見：`exploration/findings/2.4G-experiment-analysis.md`

### 事件驅動架構

**核心事件**: TaskUpdateSignal

**特點**:
- 通過DAPR Pubsub解耦
- 統一的任務狀態更新
- 支持水平擴展

**實際log證據**:
```
15:43:52 [EventSourcing] DAPR Publish T_TaskUpdateSignal(11961)
```

### Actor模式

**每個Robot/Station都是獨立的Actor實例**:
```python
actor_id = "s504.u1010020021"  # site_uid=504, unit_uid=1010020021
```

**優勢**:
- 狀態變更集中管理
- 避免分散式鎖競爭
- Redis每秒更新UnitConsensus

---

## 📊 探索成果統計

### 探索時間軸

- **2025-10-28**: 階段1-3探索 + 實際log驗證（15:42-15:45）
- **2025-10-28**: 2.4G物理實驗（16:32-16:36）
- **2025-10-30**: 階段4-5探索
- **2025-10-31**: 技術文檔編寫

### 探索覆蓋度

- ✅ 5個探索階段全部完成（100%）
- ✅ 1次物理實驗驗證
- ✅ 4組實際log文件分析
- ✅ 50+ 微服務配置檔案審查
- ✅ 5篇完整技術文檔

### 產出物統計

| 類型 | 數量 | 說明 |
|------|------|------|
| 探索發現文檔 | 5篇 | 階段1-5 |
| 物理實驗報告 | 1篇 | 2.4G驗證 |
| 實際log文件 | 8個 | 2個時間段，4個服務 |
| 技術文檔 | 5篇 | 架構、流程、API、運維、最佳實踐 |
| 文檔總字數 | ~46,000字 | 約150頁A4 |
| Mermaid圖表 | 10+ 張 | 時序圖、架構圖 |

---

## 🎓 學習路徑

### 新手入門（1-2天）

**Day 1: 理解系統**
1. 閱讀：系統架構總覽（2小時）
2. 閱讀：任務閉環流程（3小時）
3. 動手：連接K8s集群，查看Pod狀態（1小時）

**Day 2: 熟悉API**
1. 閱讀：API參考文檔（2小時）
2. 動手：使用grpcurl測試gRPC API（1小時）
3. 動手：使用curl測試HTTP API（1小時）
4. 閱讀：最佳實踐中的開發規範（1小時）

### 進階學習（1週）

**Week 1**:
- 閱讀所有探索發現文檔（`exploration/findings/`）
- 分析實際log文件（`clue/`）
- 練習故障排查（參考運維手冊）
- 理解DAPR、Actor、TimescaleDB等技術棧

### 專家級（1個月）

**Month 1**:
- 深入理解雙層通信架構
- 研究DAPR事件驅動模式
- 優化現有代碼
- 實施改進建議中的快速勝利項目

---

## 🛠️ 推薦工具

### 開發工具

- **IDE**: VSCode / PyCharm
- **gRPC測試**: grpcurl, BloomRPC
- **API測試**: Postman, curl
- **K8s管理**: kubectl, k9s, Lens

### 監控工具

- **Grafana**: https://robot-api.aurotek.com/grafana
- **阿里雲Log**: 阿里雲控制台
- **kubectl logs**: 實時日誌查看

### 文檔工具

- **Markdown編輯器**: Typora, VSCode
- **圖表**: Mermaid, draw.io
- **API文檔**: Swagger, Postman

---

## 📞 聯絡和反饋

### 文檔維護

- **負責人**: 技術團隊
- **更新頻率**: 季度審查
- **問題反饋**: 請通過團隊溝通渠道

### 貢獻指南

歡迎貢獻！如果你發現：
- 文檔錯誤或過時
- 新的最佳實踐
- 改進建議
- 實用的腳本或工具

請提交PR或聯絡技術Lead。

---

## 📋 附錄

### 相關探索文檔

| 文檔 | 路徑 |
|------|------|
| 探索進度總覽 | `exploration/PROGRESS.md` |
| 階段1發現 | `exploration/findings/01-cloud-task-assignment.md` |
| 階段2發現 | `exploration/findings/02-station-task-reception.md` |
| 階段3發現（已驗證） | `exploration/findings/03-robot-task-execution-VERIFIED.md` |
| 2.4G實驗報告 | `exploration/findings/2.4G-experiment-analysis.md` |
| 階段4發現 | `exploration/findings/04-task-completion-report.md` |
| 階段5發現 | `exploration/findings/05-monitoring-and-observability.md` |

### 實際log文件

| 文件 | 時間段 | 說明 |
|------|--------|------|
| `clue/delivery-actor-robot_1542.md` | 15:42 | Robot Actor日誌 |
| `clue/app-delivery-pubsub_1542.md` | 15:42 | Pubsub事件日誌 |
| `clue/delivery_javis_iot_1542.md` | 15:42 | IoT層通信日誌 |
| `clue/delivery_rpc_1542.md` | 15:42 | RPC服務日誌 |
| `clue/delivery-actor-robot_1632.md` | 16:32 | 2.4G實驗Robot Actor |
| `clue/delivery_javis_iot_1632.md` | 16:32 | 2.4G實驗IoT層 |
| `clue/app-delivery-pubsub_1632.md` | 16:32 | 2.4G實驗Pubsub |

### 系統配置文件

| 類型 | 路徑 |
|------|------|
| Traefik路由 | `traefik/*.yaml` |
| 應用服務 | `apps/app_yaml/*.yaml` |
| SPF服務 | `apps/spf_yaml/*.yaml` |
| 前端服務 | `apps/fe-spf_yaml/*.yaml` |
| 基礎設施 | `apps/infra_yaml/*.yaml` |

---

## 🎉 快速開始

想要快速了解系統？跟著這個5分鐘教程：

1. **了解系統是什麼**（2分鐘）
   - 閱讀：`01-system-architecture-overview.md` 的「系統簡介」部分

2. **看一個實際案例**（2分鐘）
   - 閱讀：`02-task-loop-technical-doc.md` 的「實際案例分析」部分

3. **嘗試一個API**（1分鐘）
   - 參考：`03-api-reference.md` 的「認證機制」和「查詢任務狀態」

現在你已經對系統有基本了解了！🚀

---

**文檔版本**: v1.0 | **最後更新**: 2025-10-31 | **授權**: 內部使用
