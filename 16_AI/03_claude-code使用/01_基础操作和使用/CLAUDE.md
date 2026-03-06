# Claude 工作规范

> **本文档定义了 Claude 在本项目中的工作规范、行为准则和禁止事项。**
> **Claude 必须严格遵守本文档中的所有规则。**

---

## 核心原则

### 1. 真实性原则 (Truth Principle)
- **绝对禁止编造、虚构或假装完成任务**
- 必须基于实际文件内容和执行结果进行回复
- 不确定时必须明确说明"不确定"或"需要验证"

### 2. 证据原则 (Evidence Principle)
- **每次宣称任务完成，必须提供证据**
- 证据必须包含具体文件路径、命令输出或测试结果
- 所有证据必须记录到 `PROGRESS.md` 中

### 3. 验证原则 (Verification Principle)
- 完成任务后必须执行验收命令验证结果
- 验收命令参考 `DEVELOPMENT_PLAN.md` 中定义的标准
- 验收失败必须修复问题，不能标记为完成

---

## 强制工作流程

### 第一步：开始任何任务前必须执行

**每次对话开始时，或接收到新任务时，必须先执行以下步骤：**

1. **读取项目核心文档，了解当前状态**
   ```
   必须按顺序读取以下文件：
   1. ARCHITECTURE.md  - 了解系统架构和设计
   2. DEVELOPMENT_PLAN.md - 了解开发计划和验收标准
   3. PROGRESS.md - 了解当前进度和已完成任务
   ```

2. **确认当前状态**
   - 当前处于哪个阶段？
   - 哪些任务已完成？
   - 哪些任务正在进行？
   - 是否有遗留问题？

3. **明确本次任务目标**
   - 用户要求做什么？
   - 该任务在开发计划中的位置？
   - 该任务的验收标准是什么？

**禁止**: 在未读取核心文档的情况下直接开始工作

### 第二步：执行任务

1. **创建任务追踪清单**
   - 将任务拆解为具体步骤
   - 在内部维护 checklist

2. **逐步执行任务**
   - 按照 `DEVELOPMENT_PLAN.md` 中的指导执行
   - 遇到问题立即记录

3. **实时验证**
   - 每完成一个步骤，执行相应的验证命令
   - 确保每一步都真实有效

### 第三步：验收与记录

1. **执行验收命令**
   ```bash
   # 参考 DEVELOPMENT_PLAN.md 中对应任务的验收命令
   # 必须执行所有验收命令并提供输出结果
   ```

2. **收集证据**
   - 命令执行输出
   - 文件路径和内容
   - 测试结果截图/日志
   - 服务运行状态

3. **更新 PROGRESS.md**
   - 标记任务完成状态 ✅
   - 添加证据链接
   - 更新阶段进度
   - 记录到"完成任务记录"中

4. **向用户报告**
   - 简要说明完成的任务
   - 提供证据文件链接（相对路径）
   - 说明验收结果

---

## 严格禁止事项

### 🚫 禁止 1: 虚假完成
**描述**: 禁止在未实际完成任务的情况下声称任务完成

**错误示例**:
```
❌ "我已经创建了用户服务，代码已经完成。"
（实际上没有创建任何文件）
```

**正确示例**:
```
✅ "我已经创建了用户服务，文件位于 services/user/internal/handler/user.go。
   验收命令执行结果：
   $ go build ./services/user/...
   [输出结果]

   证据已记录到 PROGRESS.md:125"
```

### 🚫 禁止 2: 省略验收步骤
**描述**: 禁止跳过 DEVELOPMENT_PLAN.md 中定义的验收命令

**错误示例**:
```
❌ "用户服务已完成，应该没问题。"
（没有执行任何验收命令）
```

**正确示例**:
```
✅ "用户服务已完成，验收结果如下：

   1. 服务编译检查:
   $ go build ./services/user/...
   [成功输出]

   2. 单元测试:
   $ cd services/user && go test -v ./...
   PASS
   coverage: 82.5% of statements

   3. 服务注册检查:
   $ curl http://localhost:8500/v1/catalog/service/user-service
   [返回服务信息]"
```

### 🚫 禁止 3: 不读取项目文档
**描述**: 禁止在未读取 ARCHITECTURE.md、DEVELOPMENT_PLAN.md、PROGRESS.md 的情况下开始任务

**错误示例**:
```
❌ 用户："帮我创建订单服务"
   Claude："好的，我现在开始创建订单服务..."
   （没有先读取文档了解架构和当前进度）
```

**正确示例**:
```
✅ 用户："帮我创建订单服务"
   Claude："让我先读取项目文档了解当前状态。

   [读取 ARCHITECTURE.md]
   [读取 DEVELOPMENT_PLAN.md]
   [读取PROGRESS.md]

   当前状态：
   - 项目处于阶段3：购物车与订单开发
   - 购物车服务已完成
   - 订单服务任务清单参考 DEVELOPMENT_PLAN.md 第3.2节

   我将按照开发计划创建订单服务..."
```

### 🚫 禁止 4: 不更新 PROGRESS.md
**描述**: 禁止完成任务后不更新 PROGRESS.md

**错误示例**:
```
❌ "用户服务已完成，你现在可以运行服务了。"
（PROGRESS.md 没有任何更新）
```

**正确示例**:
```
✅ "用户服务已完成，我已更新 PROGRESS.md：

   - 阶段1任务1.1标记为完成 ✅
   - 添加证据链接到 services/user/
   - 更新阶段进度为 33% (1/3)
   - 在完成任务记录中添加详细信息

   请查看 PROGRESS.md:125 获取完整记录。"
```

### 🚫 禁止 5: 编造文件或命令输出
**描述**: 禁止虚构文件内容或命令执行结果

**错误示例**:
```
❌ "我检查了日志，服务运行正常。"
（实际上没有读取任何日志文件）
```

**正确示例**:
```
✅ "我读取了日志文件 logs/user-service.log:

   [显示实际日志内容]

   从日志中可以看到服务成功启动在端口 9001。"
```

### 🚫 禁止 6: 跳过依赖检查
**描述**: 禁止在依赖未满足的情况下执行任务

**错误示例**:
```
❌ 用户："创建订单服务"
   Claude："好的，我现在创建订单服务..."
   （没有检查用户服务、库存服务等依赖是否完成）
```

**正确示例**:
```
✅ 用户："创建订单服务"
   Claude："让我先检查订单服务的依赖项：

   [检查 PROGRESS.md]

   发现问题：
   - 用户服务：✅ 已完成
   - 库存服务：❌ 未完成
   - 购物车服务：❌ 未完成

   订单服务依赖库存服务和购物车服务，建议先完成这些服务。
   是否继续创建订单服务框架？"
```

### 🚫 禁止 7: 不提供具体路径
**描述**: 禁止使用模糊的描述，必须提供具体文件路径

**错误示例**:
```
❌ "代码已经写好了，在用户服务的 handler 文件里。"
```

**正确示例**:
```
✅ "代码已写入以下文件：
   - services/user/internal/handler/user.go:25-87
   - services/user/internal/service/user_service.go:15-120
   - services/user/cmd/main.go:10-45"
```

### 🚫 禁止 8: 不执行测试
**描述**: 禁止声称功能完成但不执行测试

**错误示例**:
```
❌ "功能已完成，应该能正常工作。"
```

**正确示例**:
```
✅ "功能已完成，测试结果：

   单元测试：
   $ go test -v ./services/user/internal/service/...
   PASS
   ok      services/user/internal/service    0.523s

   覆盖率：
   $ go test -cover ./services/user/...
   coverage: 85.2% of statements"
```

---

## 证据要求规范

### 必须提供的证据类型

#### 1. 代码文件证据
```
示例：
- 文件路径: services/user/internal/handler/user.go
- 文件路径: services/user/internal/service/user_service.go
- 关键代码行: services/user/internal/handler/user.go:25-87
```

#### 2. 编译/构建证据
```
示例：
$ go build ./services/user/...
[无错误输出或成功信息]
```

#### 3. 测试证据
```
示例：
$ go test -v ./services/user/...
=== RUN   TestUserRegister
--- PASS: TestUserRegister (0.01s)
=== RUN   TestUserLogin
--- PASS: TestUserLogin (0.01s)
PASS
coverage: 85.2% of statements
ok      services/user/internal/handler    0.523s
```

#### 4. 服务运行证据
```
示例：
$ docker-compose ps
NAME                STATUS
shop-mysql         Up 2 minutes
shop-redis         Up 2 minutes

$ curl http://localhost:8500/v1/catalog/service/user-service
[返回服务注册信息]
```

#### 5. API 测试证据
```
示例：
$ curl -X POST http://localhost:8080/api/v1/users/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123456","email":"test@example.com"}'

{"code":0,"message":"success","data":{"user_id":1}}
```

#### 6. 数据库证据
```
示例：
$ docker exec -it shop-mysql mysql -uroot -p shop -e "SELECT COUNT(*) FROM users;"
+----------+
| COUNT(*) |
+----------+
|       15 |
+----------+
```

---

## PROGRESS.md 更新规范

### 每次完成任务必须更新的内容

1. **任务清单状态**
   ```markdown
   ##### 1.1 用户服务 (User Service)
   - [x] 创建用户服务项目结构
   - [x] 实现数据模型和数据库迁移
   - [x] 实现 gRPC 接口：用户注册

   **状态**: ✅ 已完成
   **证据链接**:
   - services/user/internal/handler/user.go
   - services/user/internal/service/user_service.go
   - 测试结果: docs/test_results/user_service_test.log
   ```

2. **完成任务记录**
   ```markdown
   ### 2025-12-20

   #### ✅ 用户服务开发完成
   - **任务**: 实现用户服务的注册、登录、信息管理功能
   - **完成时间**: 2025-12-20 14:30
   - **证据文件**:
     - [services/user/internal/handler/user.go](services/user/internal/handler/user.go)
     - [services/user/internal/service/user_service.go](services/user/internal/service/user_service.go)
     - [测试结果](docs/test_results/user_service_test.log)
   - **验收结果**:
     - 编译成功
     - 测试覆盖率: 85.2%
     - 服务成功注册到 Consul
     - gRPC 接口响应正常
   ```

3. **阶段进度更新**
   ```markdown
   ### 阶段 1: 核心服务开发 - 用户与认证

   **状态**: 🚧 进行中
   **进度**: 33% (1/3)
   ```

4. **当前状态更新**
   ```markdown
   ### 正在进行的任务
   - 认证服务开发（阶段1任务1.2）

   ### 已完成的任务
   - 用户服务开发（阶段1任务1.1）
   ```

---

## 响应模板

### 开始新任务时的标准响应

```
我将开始[任务名称]。首先让我读取项目文档了解当前状态。

[读取 ARCHITECTURE.md]
[读取 DEVELOPMENT_PLAN.md]
[读取 PROGRESS.md]

当前状态确认：
- 当前阶段：[阶段名称]
- 已完成任务：[列表]
- 正在进行：[列表]
- 待完成：[列表]

本次任务：
- 任务名称：[名称]
- 所属阶段：[阶段X任务X.X]
- 验收标准：[参考 DEVELOPMENT_PLAN.md X.X 节]

开始执行...
```

### 完成任务时的标准响应

```
[任务名称]已完成。

验收结果：

1. [验收项1]
   $ [验收命令]
   [命令输出]

2. [验收项2]
   $ [验收命令]
   [命令输出]

证据文件：
- [文件路径1]
- [文件路径2]
- [测试结果路径]

我已更新 PROGRESS.md：
- 标记任务[X.X]为完成 ✅
- 添加证据链接
- 更新阶段进度为 [X]%
- 记录到完成任务列表

详细记录请查看：PROGRESS.md:[行号]
```

---

## 异常处理规范

### 遇到错误时

1. **立即停止**：不要继续执行后续步骤
2. **记录问题**：详细描述错误信息
3. **更新文档**：在 PROGRESS.md 的"问题与风险记录"中添加
4. **询问用户**：报告问题并询问如何处理

**示例**：
```
在执行[任务]时遇到错误：

错误信息：
[完整错误输出]

问题分析：
[分析错误原因]

我已将此问题记录到 PROGRESS.md 的"问题与风险记录"表中。

建议处理方案：
1. [方案1]
2. [方案2]

请问应该如何处理？
```

### 发现文档不一致时

1. **停止执行**
2. **报告不一致**：说明具体哪里不一致
3. **请求确认**：询问用户应以哪个文档为准

---

## 代码质量要求

### 必须遵守的编码规范

1. **Go 代码规范**
   - 遵循 Effective Go
   - 通过 golint 检查
   - 通过 go vet 检查

2. **错误处理**
   - 所有错误必须处理
   - 不能使用 panic（除非必要）
   - 错误信息必须清晰

3. **日志规范**
   - 使用统一的日志格式
   - 关键操作必须记录日志
   - 日志级别使用正确（Debug/Info/Warn/Error）

4. **测试要求**
   - 单元测试覆盖率 > 70%
   - 关键路径必须有测试
   - 测试用例清晰明确

---

## 性能要求

1. **数据库**
   - 必须创建必要的索引
   - 避免 N+1 查询
   - 慢查询必须优化

2. **缓存**
   - 热点数据必须缓存
   - 缓存过期时间合理设置
   - 处理缓存穿透/击穿/雪崩

3. **并发**
   - 注意资源竞争
   - 使用乐观锁或悲观锁
   - 防止死锁

---

## 安全要求

1. **输入验证**
   - 所有外部输入必须验证
   - 防止 SQL 注入
   - 防止 XSS 攻击

2. **身份认证**
   - Token 必须验证
   - 密码必须加密存储
   - 敏感数据必须加密传输

3. **权限控制**
   - 实现 RBAC
   - 接口必须鉴权
   - 数据访问必须授权

---

## 检查清单

在宣称任务完成前，必须确认以下所有项：

- [ ] 已读取 ARCHITECTURE.md
- [ ] 已读取 DEVELOPMENT_PLAN.md
- [ ] 已读取 PROGRESS.md
- [ ] 已执行所有验收命令
- [ ] 所有验收命令通过
- [ ] 已收集所有证据文件
- [ ] 已更新 PROGRESS.md 任务状态
- [ ] 已更新 PROGRESS.md 完成记录
- [ ] 已更新 PROGRESS.md 阶段进度
- [ ] 证据链接正确可访问
- [ ] 代码通过 lint 检查
- [ ] 测试覆盖率达标
- [ ] 已向用户提供证据链接

**只有当以上所有项都确认后，才能声称任务完成！**

---

## 总结

### 核心要求（必须记住）

1. **每次开始任务前，必须先读取 ARCHITECTURE.md、DEVELOPMENT_PLAN.md、PROGRESS.md**
2. **每次宣称完成，必须提供证据并记录到 PROGRESS.md**
3. **必须执行验收命令并提供输出结果**
4. **禁止虚构、编造任何内容**
5. **遇到问题立即停止并报告**

### 记住这句话

**"没有证据，就没有完成。没有验收，就不能声称成功。"**

---

**此文档是 Claude 在本项目中工作的最高准则，任何情况下都不得违反。**
