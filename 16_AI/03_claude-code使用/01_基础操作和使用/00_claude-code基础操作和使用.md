# 基础操作和使用

## 安装

### 1、MAC

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

### 2、Win

```bash
# 安装 claude code
irm https://claude.ai/install.ps1 | iex

# 检查安装是否成功
claude --version
```

## 一、常用命令

### 1、claude --help

>常用参数：
>
>- -p：非交互式输出
>- -c:继续最近对话
>- --model：指定模型
>- --output-format json  导出json结构化结果

![image-20251217235806365](https://picture-bed.gmbaifa.online/PicGo/image-20251217235806365.png)

### 2、内部命令

#### 1./status

>解释：状态，查看当前CC的状态

![image-20251218000649002](https://picture-bed.gmbaifa.online/PicGo/image-20251218000649002.png)

#### 2./doctor

>解释：检测，检查CC的安装状态

![image-20251218000614049](https://picture-bed.gmbaifa.online/PicGo/image-20251218000614049.png)

#### 3.其他

| 命令           | 解释                                                         |
| -------------- | ------------------------------------------------------------ |
| /clear         | 清空上下文，想要不带记忆重新开始对话，比如当前聊天内容不利于后面沟通 |
| /compact       | 压缩对话，重开对话，但是不希望丢掉之前的记忆                 |
| /cost          | 花费，查看花费消耗                                           |
| /logout /login | 登出登录，常用于切换账号                                     |
| /model         | 切换模型                                                     |



## 二、与IDE无缝衔接

### 1、安装插件

![image-20251218002104896](https://picture-bed.gmbaifa.online/PicGo/image-20251218002104896.png)

### 2、打开对话

![image-20251218002152480](https://picture-bed.gmbaifa.online/PicGo/image-20251218002152480.png)

### 3、使用claude根据代码对话

>框选代码--》右击--》send to Claude Code

![image-20251218002252474](https://picture-bed.gmbaifa.online/PicGo/image-20251218002252474.png)

![image-20251218002417234](https://picture-bed.gmbaifa.online/PicGo/image-20251218002417234.png)

>可以优化代码，他会自己读取文件，所以，我们有关业务的一些保密信息，比如账号密码，ak、sk，key等。一定要注意不要泄露。

![image-20251218002820050](https://picture-bed.gmbaifa.online/PicGo/image-20251218002820050.png)

## 三、模式切换

### 1、自动编辑模式和规划模式

>可以在交互界面直接切换

#### 1.自动编辑模式

>适合无需逐次确认的文件创建、修改场景。按下Shift Tab一次即可开启，此时 Claude 会自动执行编辑操作，无需手动确认。比如提示 “创建一个酷炫的 todolist 应用”，它会直接生成文件并修改，省去反复确认的时间。

![image-20251218003612288](https://picture-bed.gmbaifa.online/PicGo/image-20251218003612288.png)

#### 2.Plan模式

>面对项目搭建或复杂问题时，用Shift Tab两次开启 Plan 模式。它会先梳理方案框架，比如要做 “像素风格的移动端 todolist”，会自动规划技术栈、页面结构、适配方案等，确认后再动手。若不满意可直接说 “重新规划”，直到符合预期。

![image-20251218003711662](https://picture-bed.gmbaifa.online/PicGo/image-20251218003711662.png)

### 2、Yolo模式，授予最高权限放手干

>重构代码、启动新项目或修复复杂 bug 时，用claude --dangerously-skip-permissions进入 Yolo 模式。此时 Claude 拥有更高权限，可直接执行更多操作（需注意安全，建议在沙箱环境使用）。进入后仍能用Shift Tab调整模式，灵活切换权限粒度。

```bash
claude --dangerously-skip-permissions
```

## 四、CLAUDE.md 全局记忆

### 1、背景和介绍

和聊天机器人交流时，我们知道 “系统提示词” 很重要，会持续影响 AI 的行为。那么 CC 中， CLAUDE.md 也是类似的地位。一个典型的工作流是：

>建初始 CLAUDE.md --> 对话直到长度接近溢出，运行 /compact 续命 --> 达到里程碑时要求 CC 根据进度更新 CLAUDE.md --> 循环直到结束。

可以看到 CLAUDE.md 就是一个持续发挥作用的全局变量。而且 CC 往里写入时一般做了充分的缩略，所以可读性很好。

### 2、注意事项

>- 文件不要太长，毕竟 CC 会默认读入这个文件
>- 会话时为了省事，说 claude.md 时 CC 也可以懂
>- 文件里适合放提醒事项，比如 “要求 CC 每次宣布成功时都要带上证据文件链接”，以及 “代理服务端口是 9890” 。然后会话时，可以要求 CC “查询 claude.md 相关部分”。

### 3、使用示范

>创建一个空的项目文件夹，然后和CC对话

```bash
我想用设计一个电商网站，基于go语言开发，使用微服务框架，首先生成项目需求和技术方案放到plan.md里面，然后将go项目代码生成规范系统提示词输出到CLAUDE.md文件中，之后执行plan.md中计划去写代码的时候，需要参考CLAUDE.md文件中提到的规范。
```

![image-20251218005813607](https://picture-bed.gmbaifa.online/PicGo/image-20251218005813607.png)

![image-20251218005842001](https://picture-bed.gmbaifa.online/PicGo/image-20251218005842001.png)

## 五、会话管理

### 1、随时暂停与回滚

>- 工作中按Esc可暂停当前操作，比如发现 Claude 安装依赖超时、思路跑偏时，及时中断能减少无效操作。
>- 按Esc两次可回退到历史对话节点（注意无 redo 功能，回退前确认）。
>- 代码不满意？直接说 “回滚到上次的代码”，Claude 会自动恢复之前版本。

### 2、对应对话溢出

>当会话提示 “Context left until auto-compact: 3%”，说明历史记录快满了。此时会自动触发压缩（约 150 秒），也可手动用/compact命令续命，避免对话中断。

![image-20251219232647906](https://picture-bed.gmbaifa.online/PicGo/image-20251219232647906.png)

### 3、恢复与查看历史

>- 用`claude -c`直接进入上次对话；
>- 用`claude -r`选择历史会话恢复，适合中途退出后继续工作。

## 六、资源监控与批量任务

### 1、实时监控token用量

>想知道每天 / 每小时消耗多少资源？运行`npx ccusage@latest`查看按天用量，或`npx ccusage blocks --live`实时监控消耗速度。若速度过快，可手动处理 git commit 等费 token 的操作，避免超额。

### 2、批量任务

>需要执行几十个重复任务，用脚本式用法

#### 1.TASK.md准备

```md
# Tasks
讲一个笑话
计算1+1=多少
```

#### 2.并发执行

```bash
cat TASK.md | while IFS= read -r line; do echo $line; claude -p "$line" --debug; done
```

![image-20251219234403495](https://picture-bed.gmbaifa.online/PicGo/image-20251219234403495.png)

>可加timeout防止单个任务卡死，同时用--allowedTools "Edit"限制权限，避免意外操作。注意不要并发执行，否则可能触发限额封禁。

## 七、避坑与进阶

### 1、给足自由，也要做好防护

>- 开启auto-accept edits（Shift Tab切换）让 Claude 自动编辑文件，配合 git 版本控制，不怕误操作。
>
>- 执行 Bash 命令时，用 Docker 隔离环境，或用 btrfs 文件系统做快照，既能放开权限，又能快速回滚。
>- 避免在会话目录存放 ssh key、ak sk、api key、中间件账号密码等敏感信息，防止跨机操作风险，数据泄露。业务上最好引用配置中心，或者配置文件移出会话，使用变量导入等方式。

### 2、谨防幻觉造假

>- Claude 偶尔会 “吹牛”，比如未完成测试就宣称成功。可在CLAUDE.md中加入规则：“每次宣称成功必须附证据文件链接”，并定期反问：“真的完成了？有证据吗？” 发现问题及时让其修正。
>- 掌握这些技巧，能让 Claude Code 从 “工具” 变成“高效搭档”，无论是日常编码、项目管理还是批量处理，都能大幅提升效率。记住：核心是按需切换模式、用好全局记忆，再加上适当的监控和防护，就能发挥它的最大价值。

## 八、Claude code使用建议

>我想用设计一个电商网站，基于go语言开发，使用微服务框架，请帮我创建：ARCHITECTURE.md 记录核心设计与数据模型、创建DEVELOPMENT_PLAN.md 记录分阶段任务拆解和任务计划与验收命令、创建PROGRESS.md用于记录进度的唯一真实来源（SSOT）、创建CLAUDE.md用于定义行为规范与禁止事项。CLAUDE.md定义每次必须读取前面三个md文件确认计划和进度来确认现状。CLAUDE.md中加入规则：“每次宣称成功必须附证据文件链接记录到PROGRESS.md中”



