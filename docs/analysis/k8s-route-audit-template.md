
# Kubernetes 路由追蹤工作單（VS Code 版）
更新時間：2025-10-20 01:28

> 使用方式：每條路由建立一節，將 VS Code Kubernetes Explorer 的 **Describe / View YAML / Show Endpoints / Logs** 內容整理後貼入。

---

## 叢集與命名空間
- 叢集：`default`
- 命名空間：`infrastructure`
- 入口控制器：`traefik`（Deployment/Service 名稱）
- 備註：

---

## 路由清單盤點
> 先貼 `kubectl -n infrastructure get ingressroute,ingressroutetcp,ingressrouteudp` 的輸出

```
# 貼輸出
```

---

## 範本：L7（IngressRoute）
### 路由名稱：`<ingressroute-name>`
- EntryPoints：`<web|websecure|...>`
- 規則 Match：`Host(\"...\") && PathPrefix(\"/api\")`（或其他）
- Middlewares（順序即為套用順序）：`[ ... ]`
- TLS：`terminating | passthrough`（其一）
- 服務對應：
  - Service：`<svc-name>`
  - Port：`<port or name>`
- 後端 Endpoints（Pod IP / Ready 狀態）：
  - `10.x.x.x:xxxx`（Pod：`<pod-name>`）
- 驗證指令（貼成功/失敗與回應碼）：
  - `curl -I https://<host>/healthz`
  - `grpcurl -insecure -authority <host> <host>:443 list`
- 記錄截圖/重點：
  - Traefik Access Log：`...`
  - 後端服務日誌：`...`
- 結論與問題：

---

## 範本：L4（IngressRouteTCP / IngressRouteUDP）
### 路由名稱：`<ingressroutetcp-or-udp-name>`（協定：`TCP|UDP`）
- EntryPoints / 端口：`<:1883 / :9092 / :4222 ...>`
- 服務對應：
  - Service：`<svc-name>`
  - Port：`<port>`
- 後端 Endpoints（Pod IP / Ready 狀態）：
  - `10.x.x.x:xxxx`（Pod：`<pod-name>`）
- 驗證指令：
  - MQTT（例）：`mosquitto_pub/sub ...`
  - Kafka（例）：`kafkacat ...`
  - NATS（例）：`nats ...`
  - UDP（例）：`echo -n "ping" | nc -u <host> <port>`
- 記錄截圖/重點：
  - Traefik Access/Service Log（若已開啟）：`...`
  - 後端服務日誌：`...`
- 結論與問題：

---

## 中介（Middlewares）盤點
> `kubectl -n infrastructure get middleware` 與每個 middleware 的用途與掛載位置

| 名稱 | 類型/功能 | 作用於哪些路由（順序） | 備註 |
|---|---|---|---|
| `http-redirect-https` | 301 轉址 | `app-web` → 1 |  |
| `strip-api-prefix` | PathPrefixStrip | `app-open-api` → 1 |  |
| `django-basic-auth` | BasicAuth | `external-nginx` → 1 |  |
| ... | ... | ... | ... |

---

## 服務/端點對照（快速表）
| 協定 | Host/Port/EntryPoint | IngressRoute(TCP/UDP) | Middlewares | Service:Port | Pods(Ready/IP) |
|---|---|---|---|---|---|
| HTTP | `websecure` `app-web` | `app-web` | `gzip, http-redirect-https` | `web-svc:443` | `web-xxx` |
| gRPC | `websecure` `jarvis-grpc` | `jarvis-grpc` | `-` | `jarvis-grpc-svc:50051` | `jarvis-xxx` |
| MQTT | `:1883` | `mqtt-route (TCP)` | `-` | `emqx:1883` | `emqx-0` |
| UDP | `:xxxxx` | `jarvis-udp (UDP)` | `-` | `jarvis-udp-svc:xxxxx` | `pod-xxx` |

---

## 常見問題檢查清單（逐條打勾）
- [ ] DNS/Host 指到正確 LB / Node IP
- [ ] IngressRoute 的 `match` 與實際 Host/Path 相符
- [ ] Middlewares 順序正確、規則生效（301 / strip / auth 等）
- [ ] Service Port 與後端容器 Port 一致
- [ ] Endpoints 存在且 Pod Ready
- [ ] Traefik Access Log 有紀錄、狀態碼合理
- [ ] 防火牆開通外網端口（TCP/UDP）
- [ ] 測試指令往返成功（含憑證/SNI/grpc h2c 等）

