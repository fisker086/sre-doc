# 电商系统开发计划

## 项目分阶段开发计划

本文档按照优先级和依赖关系，将整个电商系统拆解为多个阶段，每个阶段包含具体的任务、验收标准和验收命令。

---

## 阶段 0: 项目初始化与基础设施

### 目标
搭建项目基础框架和开发环境

### 任务列表

#### 0.1 项目结构初始化
- [ ] 创建项目根目录结构
- [ ] 初始化 Go Modules
- [ ] 创建各微服务目录
- [ ] 配置 .gitignore

**验收命令**
```bash
# 验证目录结构
tree -L 2 .

# 验证 Go Modules
go mod verify
```

**验收标准**
- 目录结构符合微服务架构规范
- go.mod 和 go.sum 文件正常
- 各服务目录清晰划分

#### 0.2 开发环境 Docker Compose 配置
- [ ] 配置 MySQL 容器
- [ ] 配置 Redis 容器
- [ ] 配置 MongoDB 容器
- [ ] 配置 RabbitMQ 容器
- [ ] 配置 Consul 容器
- [ ] 配置 Jaeger 容器

**验收命令**
```bash
# 启动所有基础设施
docker-compose up -d

# 验证容器状态
docker-compose ps

# 验证 MySQL 连接
docker exec -it shop-mysql mysql -uroot -p -e "SELECT VERSION();"

# 验证 Redis 连接
docker exec -it shop-redis redis-cli ping

# 验证 Consul UI
curl http://localhost:8500/v1/status/leader
```

**验收标准**
- 所有容器正常运行
- 各服务端口正常监听
- 可通过命令行/UI访问各基础服务

#### 0.3 公共库开发
- [ ] 创建 common 库（配置、日志、错误码）
- [ ] 创建 gRPC proto 文件
- [ ] 生成 protobuf 代码
- [ ] 封装数据库连接池
- [ ] 封装 Redis 客户端
- [ ] 封装 JWT 工具

**验收命令**
```bash
# 验证 proto 文件生成
ls -la pkg/proto/**/*.pb.go

# 运行公共库单元测试
go test -v ./pkg/common/...

# 验证编译
go build ./pkg/...
```

**验收标准**
- proto 文件编译无错误
- 公共库测试覆盖率 > 80%
- 代码通过 golint 检查

---

## 阶段 1: 核心服务开发 - 用户与认证

### 目标
实现用户注册、登录、认证授权功能

### 任务列表

#### 1.1 用户服务 (User Service)
- [ ] 创建用户服务项目结构
- [ ] 实现数据模型和数据库迁移
- [ ] 实现 gRPC 接口：用户注册
- [ ] 实现 gRPC 接口：用户信息查询/更新
- [ ] 实现用户地址管理接口
- [ ] 编写单元测试

**验收命令**
```bash
# 启动用户服务
cd services/user && go run cmd/main.go

# 验证服务注册到 Consul
curl http://localhost:8500/v1/catalog/service/user-service

# 运行单元测试
cd services/user && go test -v ./...

# 使用 grpcurl 测试接口
grpcurl -plaintext -d '{"username":"test","password":"123456","email":"test@example.com"}' \
  localhost:9001 user.UserService/Register
```

**验收标准**
- 服务成功注册到 Consul
- 所有接口正常响应
- 单元测试覆盖率 > 70%
- 密码正确加密存储

#### 1.2 认证服务 (Auth Service)
- [ ] 创建认证服务项目结构
- [ ] 实现登录接口（用户名/密码）
- [ ] 实现 JWT Token 生成
- [ ] 实现 Token 验证中间件
- [ ] 实现刷新 Token 接口
- [ ] 集成 Redis 存储 Token 黑名单
- [ ] 编写单元测试

**验收命令**
```bash
# 启动认证服务
cd services/auth && go run cmd/main.go

# 测试登录接口
grpcurl -plaintext -d '{"username":"test","password":"123456"}' \
  localhost:9002 auth.AuthService/Login

# 测试 Token 验证
grpcurl -plaintext -d '{"token":"eyJhbGc..."}' \
  localhost:9002 auth.AuthService/ValidateToken

# 运行测试
cd services/auth && go test -v ./...
```

**验收标准**
- 登录成功返回有效 JWT Token
- Token 验证正确识别有效/无效 Token
- Token 黑名单机制正常工作
- 测试覆盖率 > 70%

#### 1.3 API 网关集成（用户模块）
- [ ] 配置 API 网关路由
- [ ] 集成认证中间件
- [ ] 实现 RESTful API 层（用户注册/登录）
- [ ] 配置限流规则
- [ ] 编写 API 文档

**验收命令**
```bash
# 启动 API 网关
cd gateway && go run main.go

# 测试用户注册 API
curl -X POST http://localhost:8080/api/v1/users/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123456","email":"test@example.com"}'

# 测试用户登录 API
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123456"}'

# 测试需要认证的接口
TOKEN="eyJhbGc..."
curl -X GET http://localhost:8080/api/v1/users/profile \
  -H "Authorization: Bearer $TOKEN"
```

**验收标准**
- 所有 RESTful API 正常响应
- 认证中间件正确拦截未授权请求
- API 限流正常工作
- API 文档完整准确

---

## 阶段 2: 核心服务开发 - 商品与库存

### 目标
实现商品管理、商品搜索、库存管理功能

### 任务列表

#### 2.1 商品服务 (Product Service)
- [ ] 创建商品服务项目结构
- [ ] 实现数据模型和数据库迁移
- [ ] 实现商品分类管理接口
- [ ] 实现商品 CRUD 接口
- [ ] 实现 SKU 管理接口
- [ ] 实现商品列表查询（分页、排序、筛选）
- [ ] 集成 Redis 缓存热点商品
- [ ] 编写单元测试

**验收命令**
```bash
# 启动商品服务
cd services/product && go run cmd/main.go

# 测试创建商品分类
grpcurl -plaintext -d '{"name":"电子产品","level":1}' \
  localhost:9003 product.ProductService/CreateCategory

# 测试创建商品
grpcurl -plaintext -d '{"category_id":1,"name":"iPhone 15","price":699900}' \
  localhost:9003 product.ProductService/CreateProduct

# 测试商品查询
grpcurl -plaintext -d '{"product_id":1}' \
  localhost:9003 product.ProductService/GetProduct

# 运行测试
cd services/product && go test -v ./...
```

**验收标准**
- 商品 CRUD 功能正常
- 商品列表支持分页和筛选
- 热点商品命中缓存
- 测试覆盖率 > 70%

#### 2.2 Elasticsearch 商品搜索集成
- [ ] 配置 Elasticsearch 容器
- [ ] 创建商品索引和映射
- [ ] 实现商品数据同步（MySQL -> ES）
- [ ] 实现商品搜索接口（全文搜索、过滤、排序）
- [ ] 实现搜索结果高亮
- [ ] 编写测试

**验收命令**
```bash
# 启动 Elasticsearch
docker-compose up -d elasticsearch

# 验证索引创建
curl -X GET "localhost:9200/products/_mapping?pretty"

# 同步商品数据
cd services/product && go run cmd/sync_es.go

# 测试搜索接口
grpcurl -plaintext -d '{"keyword":"iPhone","page":1,"page_size":10}' \
  localhost:9003 product.ProductService/SearchProducts
```

**验收标准**
- 商品数据成功同步到 ES
- 搜索功能正常，返回相关结果
- 支持多条件过滤和排序
- 搜索性能 < 100ms

#### 2.3 库存服务 (Inventory Service)
- [ ] 创建库存服务项目结构
- [ ] 实现数据模型和数据库迁移
- [ ] 实现库存查询接口
- [ ] 实现库存扣减接口（乐观锁）
- [ ] 实现库存预留/释放接口（TCC模式）
- [ ] 实现库存变更日志记录
- [ ] 集成 Redis 缓存库存信息
- [ ] 编写单元测试

**验收命令**
```bash
# 启动库存服务
cd services/inventory && go run cmd/main.go

# 测试库存查询
grpcurl -plaintext -d '{"sku_id":1}' \
  localhost:9004 inventory.InventoryService/GetStock

# 测试库存预留
grpcurl -plaintext -d '{"sku_id":1,"quantity":2,"order_no":"ORD123"}' \
  localhost:9004 inventory.InventoryService/ReserveStock

# 测试库存扣减
grpcurl -plaintext -d '{"sku_id":1,"quantity":2,"order_no":"ORD123"}' \
  localhost:9004 inventory.InventoryService/DeductStock

# 运行并发测试（验证乐观锁）
cd services/inventory && go test -v -run TestConcurrentDeduct ./...
```

**验收标准**
- 库存扣减支持乐观锁，无超卖问题
- TCC 模式预留/释放机制正常
- 库存变更日志完整记录
- 并发测试通过
- 测试覆盖率 > 70%

#### 2.4 API 网关集成（商品模块）
- [ ] 实现商品列表 API
- [ ] 实现商品详情 API
- [ ] 实现商品搜索 API
- [ ] 实现分类列表 API
- [ ] 更新 API 文档

**验收命令**
```bash
# 测试商品列表
curl "http://localhost:8080/api/v1/products?page=1&page_size=20"

# 测试商品详情
curl "http://localhost:8080/api/v1/products/1"

# 测试商品搜索
curl "http://localhost:8080/api/v1/products/search?keyword=iPhone&page=1"

# 测试分类列表
curl "http://localhost:8080/api/v1/categories"
```

**验收标准**
- 所有商品相关 API 正常响应
- API 响应时间 < 200ms
- API 文档完整

---

## 阶段 3: 核心服务开发 - 购物车与订单

### 目标
实现购物车管理和订单流程

### 任务列表

#### 3.1 购物车服务 (Cart Service)
- [ ] 创建购物车服务项目结构
- [ ] 实现数据模型（Redis 存储）
- [ ] 实现添加商品到购物车接口
- [ ] 实现购物车列表查询接口
- [ ] 实现更新购物车商品数量接口
- [ ] 实现删除购物车商品接口
- [ ] 实现清空购物车接口
- [ ] 实现选中/取消选中商品接口
- [ ] 编写单元测试

**验收命令**
```bash
# 启动购物车服务
cd services/cart && go run cmd/main.go

# 测试添加商品
grpcurl -plaintext -d '{"user_id":1,"sku_id":1,"quantity":2}' \
  localhost:9005 cart.CartService/AddItem

# 测试查询购物车
grpcurl -plaintext -d '{"user_id":1}' \
  localhost:9005 cart.CartService/GetCart

# 测试更新数量
grpcurl -plaintext -d '{"user_id":1,"sku_id":1,"quantity":5}' \
  localhost:9005 cart.CartService/UpdateQuantity

# 运行测试
cd services/cart && go test -v ./...
```

**验收标准**
- 购物车所有操作正常
- 购物车数据实时更新
- 支持多 SKU 管理
- 测试覆盖率 > 70%

#### 3.2 订单服务 (Order Service)
- [ ] 创建订单服务项目结构
- [ ] 实现数据模型和数据库迁移
- [ ] 实现创建订单接口（Saga 模式）
  - 验证商品信息
  - 验证库存
  - 锁定库存（调用库存服务）
  - 创建订单
  - 清空购物车（调用购物车服务）
- [ ] 实现订单列表查询接口
- [ ] 实现订单详情查询接口
- [ ] 实现取消订单接口（释放库存）
- [ ] 实现订单状态更新接口
- [ ] 集成 RabbitMQ 处理订单事件
- [ ] 编写单元测试和集成测试

**验收命令**
```bash
# 启动订单服务
cd services/order && go run cmd/main.go

# 测试创建订单
grpcurl -plaintext -d '{
  "user_id":1,
  "items":[{"sku_id":1,"quantity":2}],
  "address_id":1
}' localhost:9006 order.OrderService/CreateOrder

# 测试订单列表
grpcurl -plaintext -d '{"user_id":1,"page":1,"page_size":10}' \
  localhost:9006 order.OrderService/ListOrders

# 测试订单详情
grpcurl -plaintext -d '{"order_id":1}' \
  localhost:9006 order.OrderService/GetOrder

# 测试取消订单
grpcurl -plaintext -d '{"order_id":1,"user_id":1}' \
  localhost:9006 order.OrderService/CancelOrder

# 运行集成测试（验证分布式事务）
cd services/order && go test -v -tags=integration ./...
```

**验收标准**
- 订单创建流程完整（库存锁定、订单生成、购物车清空）
- 订单取消正确释放库存
- 分布式事务补偿机制正常
- 订单状态流转正确
- 集成测试通过

#### 3.3 API 网关集成（购物车与订单模块）
- [ ] 实现购物车相关 API
- [ ] 实现订单相关 API
- [ ] 更新 API 文档

**验收命令**
```bash
TOKEN="eyJhbGc..."

# 测试添加购物车
curl -X POST http://localhost:8080/api/v1/cart/items \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sku_id":1,"quantity":2}'

# 测试查询购物车
curl -X GET http://localhost:8080/api/v1/cart \
  -H "Authorization: Bearer $TOKEN"

# 测试创建订单
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"sku_id":1,"quantity":2}],"address_id":1}'

# 测试订单列表
curl -X GET "http://localhost:8080/api/v1/orders?page=1&page_size=10" \
  -H "Authorization: Bearer $TOKEN"
```

**验收标准**
- 购物车和订单 API 正常工作
- 需要认证的接口正确验证 Token
- API 文档完整

---

## 阶段 4: 支付与通知服务

### 目标
实现支付流程和消息通知功能

### 任务列表

#### 4.1 支付服务 (Payment Service)
- [ ] 创建支付服务项目结构
- [ ] 实现数据模型和数据库迁移
- [ ] 实现支付订单创建接口
- [ ] 集成支付宝沙箱环境
- [ ] 实现支付宝支付接口
- [ ] 实现支付回调处理接口
- [ ] 实现支付状态查询接口
- [ ] 实现退款接口
- [ ] 发送消息到订单服务（支付成功事件）
- [ ] 编写单元测试

**验收命令**
```bash
# 启动支付服务
cd services/payment && go run cmd/main.go

# 测试创建支付订单
grpcurl -plaintext -d '{"order_no":"ORD123","amount":199900,"pay_type":1}' \
  localhost:9007 payment.PaymentService/CreatePayment

# 测试支付回调（模拟）
curl -X POST http://localhost:9007/callback/alipay \
  -d "out_trade_no=PAY123&trade_no=2024xxx&trade_status=TRADE_SUCCESS"

# 测试支付查询
grpcurl -plaintext -d '{"payment_no":"PAY123"}' \
  localhost:9007 payment.PaymentService/QueryPayment

# 运行测试
cd services/payment && go test -v ./...
```

**验收标准**
- 支付订单创建成功
- 支付宝支付接口对接正常（沙箱测试通过）
- 支付回调正确处理并通知订单服务
- 支付流水完整记录
- 测试覆盖率 > 70%

#### 4.2 通知服务 (Notification Service)
- [ ] 创建通知服务项目结构
- [ ] 实现邮件通知功能
- [ ] 实现短信通知功能（集成阿里云SMS）
- [ ] 实现站内消息功能
- [ ] 订阅 RabbitMQ 订单事件
- [ ] 实现通知模板管理
- [ ] 编写单元测试

**验收命令**
```bash
# 启动通知服务
cd services/notification && go run cmd/main.go

# 测试发送邮件
grpcurl -plaintext -d '{
  "type":"email",
  "recipient":"test@example.com",
  "subject":"订单通知",
  "content":"您的订单已创建"
}' localhost:9008 notification.NotificationService/Send

# 验证消息队列消费（查看日志）
docker logs -f shop-notification

# 运行测试
cd services/notification && go test -v ./...
```

**验收标准**
- 邮件发送成功
- 短信发送成功（或模拟成功）
- 消息队列正确消费订单事件
- 通知记录完整保存

#### 4.3 订单支付流程集成测试
- [ ] 编写端到端测试脚本
- [ ] 测试完整支付流程

**验收命令**
```bash
# 运行端到端测试
cd tests/e2e && go test -v -run TestOrderPaymentFlow

# 手动测试流程
# 1. 创建订单
TOKEN="eyJhbGc..."
ORDER_RESP=$(curl -X POST http://localhost:8080/api/v1/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"sku_id":1,"quantity":1}],"address_id":1}')

ORDER_NO=$(echo $ORDER_RESP | jq -r '.data.order_no')

# 2. 发起支付
PAYMENT_RESP=$(curl -X POST http://localhost:8080/api/v1/payments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"order_no\":\"$ORDER_NO\",\"pay_type\":1}")

# 3. 查询订单状态
curl -X GET "http://localhost:8080/api/v1/orders/$ORDER_NO" \
  -H "Authorization: Bearer $TOKEN"
```

**验收标准**
- 完整支付流程无错误
- 支付成功后订单状态正确更新
- 库存正确扣减
- 用户收到通知

---

## 阶段 5: 评价与其他功能

### 任务列表

#### 5.1 评价服务 (Review Service)
- [ ] 创建评价服务项目结构
- [ ] 实现数据模型（MongoDB）
- [ ] 实现提交评价接口
- [ ] 实现评价列表查询接口
- [ ] 实现评价审核接口
- [ ] 实现评价统计接口
- [ ] 编写单元测试

**验收命令**
```bash
# 启动评价服务
cd services/review && go run cmd/main.go

# 测试提交评价
grpcurl -plaintext -d '{
  "order_id":1,
  "product_id":1,
  "user_id":1,
  "rating":5,
  "content":"非常好"
}' localhost:9009 review.ReviewService/CreateReview

# 测试查询评价
grpcurl -plaintext -d '{"product_id":1,"page":1,"page_size":10}' \
  localhost:9009 review.ReviewService/ListReviews

# 运行测试
cd services/review && go test -v ./...
```

**验收标准**
- 评价功能正常工作
- 支持图片上传
- 评价统计准确

#### 5.2 API 网关集成（评价模块）
- [ ] 实现评价相关 API
- [ ] 更新 API 文档

---

## 阶段 6: 监控、日志与部署

### 任务列表

#### 6.1 监控系统搭建
- [ ] 配置 Prometheus 采集各服务指标
- [ ] 配置 Grafana 仪表盘
- [ ] 配置告警规则
- [ ] 实现自定义业务指标

**验收命令**
```bash
# 启动监控栈
docker-compose up -d prometheus grafana

# 验证 Prometheus
curl http://localhost:9090/api/v1/targets

# 验证 Grafana（访问 UI）
open http://localhost:3000

# 验证服务指标暴露
curl http://localhost:9001/metrics  # user-service
```

**验收标准**
- Prometheus 正常采集所有服务指标
- Grafana 仪表盘展示完整
- 告警规则正常触发

#### 6.2 日志系统集成
- [ ] 配置 ELK Stack
- [ ] 配置各服务日志格式
- [ ] 实现日志聚合
- [ ] 配置 Kibana 仪表盘

**验收命令**
```bash
# 启动 ELK Stack
docker-compose up -d elasticsearch logstash kibana

# 验证 Kibana
open http://localhost:5601

# 验证日志收集
curl -X GET "localhost:9200/_cat/indices?v" | grep logs
```

**验收标准**
- 所有服务日志成功收集到 Elasticsearch
- Kibana 可正常查询日志
- 日志格式统一

#### 6.3 链路追踪集成
- [ ] 各服务集成 Jaeger SDK
- [ ] 配置 Jaeger Collector
- [ ] 验证链路追踪

**验收命令**
```bash
# 验证 Jaeger UI
open http://localhost:16686

# 发起测试请求
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"sku_id":1,"quantity":1}],"address_id":1}'

# 在 Jaeger UI 中查看完整调用链
```

**验收标准**
- 可在 Jaeger UI 查看完整请求链路
- 追踪数据包含所有微服务调用
- 性能瓶颈可识别

#### 6.4 Kubernetes 部署
- [ ] 编写各服务 Dockerfile
- [ ] 编写 Kubernetes Deployment 配置
- [ ] 编写 Kubernetes Service 配置
- [ ] 编写 Ingress 配置
- [ ] 配置 ConfigMap 和 Secret
- [ ] 配置 HPA（水平自动伸缩）

**验收命令**
```bash
# 构建镜像
make docker-build

# 部署到 Kubernetes
kubectl apply -f k8s/

# 验证部署
kubectl get pods -n shop
kubectl get services -n shop
kubectl get ingress -n shop

# 验证服务可访问
curl http://shop.example.com/api/v1/products
```

**验收标准**
- 所有服务成功部署到 K8s
- 服务间通信正常
- Ingress 正常转发请求
- HPA 自动伸缩正常

---

## 阶段 7: 性能优化与压力测试

### 任务列表

#### 7.1 性能优化
- [ ] 实现多级缓存策略
- [ ] 优化数据库查询（索引、慢查询）
- [ ] 实现数据库读写分离
- [ ] 实现消息队列削峰
- [ ] 优化 gRPC 连接池

**验收命令**
```bash
# 慢查询分析
mysql -uroot -p -e "SELECT * FROM mysql.slow_log;"

# 缓存命中率监控
redis-cli INFO stats | grep hit

# 性能基准测试
go test -bench=. -benchmem ./services/product/...
```

**验收标准**
- 缓存命中率 > 80%
- 慢查询优化完成
- API 响应时间 < 100ms (P99)

#### 7.2 压力测试
- [ ] 编写压测脚本（使用 k6 / wrk）
- [ ] 对关键接口进行压测
- [ ] 生成压测报告

**验收命令**
```bash
# 商品列表接口压测
k6 run tests/load/product_list.js

# 下单流程压测
k6 run tests/load/order_create.js

# 支付接口压测
k6 run tests/load/payment.js
```

**验收标准**
- 商品列表接口 QPS > 1000
- 下单接口 QPS > 500
- 系统无明显瓶颈

---

## 阶段 8: 文档与交付

### 任务列表

#### 8.1 文档完善
- [ ] 完善 API 文档（Swagger/OpenAPI）
- [ ] 编写部署文档
- [ ] 编写运维手册
- [ ] 编写故障排查指南

**验收命令**
```bash
# 生成 API 文档
make docs

# 验证 Swagger UI
open http://localhost:8080/swagger/index.html
```

**验收标准**
- API 文档完整准确
- 部署文档清晰可执行
- 运维手册覆盖常见场景

#### 8.2 最终验收
- [ ] 完整功能回归测试
- [ ] 安全扫描
- [ ] 性能测试报告
- [ ] 代码质量检查

**验收命令**
```bash
# 运行所有测试
make test-all

# 代码覆盖率报告
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# 安全扫描
gosec ./...

# 代码质量
golangci-lint run ./...
```

**验收标准**
- 所有测试通过
- 测试覆盖率 > 70%
- 无严重安全漏洞
- 代码质量符合规范

---

## 常用验收命令汇总

### 服务健康检查
```bash
# 检查所有服务状态
curl http://localhost:8500/v1/health/state/passing

# 检查特定服务
curl http://localhost:8500/v1/health/service/user-service
```

### 数据库检查
```bash
# 检查数据库连接
docker exec -it shop-mysql mysql -uroot -p -e "SHOW DATABASES;"

# 查看表结构
docker exec -it shop-mysql mysql -uroot -p shop -e "SHOW TABLES;"
```

### 日志查看
```bash
# 查看服务日志
docker logs -f shop-user-service

# 查看最近100条日志
docker logs --tail 100 shop-order-service
```

### 性能监控
```bash
# CPU 和内存使用
docker stats

# Redis 监控
docker exec -it shop-redis redis-cli INFO stats
```

---

## 开发规范

### Git 提交规范
```
feat: 新功能
fix: 修复bug
docs: 文档更新
style: 代码格式调整
refactor: 重构
test: 测试相关
chore: 构建/工具相关
```

### 分支管理
- `main`: 主分支，稳定版本
- `develop`: 开发分支
- `feature/*`: 功能分支
- `bugfix/*`: 修复分支
- `release/*`: 发布分支

### Code Review 检查点
- [ ] 代码符合 Go 编码规范
- [ ] 单元测试覆盖关键逻辑
- [ ] 错误处理完整
- [ ] 日志记录合理
- [ ] 无明显性能问题
- [ ] 无安全漏洞

---

## 项目里程碑

- **里程碑 1**: 完成阶段 0-1（基础设施 + 用户认证） - 预计 2 周
- **里程碑 2**: 完成阶段 2（商品与库存） - 预计 2 周
- **里程碑 3**: 完成阶段 3（购物车与订单） - 预计 2 周
- **里程碑 4**: 完成阶段 4-5（支付、通知、评价） - 预计 2 周
- **里程碑 5**: 完成阶段 6-8（监控、优化、文档） - 预计 2 周

**总计**: 约 10 周完成核心功能开发和部署
