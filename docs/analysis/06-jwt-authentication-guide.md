# JWT認證完整指南

**文檔版本**: v1.0
**最後更新**: 2025-11-05
**狀態**: ✅ 完成

---

## 目錄

1. [JWT認證概述](#jwt認證概述)
2. [認證架構](#認證架構)
3. [獲取JWT Token](#獲取jwt-token)
4. [使用JWT Token](#使用jwt-token)
5. [Token刷新和過期處理](#token刷新和過期處理)
6. [實際代碼範例](#實際代碼範例)
7. [常見問題](#常見問題)

---

## JWT認證概述

### 什麼是JWT？

JWT (JSON Web Token) 是一種開放標準 (RFC 7519)，用於在各方之間安全地傳輸信息。

**JWT結構**：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
│                Header                │                 Payload                │           Signature          │
```

三個部分：
1. **Header**: 算法和令牌類型
2. **Payload**: 聲明（用戶信息、過期時間等）
3. **Signature**: 簽名，用於驗證令牌未被篡改

### 系統中的JWT

在K8s一站式系統中，所有API調用都需要JWT認證：

- ✅ **HTTP API**: 通過 `Authorization` header傳遞
- ✅ **gRPC API**: 通過 metadata 傳遞
- ✅ **WebSocket**: 通過查詢參數或首次握手傳遞

---

## 認證架構

### 認證服務

系統有專門的認證服務處理JWT：

```
┌─────────────────────────────────────────────┐
│          認證服務層 (SPF Namespace)          │
├─────────────────────────────────────────────┤
│                                             │
│  jarvis-auth (gRPC)                         │
│    - 生成JWT Token                          │
│    - 驗證Token有效性                         │
│    - 用戶權限管理                            │
│    - Port: 5000                             │
│                                             │
│  jarvis-auth-gw (HTTP Gateway)              │
│    - HTTP → gRPC 轉換                       │
│    - RESTful API 提供                       │
│    - Port: 8000                             │
│                                             │
│  jarvis-auth-admin (Django Admin)           │
│    - 用戶管理後台                            │
│    - 權限配置                                │
│    - Port: 8000                             │
│                                             │
└─────────────────────────────────────────────┘
```

### 認證流程

```mermaid
sequenceDiagram
    participant User as 用戶/設備
    participant Traefik as Traefik Gateway
    participant AuthGW as jarvis-auth-gw
    participant Auth as jarvis-auth
    participant API as 業務API

    Note over User,API: 1. 登錄獲取Token
    User->>Traefik: POST /auth/login
    Traefik->>AuthGW: 轉發請求
    AuthGW->>Auth: gRPC LoginUser
    Auth->>Auth: 驗證用戶名密碼
    Auth->>Auth: 生成JWT Token
    Auth-->>AuthGW: Token + 過期時間
    AuthGW-->>Traefik: 返回Token
    Traefik-->>User: {access_token, expires_in}

    Note over User,API: 2. 使用Token調用API
    User->>Traefik: GET /api/tasks<br/>Authorization: Bearer {token}
    Traefik->>Traefik: 驗證JWT (Middleware)
    Traefik->>API: 轉發請求 (帶用戶信息)
    API->>API: 業務邏輯處理
    API-->>User: 返回數據
```

---

## 獲取JWT Token

### 方法1: 用戶名密碼登錄

**HTTP端點**:
```http
POST https://robot-api.aurotek.com/auth/login
Content-Type: application/json

{
  "username": "user@example.com",
  "password": "your_password"
}
```

**響應**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIiwiaWF0IjoxNzMwNzg0MDAwLCJleHAiOjE3MzA3ODc2MDB9.xyz...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "refresh_token_here"  // 可選
}
```

**Python範例**:
```python
import requests

def login(username: str, password: str) -> str:
    """登錄獲取JWT Token"""
    response = requests.post(
        'https://robot-api.aurotek.com/auth/login',
        json={
            'username': username,
            'password': password
        },
        timeout=10
    )
    response.raise_for_status()

    data = response.json()
    return data['access_token']

# 使用
jwt_token = login('user@example.com', 'password123')
print(f"Token: {jwt_token[:50]}...")
```

### 方法2: 設備認證（Robot/Station）

Robot和Station使用設備ID和密鑰進行認證。

**gRPC方法**: `jarvis_auth.JarvisAuth/LoginDevice`

**請求**（推導）:
```protobuf
message LoginDeviceRequest {
  int64 unit_uid = 1;        // 設備ID
  string device_type = 2;    // "ROBOT" or "STATION"
  string device_key = 3;     // 設備密鑰
  string firmware_version = 4;
}

message LoginDeviceResponse {
  bool success = 1;
  string access_token = 2;
  int32 expires_in = 3;
  string session_id = 4;
}
```

**Python範例**:
```python
# 需要proto生成的stub
from jarvis_auth import jarvis_auth_pb2, jarvis_auth_pb2_grpc
import grpc

def device_login(unit_uid: int, device_key: str) -> str:
    """設備登錄獲取Token"""
    channel = grpc.secure_channel(
        'robot-rpc.aurotek.com:443',
        grpc.ssl_channel_credentials()
    )
    stub = jarvis_auth_pb2_grpc.JarvisAuthStub(channel)

    request = jarvis_auth_pb2.LoginDeviceRequest(
        unit_uid=unit_uid,
        device_type='ROBOT',
        device_key=device_key,
        firmware_version='v2.3.5'
    )

    response = stub.LoginDevice(request)
    return response.access_token

# 使用
token = device_login(unit_uid=1010020021, device_key='device_secret_key')
```

### 方法3: 第三方OAuth登錄

系統支持微信、支付寶等第三方登錄。

**微信登錄端點**:
```http
GET https://robot-api.aurotek.com/chogori/api/v1/login/authorize/wechat?code={auth_code}
```

**支付寶登錄端點**:
```http
GET https://robot-api.aurotek.com/v3/chogori/api/v1/login/authorize/alipay?code={auth_code}
```

---

## 使用JWT Token

### HTTP API調用

在所有HTTP請求中添加 `Authorization` header：

```http
GET https://robot-api.aurotek.com/app-delivery/tasks/67985120
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Python範例**:
```python
import requests

def get_task_status(task_id: int, jwt_token: str) -> dict:
    """查詢任務狀態"""
    response = requests.get(
        f'https://robot-api.aurotek.com/app-delivery/tasks/{task_id}',
        headers={
            'Authorization': f'Bearer {jwt_token}'
        },
        timeout=10
    )
    response.raise_for_status()
    return response.json()

# 使用
task = get_task_status(67985120, jwt_token)
print(f"任務狀態: {task['status']}")
```

### gRPC調用

在gRPC metadata中添加認證信息：

```python
import grpc

# 創建認證metadata
metadata = [('authorization', f'Bearer {jwt_token}')]

# 調用gRPC方法
response = stub.GetTask(request, metadata=metadata)
```

**完整範例**:
```python
from app_delivery import app_delivery_pb2, app_delivery_pb2_grpc
import grpc

def query_tasks_grpc(site_uid: int, jwt_token: str):
    """通過gRPC查詢任務"""
    # 建立連接
    channel = grpc.secure_channel(
        'robot-rpc.aurotek.com:443',
        grpc.ssl_channel_credentials()
    )
    stub = app_delivery_pb2_grpc.AppDeliveryV2Stub(channel)

    # 構建請求
    request = app_delivery_pb2.GetTaskBoardRequest(
        site_uid=site_uid,
        status=['EXECUTING'],
        limit=10
    )

    # 添加認證metadata
    metadata = [('authorization', f'Bearer {jwt_token}')]

    # 調用
    response = stub.GetDispatchTaskBoard(request, metadata=metadata)

    return response.tasks
```

### cURL範例

```bash
# 登錄獲取Token
TOKEN=$(curl -s -X POST https://robot-api.aurotek.com/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"user@example.com","password":"password"}' \
  | jq -r '.access_token')

# 使用Token調用API
curl -H "Authorization: Bearer $TOKEN" \
  https://robot-api.aurotek.com/app-delivery/tasks/67985120
```

---

## Token刷新和過期處理

### Token過期時間

根據環境變量配置（`jarvis-auth.yaml`）：

```yaml
- name: AUTH_ACCESS_TOKEN_EXPIRATION_IN_SECONDS
  value: "604800"  # 7天 = 604800秒
```

**默認過期時間**: 7天（604800秒）

### 自動刷新實現

```python
from datetime import datetime, timedelta
from typing import Optional

class TokenManager:
    """JWT Token管理器（自動刷新）"""

    def __init__(self, username: str, password: str):
        self.username = username
        self.password = password
        self.token: Optional[str] = None
        self.expires_at: Optional[datetime] = None

    def get_token(self) -> str:
        """獲取有效Token（自動刷新）"""
        # 檢查是否需要刷新
        if self._needs_refresh():
            self._refresh_token()

        return self.token

    def _needs_refresh(self) -> bool:
        """檢查是否需要刷新"""
        if not self.token or not self.expires_at:
            return True

        # 提前5分鐘刷新
        return datetime.now() >= self.expires_at - timedelta(minutes=5)

    def _refresh_token(self):
        """刷新Token"""
        response = requests.post(
            'https://robot-api.aurotek.com/auth/login',
            json={
                'username': self.username,
                'password': self.password
            }
        )
        response.raise_for_status()

        data = response.json()
        self.token = data['access_token']
        expires_in = data.get('expires_in', 3600)
        self.expires_at = datetime.now() + timedelta(seconds=expires_in)

        print(f"✅ Token已刷新，有效期至: {self.expires_at}")

# 使用
token_manager = TokenManager('user@example.com', 'password')

# 自動獲取有效Token
jwt_token = token_manager.get_token()
```

### 處理401錯誤

```python
import requests
from requests.exceptions import HTTPError

def api_call_with_retry(url: str, token_manager: TokenManager):
    """帶重試的API調用"""
    max_retries = 2

    for attempt in range(max_retries):
        token = token_manager.get_token()

        try:
            response = requests.get(
                url,
                headers={'Authorization': f'Bearer {token}'},
                timeout=10
            )
            response.raise_for_status()
            return response.json()

        except HTTPError as e:
            if e.response.status_code == 401:
                # Token過期，強制刷新
                print(f"⚠️  Token過期，重試中... (嘗試 {attempt + 1}/{max_retries})")
                token_manager._refresh_token()

                if attempt == max_retries - 1:
                    raise
            else:
                raise
```

---

## 實際代碼範例

### 完整的Python客戶端

```python
import requests
from typing import Optional, Dict
from datetime import datetime, timedelta
import logging

logger = logging.getLogger(__name__)

class RobotApiClient:
    """K8s一站式系統API客戶端"""

    def __init__(self, base_url: str, username: str, password: str):
        self.base_url = base_url
        self.username = username
        self.password = password
        self.session = requests.Session()

        self._token: Optional[str] = None
        self._token_expires_at: Optional[datetime] = None

    def _get_valid_token(self) -> str:
        """獲取有效Token（自動刷新）"""
        if self._needs_refresh():
            self._login()
        return self._token

    def _needs_refresh(self) -> bool:
        """檢查是否需要刷新Token"""
        if not self._token or not self._token_expires_at:
            return True
        return datetime.now() >= self._token_expires_at - timedelta(minutes=5)

    def _login(self):
        """登錄獲取Token"""
        logger.info("🔐 正在登錄...")

        response = self.session.post(
            f"{self.base_url}/auth/login",
            json={
                'username': self.username,
                'password': self.password
            },
            timeout=10
        )
        response.raise_for_status()

        data = response.json()
        self._token = data['access_token']
        expires_in = data.get('expires_in', 3600)
        self._token_expires_at = datetime.now() + timedelta(seconds=expires_in)

        logger.info(f"✅ 登錄成功，Token有效期: {expires_in}秒")

    def _request(self, method: str, path: str, **kwargs) -> Dict:
        """統一的請求方法（自動處理認證）"""
        url = f"{self.base_url}{path}"
        token = self._get_valid_token()

        # 添加認證header
        headers = kwargs.pop('headers', {})
        headers['Authorization'] = f'Bearer {token}'

        # 發送請求
        response = self.session.request(
            method, url, headers=headers, **kwargs
        )

        # 處理401錯誤（Token過期）
        if response.status_code == 401:
            logger.warning("⚠️  Token過期，重新登錄...")
            self._login()

            # 重試一次
            token = self._token
            headers['Authorization'] = f'Bearer {token}'
            response = self.session.request(
                method, url, headers=headers, **kwargs
            )

        response.raise_for_status()
        return response.json()

    # ========== API方法 ==========

    def create_order(self, product_id: int, delivery_location: Dict) -> Dict:
        """創建訂單"""
        return self._request(
            'POST',
            '/app-shop/customer/order2/new',
            json={
                'product_id': product_id,
                'quantity': 1,
                'delivery_location': delivery_location,
                'customer_phone': '0912345678'
            }
        )

    def get_task_status(self, task_id: int) -> Dict:
        """查詢任務狀態"""
        return self._request('GET', f'/app-delivery/tasks/{task_id}')

    def get_task_list(self, site_uid: int, status: Optional[str] = None) -> Dict:
        """查詢任務列表"""
        params = {'site_uid': site_uid}
        if status:
            params['status'] = status
        return self._request('GET', '/app-delivery/tasks', params=params)

    def get_robot_status(self, unit_uid: int) -> Dict:
        """查詢Robot狀態"""
        return self._request('GET', f'/jarvis-iot/units/{unit_uid}')

# ========== 使用範例 ==========

# 創建客戶端
client = RobotApiClient(
    base_url='https://robot-api.aurotek.com',
    username='user@example.com',
    password='your_password'
)

# 調用API（自動處理認證）
order = client.create_order(
    product_id=12345,
    delivery_location={'building': 'A棟', 'floor': 5, 'room': '501'}
)
print(f"訂單創建成功，任務ID: {order['task_id']}")

# 查詢任務
task = client.get_task_status(order['task_id'])
print(f"任務狀態: {task['status']}")

# 查詢Robot狀態
robot = client.get_robot_status(1010020021)
print(f"Robot電量: {robot['battery']['level_percent']}%")
```

---

## 常見問題

### Q1: JWT Token在哪裡存儲？

**回答**:
- **瀏覽器**: 推薦存儲在 `httpOnly` Cookie 中（防止XSS攻擊）
- **移動App**: 存儲在安全的 KeyChain/KeyStore
- **Robot/Station**: 存儲在內存中，重啟後重新登錄

**示例（瀏覽器）**:
```javascript
// 不推薦：localStorage（易受XSS攻擊）
localStorage.setItem('jwt_token', token);

// 推薦：httpOnly Cookie（後端設置）
// Set-Cookie: jwt_token=xxx; HttpOnly; Secure; SameSite=Strict
```

### Q2: Token過期了怎麼辦？

**回答**:
1. **主動刷新**: 在過期前5分鐘自動刷新
2. **被動刷新**: 收到401錯誤時重新登錄
3. **Refresh Token**: 使用refresh_token獲取新的access_token（如果系統支持）

### Q3: 如何驗證Token有效性？

**方法1: 解析JWT檢查過期時間**
```python
import jwt
from datetime import datetime

def is_token_valid(token: str) -> bool:
    """檢查Token是否有效"""
    try:
        # 解碼JWT（不驗證簽名）
        payload = jwt.decode(token, options={"verify_signature": False})

        # 檢查過期時間
        exp = payload.get('exp')
        if exp:
            return datetime.fromtimestamp(exp) > datetime.now()
        return False
    except:
        return False
```

**方法2: 調用驗證API**
```python
def verify_token(token: str) -> bool:
    """通過API驗證Token"""
    response = requests.get(
        'https://robot-api.aurotek.com/auth/verify',
        headers={'Authorization': f'Bearer {token}'}
    )
    return response.status_code == 200
```

### Q4: gRPC如何傳遞JWT？

**回答**:
通過metadata傳遞：

```python
# 單個調用
metadata = [('authorization', f'Bearer {token}')]
response = stub.Method(request, metadata=metadata)

# 所有調用自動添加（使用Interceptor）
class AuthInterceptor(grpc.UnaryUnaryClientInterceptor):
    def __init__(self, token):
        self.token = token

    def intercept_unary_unary(self, continuation, client_call_details, request):
        # 添加認證metadata
        metadata = list(client_call_details.metadata or [])
        metadata.append(('authorization', f'Bearer {self.token}'))

        new_details = client_call_details._replace(metadata=metadata)
        return continuation(new_details, request)

# 使用
interceptor = AuthInterceptor(jwt_token)
channel = grpc.intercept_channel(channel, interceptor)
stub = ServiceStub(channel)
```

### Q5: 認證失敗的常見原因？

| 錯誤 | 原因 | 解決方案 |
|------|------|---------|
| 401 Unauthorized | Token無效或過期 | 重新登錄獲取新Token |
| 403 Forbidden | 權限不足 | 檢查用戶權限配置 |
| Token格式錯誤 | Header格式不正確 | 確保格式為 `Bearer {token}` |
| 簽名驗證失敗 | Token被篡改 | 使用原始Token，不要修改 |

---

## 安全最佳實踐

### ✅ DO（推薦）

1. **HTTPS傳輸**: 始終使用HTTPS傳輸Token
2. **短過期時間**: 設置合理的過期時間（1小時到7天）
3. **安全存儲**: 避免在localStorage等不安全位置存儲
4. **最小權限**: 僅授予必要的權限
5. **自動刷新**: 實現Token自動刷新機制

### ❌ DON'T（避免）

1. **URL參數傳遞**: 不要在URL中傳遞Token（會被記錄在日誌中）
2. **明文存儲**: 不要明文存儲在localStorage
3. **客戶端驗證**: 不要只在客戶端驗證Token
4. **長期有效**: 避免設置過長的過期時間
5. **共享Token**: 不同設備/用戶使用各自的Token

---

## 相關文檔

- [API參考文檔](./03-api-reference.md) - 完整API列表
- [系統架構總覽](./01-system-architecture-overview.md) - 認證服務架構
- [運維和故障排查手冊](./04-operations-troubleshooting.md) - 認證問題排查

---

**文檔維護**: 本文檔基於系統實際配置和最佳實踐編寫，如有更新請及時同步。
