 一站式系统路由分析报告

  📋 外部访问域名

  - gRPC服务: robot-rpc.aurotek.com
  - HTTP/Web服务: robot-api.aurotek.com

  ---
  🔵 gRPC路由配置

  1. 应用层服务 (app-grpc.yaml)

  | 路由路径                             | 目标服务                 | 命名空间        | 端口   | 用途     |
  |----------------------------------|----------------------|-------------|------|--------|
  | /app_resource.AppResource/*      | app-resource-rpc-svc | application | 5000 | 资源管理   |
  | /app_ispace.AppIspace/*          | app-ispace-rpc-svc   | application | 5000 | 空间管理   |
  | /app_delivery_v2.AppDeliveryV2/* | app-delivery-rpc-svc | application | 5000 | 任务分发核心 |
  | /app_shop.AppTask/*              | app-task-rpc-svc     | application | 5000 | 订单任务管理 |
  | /app_user.AppUser/*              | app-user-svc         | application | 5000 | 用户管理   |

  关键发现：
  - ✅ app-delivery-v2 是任务分发的主服务
  - ✅ app-shop 处理商城订单转任务
  - 🔒 所有请求都通过 h2c (HTTP/2 Cleartext)

  ---
  2. IoT设备层服务 (javis-grpc.yaml)

  | 路由路径                                      | 目标服务              | 命名空间        | 端口   |
  优先级 | 用途           |
  |-------------------------------------------|-------------------|-------------|------|-----|-------------
  -|
  | /jarvis_iot.JarvisIot/*                   | jarvis-iot-v6-svc | spf         | 5000 | 20  |
  IoT设备通信核心    |
  | /jarvis_site/*                            | jarvis-site-svc   | spf         | 5000 | 10  | 站点管理
       |
  | /jarvis_site.JarvisSite/GetSiteUnitConfig | jarvis-site-svc   | spf         | 5000 | 11  |
  获取站点配置(特殊路由) |
  | /jarvis_cmdb/*                            | jarvis-cmdb-svc   | spf         | 80   | 10  |
  配置管理数据库      |
  | /jarvis_imp/*                             | jarvis-imp-svc    | spf         | 80   | 10  | 实施管理
       |
  | /jarvis_dobby/*                           | jarvis-dobby-svc  | spf         | 80   | 10  | Dobby服务
     |
  | /notification.Notification/*              | notification-svc  | application | 5000 | 10  | 通知服务
       |
  | /xsite.Xsite/*                            | xsite-rpc-svc     | spf         | 5000 | 20  | Xsite服务
     |

  关键发现：
  - ✅ jarvis-iot 是 Station/Robot 登录和通信的主入口
  - ✅ 优先级 20 > 10，更精确的路由优先匹配
  - 🎯 Station/Robot 通过 4G 调用 jarvis_iot.JarvisIot 服务

  ---
  🌐 HTTP/Web 路由配置

  3. IoT设备HTTP接口 (javis-web.yaml)

  | 路由路径   | 目标服务                 | 端口   | 用途          |
  |--------|----------------------|------|-------------|
  | /iot/* | jarvis-iot-gw-v6-svc | 8000 | IoT设备HTTP网关 |

  ---
  🔄 任务闭环的路由流程

  ┌─────────────────────────────────────────────────────────┐
  │  任务分配流程 (云端 → 设备)                              │
  └─────────────────────────────────────────────────────────┘

  1️⃣ 用户下单
     HTTP POST → robot-api.aurotek.com/app-shop/customer/order2/new
     → app-shop-rpc-svc (订单服务)

  2️⃣ 创建配送任务
     gRPC → app-delivery-rpc-svc
     → AssignTask(task_uuid, unit_uid, ...)

  3️⃣ 推送到 Station
     通过 jarvis-iot 推送任务
     → robot-rpc.aurotek.com/jarvis_iot.JarvisIot/[PushTask方法]
     或 通过 MQTT 异步推送

  ┌─────────────────────────────────────────────────────────┐
  │  任务回报流程 (设备 → 云端)                              │
  └─────────────────────────────────────────────────────────┘

  4️⃣ Station/Robot 回报
     gRPC → robot-rpc.aurotek.com/jarvis_iot.JarvisIot/[上报方法]
     → jarvis-iot-v6-svc
     → 记录到 jarvis-cmdb (日志/事件)

  5️⃣ 更新任务状态
     jarvis-iot → app-delivery-rpc-svc
     → ReportTask / CompleteTask
     → 更新 t_app_task_ledger2 表

  ---
  📊 服务命名空间分布

  application 命名空间

  业务应用层，74个部署：
  - app-delivery 系列 (任务分发)
  - app-shop 系列 (商城)
  - app-resource (资源)
  - app-user (用户)
  - notification (通知)

  spf 命名空间

  IoT平台层，45个部署：
  - jarvis-iot-v6 (设备通信核心)
  - jarvis-cmdb (配置数据库)
  - jarvis-site (站点)
  - xsite (扩展站点)

  infrastructure 命名空间

  基础设施：
  - traefik (路由网关)
  - tsdb (时序数据库)
  - redis, kafka, minio 等

  ---
  🎯 关键路由优先级

  优先级 20 (高):
    - /jarvis_iot.JarvisIot/*        ← Station/Robot 主通信
    - /app_delivery_v2.AppDeliveryV2/* ← 任务分发
    - /xsite.Xsite/*

  优先级 11 (特殊):
    - /jarvis_site.JarvisSite/GetSiteUnitConfig ← 配置获取

  优先级 10 (一般):
    - 其他所有服务

  ---
  💡 重要发现

  1. 双层路由架构：
    - 应用层 (application) - 处理业务逻辑
    - IoT层 (spf) - 处理设备通信
  2. 任务流核心：
    - app-delivery-v2 负责任务调度
    - jarvis-iot 负责设备通信
    - 两者通过 gRPC 或 MQTT 协作
  3. 统一入口：
    - gRPC: robot-rpc.aurotek.com
    - HTTP: robot-api.aurotek.com
    - 内部: inner-rpc.com
