# 电商系统架构设计

## 1. 系统概述

基于Go语言开发的分布式微服务电商平台，采用领域驱动设计（DDD）和微服务架构模式。

## 2. 技术栈

### 2.1 核心技术
- **语言**: Go 1.21+
- **微服务框架**: Go-Micro / Go-Kit / Kratos
- **API网关**: Kong / Traefik
- **服务注册与发现**: Consul / Etcd
- **负载均衡**: Nginx / Envoy
- **RPC框架**: gRPC
- **消息队列**: RabbitMQ / Kafka
- **缓存**: Redis (单机/集群)
- **数据库**:
  - MySQL 8.0+ (主数据存储)
  - MongoDB (日志/评价等非结构化数据)
  - Elasticsearch (商品搜索)
- **配置中心**: Consul / Nacos
- **链路追踪**: Jaeger / Zipkin
- **监控**: Prometheus + Grafana
- **日志**: ELK Stack (Elasticsearch + Logstash + Kibana)

### 2.2 开发工具
- **容器化**: Docker
- **编排**: Kubernetes / Docker Compose
- **CI/CD**: GitLab CI / Jenkins
- **版本控制**: Git

## 3. 微服务架构

### 3.1 服务拆分

```
┌─────────────────────────────────────────────────────────────┐
│                        API Gateway                          │
│                    (统一入口/鉴权/限流)                        │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼──────┐    ┌────────▼────────┐   ┌───────▼──────┐
│  User Service │    │ Product Service │   │ Order Service│
│  (用户服务)    │    │   (商品服务)     │   │  (订单服务)   │
└───────────────┘    └─────────────────┘   └──────────────┘
        │                     │                     │
┌───────▼──────┐    ┌────────▼────────┐   ┌───────▼──────┐
│  Auth Service │    │ Inventory Svc   │   │ Payment Svc  │
│  (认证服务)    │    │   (库存服务)     │   │  (支付服务)   │
└───────────────┘    └─────────────────┘   └──────────────┘
        │                     │                     │
┌───────▼──────┐    ┌────────▼────────┐   ┌───────▼──────┐
│  Cart Service │    │  Review Service │   │ Notify Svc   │
│  (购物车服务)  │    │   (评价服务)     │   │  (通知服务)   │
└───────────────┘    └─────────────────┘   └──────────────┘
```

### 3.2 服务清单

#### 3.2.1 用户服务 (User Service)
- 用户注册/登录/登出
- 用户信息管理（CRUD）
- 用户地址管理
- 用户等级/积分管理

#### 3.2.2 认证服务 (Auth Service)
- JWT Token生成与验证
- OAuth2.0授权
- 权限管理（RBAC）
- 会话管理

#### 3.2.3 商品服务 (Product Service)
- 商品信息管理（CRUD）
- 商品分类管理
- 商品SKU管理
- 商品搜索（集成ES）
- 商品详情查询

#### 3.2.4 库存服务 (Inventory Service)
- 库存查询
- 库存扣减（支持事务）
- 库存预留/释放
- 库存告警

#### 3.2.5 购物车服务 (Cart Service)
- 添加/删除购物车商品
- 购物车列表查询
- 购物车商品数量更新
- 购物车清空

#### 3.2.6 订单服务 (Order Service)
- 订单创建/取消
- 订单状态管理
- 订单查询（列表/详情）
- 订单支付回调处理
- 订单退款处理

#### 3.2.7 支付服务 (Payment Service)
- 支付接口对接（支付宝/微信）
- 支付订单创建
- 支付回调处理
- 退款处理
- 支付流水记录

#### 3.2.8 评价服务 (Review Service)
- 商品评价提交
- 评价查询（商品维度）
- 评价审核
- 评价统计

#### 3.2.9 通知服务 (Notification Service)
- 短信通知（订单/物流）
- 邮件通知
- 站内消息
- 推送通知

## 4. 核心数据模型

### 4.1 用户服务数据模型

```go
// User 用户表
type User struct {
    ID          int64     `json:"id" gorm:"primaryKey"`
    Username    string    `json:"username" gorm:"uniqueIndex;size:50"`
    Email       string    `json:"email" gorm:"uniqueIndex;size:100"`
    Phone       string    `json:"phone" gorm:"uniqueIndex;size:20"`
    Password    string    `json:"-" gorm:"size:255"` // 加密存储
    Nickname    string    `json:"nickname" gorm:"size:50"`
    Avatar      string    `json:"avatar" gorm:"size:255"`
    Gender      int8      `json:"gender"` // 0:未知 1:男 2:女
    Birthday    time.Time `json:"birthday"`
    Level       int       `json:"level" gorm:"default:1"` // 用户等级
    Points      int       `json:"points" gorm:"default:0"` // 积分
    Status      int8      `json:"status" gorm:"default:1"` // 1:正常 2:禁用
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
    DeletedAt   *time.Time `json:"-" gorm:"index"`
}

// UserAddress 用户地址表
type UserAddress struct {
    ID          int64   `json:"id" gorm:"primaryKey"`
    UserID      int64   `json:"user_id" gorm:"index"`
    Receiver    string  `json:"receiver" gorm:"size:50"`
    Phone       string  `json:"phone" gorm:"size:20"`
    Province    string  `json:"province" gorm:"size:50"`
    City        string  `json:"city" gorm:"size:50"`
    District    string  `json:"district" gorm:"size:50"`
    Address     string  `json:"address" gorm:"size:255"`
    IsDefault   bool    `json:"is_default" gorm:"default:false"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}
```

### 4.2 商品服务数据模型

```go
// Category 商品分类表
type Category struct {
    ID          int64     `json:"id" gorm:"primaryKey"`
    Name        string    `json:"name" gorm:"size:50"`
    ParentID    int64     `json:"parent_id" gorm:"default:0;index"`
    Level       int       `json:"level"` // 分类层级
    Sort        int       `json:"sort" gorm:"default:0"`
    Icon        string    `json:"icon" gorm:"size:255"`
    Status      int8      `json:"status" gorm:"default:1"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}

// Product 商品表
type Product struct {
    ID              int64     `json:"id" gorm:"primaryKey"`
    CategoryID      int64     `json:"category_id" gorm:"index"`
    Name            string    `json:"name" gorm:"size:200;index"`
    SubTitle        string    `json:"sub_title" gorm:"size:255"`
    Description     string    `json:"description" gorm:"type:text"`
    MainImage       string    `json:"main_image" gorm:"size:255"`
    Images          string    `json:"images" gorm:"type:text"` // JSON array
    Price           int64     `json:"price"` // 单位：分
    OriginalPrice   int64     `json:"original_price"` // 原价
    Stock           int       `json:"stock"` // 总库存
    Sales           int       `json:"sales" gorm:"default:0"` // 销量
    Status          int8      `json:"status" gorm:"default:1"` // 1:上架 2:下架
    Sort            int       `json:"sort" gorm:"default:0"`
    CreatedAt       time.Time `json:"created_at"`
    UpdatedAt       time.Time `json:"updated_at"`
    DeletedAt       *time.Time `json:"-" gorm:"index"`
}

// ProductSKU 商品SKU表
type ProductSKU struct {
    ID          int64     `json:"id" gorm:"primaryKey"`
    ProductID   int64     `json:"product_id" gorm:"index"`
    SKUCode     string    `json:"sku_code" gorm:"uniqueIndex;size:100"`
    Name        string    `json:"name" gorm:"size:200"`
    Attributes  string    `json:"attributes" gorm:"type:text"` // JSON: {"颜色":"红色","尺码":"L"}
    Price       int64     `json:"price"` // 单位：分
    Stock       int       `json:"stock"`
    Image       string    `json:"image" gorm:"size:255"`
    Status      int8      `json:"status" gorm:"default:1"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}
```

### 4.3 订单服务数据模型

```go
// Order 订单表
type Order struct {
    ID              int64     `json:"id" gorm:"primaryKey"`
    OrderNo         string    `json:"order_no" gorm:"uniqueIndex;size:50"` // 订单号
    UserID          int64     `json:"user_id" gorm:"index"`
    TotalAmount     int64     `json:"total_amount"` // 订单总金额（分）
    PayAmount       int64     `json:"pay_amount"` // 实付金额（分）
    DiscountAmount  int64     `json:"discount_amount"` // 优惠金额（分）
    FreightAmount   int64     `json:"freight_amount"` // 运费（分）
    PayType         int8      `json:"pay_type"` // 1:支付宝 2:微信 3:余额
    PayStatus       int8      `json:"pay_status"` // 1:未支付 2:已支付 3:已退款
    OrderStatus     int8      `json:"order_status"` // 1:待付款 2:待发货 3:已发货 4:已完成 5:已取消
    Receiver        string    `json:"receiver" gorm:"size:50"`
    ReceiverPhone   string    `json:"receiver_phone" gorm:"size:20"`
    ReceiverAddress string    `json:"receiver_address" gorm:"size:255"`
    Remark          string    `json:"remark" gorm:"type:text"`
    PaidAt          *time.Time `json:"paid_at"`
    ShippedAt       *time.Time `json:"shipped_at"`
    CompletedAt     *time.Time `json:"completed_at"`
    CancelledAt     *time.Time `json:"cancelled_at"`
    CreatedAt       time.Time `json:"created_at"`
    UpdatedAt       time.Time `json:"updated_at"`
}

// OrderItem 订单明细表
type OrderItem struct {
    ID              int64   `json:"id" gorm:"primaryKey"`
    OrderID         int64   `json:"order_id" gorm:"index"`
    ProductID       int64   `json:"product_id"`
    SKUID           int64   `json:"sku_id"`
    ProductName     string  `json:"product_name" gorm:"size:200"`
    SKUName         string  `json:"sku_name" gorm:"size:200"`
    SKUAttributes   string  `json:"sku_attributes" gorm:"type:text"`
    ProductImage    string  `json:"product_image" gorm:"size:255"`
    Price           int64   `json:"price"` // 购买单价（分）
    Quantity        int     `json:"quantity"` // 购买数量
    TotalAmount     int64   `json:"total_amount"` // 小计金额（分）
    CreatedAt       time.Time `json:"created_at"`
}
```

### 4.4 库存服务数据模型

```go
// Inventory 库存表
type Inventory struct {
    ID          int64     `json:"id" gorm:"primaryKey"`
    SKUID       int64     `json:"sku_id" gorm:"uniqueIndex"`
    Stock       int       `json:"stock"` // 可用库存
    LockedStock int       `json:"locked_stock"` // 锁定库存
    SoldStock   int       `json:"sold_stock"` // 已售库存
    Version     int64     `json:"version"` // 乐观锁版本号
    UpdatedAt   time.Time `json:"updated_at"`
}

// InventoryLog 库存变更日志
type InventoryLog struct {
    ID          int64     `json:"id" gorm:"primaryKey"`
    SKUID       int64     `json:"sku_id" gorm:"index"`
    OrderNo     string    `json:"order_no" gorm:"size:50;index"`
    Type        int8      `json:"type"` // 1:扣减 2:增加 3:锁定 4:释放
    Quantity    int       `json:"quantity"`
    BeforeStock int       `json:"before_stock"`
    AfterStock  int       `json:"after_stock"`
    Remark      string    `json:"remark" gorm:"size:255"`
    CreatedAt   time.Time `json:"created_at"`
}
```

### 4.5 购物车服务数据模型

```go
// Cart 购物车表
type Cart struct {
    ID          int64     `json:"id" gorm:"primaryKey"`
    UserID      int64     `json:"user_id" gorm:"index"`
    SKUID       int64     `json:"sku_id"`
    ProductID   int64     `json:"product_id"`
    Quantity    int       `json:"quantity"`
    Selected    bool      `json:"selected" gorm:"default:true"` // 是否选中
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}
```

### 4.6 支付服务数据模型

```go
// Payment 支付订单表
type Payment struct {
    ID              int64     `json:"id" gorm:"primaryKey"`
    PaymentNo       string    `json:"payment_no" gorm:"uniqueIndex;size:50"` // 支付流水号
    OrderNo         string    `json:"order_no" gorm:"index;size:50"` // 业务订单号
    UserID          int64     `json:"user_id" gorm:"index"`
    Amount          int64     `json:"amount"` // 支付金额（分）
    PayType         int8      `json:"pay_type"` // 1:支付宝 2:微信
    PayStatus       int8      `json:"pay_status"` // 1:待支付 2:支付成功 3:支付失败 4:已退款
    ThirdPartyNo    string    `json:"third_party_no" gorm:"size:100"` // 第三方交易号
    ThirdPartyResp  string    `json:"third_party_resp" gorm:"type:text"` // 第三方响应
    PaidAt          *time.Time `json:"paid_at"`
    ExpireAt        time.Time `json:"expire_at"` // 过期时间
    CreatedAt       time.Time `json:"created_at"`
    UpdatedAt       time.Time `json:"updated_at"`
}
```

### 4.7 评价服务数据模型

```go
// Review 商品评价表
type Review struct {
    ID          int64     `json:"id" gorm:"primaryKey"`
    OrderID     int64     `json:"order_id" gorm:"index"`
    ProductID   int64     `json:"product_id" gorm:"index"`
    SKUID       int64     `json:"sku_id"`
    UserID      int64     `json:"user_id" gorm:"index"`
    Rating      int8      `json:"rating"` // 评分 1-5星
    Content     string    `json:"content" gorm:"type:text"`
    Images      string    `json:"images" gorm:"type:text"` // JSON array
    IsAnonymous bool      `json:"is_anonymous" gorm:"default:false"`
    Status      int8      `json:"status" gorm:"default:1"` // 1:待审核 2:已通过 3:已拒绝
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}
```

## 5. 核心流程设计

### 5.1 下单流程

```
1. 用户选择购物车商品 -> Cart Service
2. 创建订单 -> Order Service
3. 订单服务调用库存服务锁定库存 -> Inventory Service
4. 库存锁定成功，订单创建成功
5. 用户支付 -> Payment Service
6. 支付成功回调 -> Order Service
7. 订单服务调用库存服务扣减库存 -> Inventory Service
8. 更新订单状态为已支付
9. 发送通知 -> Notification Service
```

### 5.2 分布式事务处理

采用 **Saga模式** 处理分布式事务：
- 订单创建：TCC（Try-Confirm-Cancel）模式
- 库存扣减：预留-确认-释放机制
- 支付回滚：补偿事务

### 5.3 接口通信

- 同步通信：gRPC (服务间调用)
- 异步通信：RabbitMQ/Kafka (事件驱动)
- API层：RESTful API (对外接口)

## 6. 安全设计

### 6.1 认证授权
- JWT Token认证
- API签名验证
- OAuth2.0授权

### 6.2 数据安全
- 密码加密存储（bcrypt）
- 敏感数据加密（AES）
- HTTPS传输
- SQL注入防护
- XSS防护

### 6.3 限流熔断
- API限流：令牌桶/漏桶算法
- 服务熔断：Hystrix模式
- 降级策略

## 7. 性能优化

### 7.1 缓存策略
- 热点数据缓存（Redis）
- 多级缓存（本地缓存 + Redis）
- 缓存预热
- 缓存穿透/雪崩/击穿防护

### 7.2 数据库优化
- 读写分离
- 分库分表（订单表按时间/用户ID分片）
- 索引优化
- 慢查询监控

### 7.3 消息队列
- 异步处理（订单通知、日志记录）
- 流量削峰（秒杀场景）
- 系统解耦

## 8. 可观测性

### 8.1 日志
- 统一日志格式
- ELK日志聚合
- 日志级别管理

### 8.2 监控
- 服务健康监控
- 业务指标监控
- 告警通知

### 8.3 链路追踪
- Jaeger分布式追踪
- 请求链路分析
- 性能瓶颈定位

## 9. 部署架构

```
┌─────────────────────────────────────────────┐
│              Load Balancer (Nginx)          │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│            API Gateway Cluster              │
│           (Kong/Traefik + Consul)           │
└─────────────────┬───────────────────────────┘
                  │
      ┌───────────┼───────────┐
      │           │           │
┌─────▼─────┐ ┌──▼────┐ ┌────▼─────┐
│ Service A │ │Service│ │ Service C│
│  Cluster  │ │   B   │ │  Cluster │
└───────────┘ └───────┘ └──────────┘
      │           │           │
┌─────▼───────────▼───────────▼─────┐
│         Data Layer                 │
│  MySQL / Redis / MongoDB / MQ      │
└────────────────────────────────────┘
```

## 10. 扩展性设计

- 服务无状态设计，支持水平扩展
- 数据库分片策略
- 缓存集群
- 消息队列集群
- 微服务独立部署、独立扩展