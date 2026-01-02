# 普渡 (Pudu) API 能力說明

## 概述

普渡 API 是普渡科技提供的開放平台接口，用於管理和控制普渡機器人。API 基礎網址為：
```
https://css-open-platform.pudutech.com/pudu-entry/
```

## 認證方式

API 使用 HMAC-SHA1 簽名認證機制：
- 需要 `ApiAppKey` 和 `ApiAppSecret`
- 請求需包含 `x-date`、`Authorization` 等 Header
- POST 請求需計算 Body 的 Content-MD5

---

## API 分類

### 一、通用 API

#### 1. 基本資料

| API 名稱 | 方法 | 端點 | 說明 |
|---------|------|------|------|
| 取得店鋪列表 | GET | `/data-open-platform-service/v1/api/shop` | 查詢所有店鋪，支援分頁 (limit, offset) |
| 取得機器列表 | GET | `/data-open-platform-service/v1/api/robot` | 根據 shop_id 查詢該店鋪的機器人列表 |
| 取得地圖列表 | GET | `/data-open-platform-service/v1/api/maps` | 根據 shop_id 查詢該店鋪的地圖列表 |
| 取得地圖詳細 | GET | `/data-open-platform-service/v1/api/map` | 取得特定地圖的詳細資訊，含裝置尺寸適配 |
| 取得機器即時位置 | GET | `/data-open-platform-service/v1/api/map/robotCurrentPosition` | 根據 shop_id 和 sn 取得機器人當前位置 |

#### 2. 總覽數據 API

| API 名稱 | 方法 | 端點 | 說明 |
|---------|------|------|------|
| 機器概覽 | GET | `/data-board/v1/brief/robot` | 查詢指定時間範圍內的機器概覽數據 |
| 機器運行概覽 | GET | `/data-board/v1/brief/run` | 查詢機器運行狀態概覽 |
| 店鋪概覽 | GET | `/data-board/v1/brief/shop` | 查詢店鋪整體運營概覽 |

#### 3. 通用分析數據

| API 名稱 | 方法 | 端點 | 說明 |
|---------|------|------|------|
| 店鋪分析 | GET | `/data-board/v1/analysis/shop` | 按時間單位 (hour/day) 分析店鋪數據 |
| 店鋪分析(分頁) | GET | `/data-board/v1/analysis/shop/paging` | 店鋪分析數據，支援分頁查詢 |
| 機器運行分析 | GET | `/data-board/v1/analysis/run` | 分析機器運行數據 |

#### 4. 日誌資料

| API 名稱 | 方法 | 端點 | 說明 |
|---------|------|------|------|
| 開機自檢日誌 | GET | `/data-board/v1/log/boot/query_list` | 查詢機器開機自檢紀錄 |
| 故障事件日誌 | GET | `/data-board/v1/log/error/query_list` | 查詢故障與錯誤事件紀錄 |
| 充電記錄 | GET | `/data-board/v1/log/charge/query_list` | 查詢機器充電歷史紀錄 |

---

### 二、清潔線路 API (清潔機器人)

| API 名稱 | 方法 | 端點 | 說明 |
|---------|------|------|------|
| 任務列表 | GET | `/cleanbot-service/v1/api/open/task/list` | 查詢清潔任務列表 |
| 機器人詳細狀態 | GET | `/cleanbot-service/v1/api/open/robot/detail` | 根據 sn 取得機器人詳細狀態 |
| 對機器人下發指令 | POST | `/cleanbot-service/v1/api/open/task/exec` | 下發充電等任務指令給機器人 |

**下發指令範例 (Body)：**
```json
{
   "sn": "8110H4522050001",
   "type": 1,
   "clean": {
      "status": 1
   }
}
```

---

### 三、遞送線路 API (送餐/遞送機器人)

| API 名稱 | 方法 | 端點 | 說明 |
|---------|------|------|------|
| 取得機器可用地圖清單 | GET | `/map-service/v1/open/list` | 根據 sn 查詢機器可用的地圖 |
| 取得機器目前使用的地圖 | GET | `/map-service/v1/open/current` | 查詢機器當前使用的地圖 |
| 取得機器使用中的地圖內的點位 | GET | `/map-service/v1/open/point` | 查詢地圖中的所有點位，支援分頁 |
| 呼叫機器 (Custom Call) | POST | `/open-platform-service/v1/custom_call` | 呼叫機器人前往指定點位 |

**呼叫機器範例 (Body)：**
```json
{
  "sn": "8PN034124050003",
  "map_name": "0#0#和椿臺北2摟",
  "point": "ray的家",
  "call_device_name": "appKey"
}
```

---

### 四、系統 API

| API 名稱 | 方法 | 端點 | 說明 |
|---------|------|------|------|
| Health Check | GET | `/data-open-platform-service/v1/api/healthCheck` | API 健康檢查 |
| Health Check | POST | `/data-open-platform-service/v1/api/healthCheck` | API 健康檢查 (POST) |

---

## 常用參數說明

| 參數 | 說明 |
|------|------|
| `shop_id` | 店鋪 ID |
| `sn` | 機器人序號 |
| `map_name` | 地圖名稱 |
| `limit` | 每頁筆數 |
| `offset` | 起始偏移量 |
| `start_time` | 查詢起始時間 (Unix timestamp) |
| `end_time` | 查詢結束時間 (Unix timestamp) |
| `timezone_offset` | 時區偏移 |
| `time_unit` | 時間單位 (hour/day) |
| `Language` | 語言設定 (如 zh-cn) |

---

## 能力總結

普渡 API 主要提供以下能力：

1. **資產管理** - 查詢店鋪、機器人、地圖等基本資料
2. **即時監控** - 取得機器人即時位置
3. **數據分析** - 機器概覽、運行分析、店鋪分析等統計報表
4. **日誌查詢** - 開機自檢、故障事件、充電紀錄等歷史日誌
5. **清潔機器人控制** - 任務管理、狀態查詢、指令下發
6. **遞送機器人控制** - 地圖管理、點位查詢、呼叫機器人
