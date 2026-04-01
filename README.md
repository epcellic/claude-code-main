# Claude Code — 泄露源代码 (2026-03-31)

> **2026年3月31日，Anthropic 的 Claude Code CLI 的完整源代码通过其 npm 注册表中的 `.map` 文件泄露。**

---

## 泄露方式

[Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) 发现了泄露并公开发布：

> **"Claude code 源代码通过其 npm 注册表中的 map 文件泄露了！"**
>
> — [@Fried_rice, 2026年3月31日](https://x.com/Fried_rice/status/2038894956459290963)

发布在 npm 包中的源映射文件包含了对完整、未混淆的 TypeScript 源代码的引用，该代码可以从 Anthropic 的 R2 存储桶下载为 zip 存档。

---

## 概述

Claude Code 是 Anthropic 的官方 CLI 工具（CLI 是 Command Line Interface 的缩写，即命令行界面。在这里指 Claude Code 的命令行工具），让您直接从终端与 Claude 交互以执行软件工程任务 — 编辑文件、运行命令、搜索代码库、管理 git 工作流等。

此仓库包含泄露的 `src/` 目录。

- **泄露日期**：2026-03-31
- **语言**：TypeScript
- **运行时**：Bun
- **终端 UI**：React + [Ink](https://github.com/vadimdemedes/ink) (用于 CLI 的 React)
- **规模**：~1,900 个文件，512,000+ 行代码

---

## 目录结构

```
src/
单独文件
├── commands.ts              # 命令注册
├── context.ts               # 系统/用户上下文收集
├── cost-tracker.ts          # 令牌成本跟踪
├── costHook.ts              # 成本钩子
├── dialogLaunchers.tsx      # 对话启动器
├── history.ts               # 历史记录
├── ink.ts                   # Ink 相关
├── interactiveHelpers.tsx  # 交互助手
├── main.tsx                 # 入口点 (基于 Commander.js 的 CLI 解析器)
├── projectOnboardingState.ts # 项目引导状态
├── query.ts                 # 查询相关
├── QueryEngine.ts           # LLM 查询引擎 (核心 Anthropic API 调用器)
├── replLauncher.tsx         # REPL 启动器
├── setup.ts                 # 设置
├── Task.ts                  # 任务类型
├── tasks.ts                 # 任务相关
├── Tool.ts                  # 工具类型定义
├── tools.ts                 # 工具注册
文件夹
├── assistant/               # 助手会话和历史
├── bootstrap/               # 引导状态管理
├── bridge/                  # IDE 集成桥 (VS Code, JetBrains)
├── buddy/                   # 伴侣精灵 (复活节彩蛋)
├── cli/                     # CLI 工具和 IO
├── commands/                # 斜杠命令实现 (~50)
├── components/              # Ink UI 组件 (~140)
├── constants/               # 常量定义
├── context/                 # 上下文管理
├── coordinator/             # 多代理协调器
├── entrypoints/             # 初始化逻辑
├── hooks/                   # React 钩子
├── ink/                     # Ink 渲染器包装器
├── keybindings/             # 键绑定配置
├── memdir/                  # 内存目录 (持久内存)
├── migrations/              # 配置迁移
├── moreright/               # 更多权限或功能模块
├── native-ts/               # 原生 TypeScript 工具
├── outputStyles/            # 输出样式
├── plugins/                 # 插件系统
├── query/                   # 查询管道
├── remote/                  # 远程会话
├── schemas/                 # 配置模式 (Zod)
├── screens/                 # 全屏 UI (Doctor, REPL, Resume)
├── server/                  # 服务器模式
├── services/                # 外部服务集成
├── skills/                  # 技能系统
├── state/                   # 状态管理
├── tasks/                   # 任务管理
├── tools/                   # 代理工具实现 (~40)
├── types/                   # TypeScript 类型定义
├── upstreamproxy/           # 代理配置
├── utils/                   # 实用函数
├── vim/                     # Vim 模式
└── voice/                   # 语音输入

---

## 核心架构

### 1. 工具系统 (`src/tools/`)

Claude Code 可以调用的每个工具都实现为自包含模块。每个工具定义其输入模式、权限模型和执行逻辑。

| 工具 | 描述 |
|---|---|
| `BashTool` | Shell 命令执行 |
| `FileReadTool` | 文件读取 (图像、PDF、笔记本) |
| `FileWriteTool` | 文件创建 / 覆盖 |
| `FileEditTool` | 部分文件修改 (字符串替换) |
| `GlobTool` | 文件模式匹配搜索 |
| `GrepTool` | 基于 ripgrep 的内容搜索 |
| `WebFetchTool` | 获取 URL 内容 |
| `WebSearchTool` | 网络搜索 |
| `AgentTool` | 子代理生成 |
| `SkillTool` | 技能执行 |
| `MCPTool` | MCP 服务器工具调用 |
| `LSPTool` | 语言服务器协议集成 |
| `NotebookEditTool` | Jupyter 笔记本编辑 |
| `TaskCreateTool` / `TaskUpdateTool` | 任务创建和管理 |
| `SendMessageTool` | 代理间消息传递 |
| `TeamCreateTool` / `TeamDeleteTool` | 团队代理管理 |
| `EnterPlanModeTool` / `ExitPlanModeTool` | 计划模式切换 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git 工作树隔离 |
| `ToolSearchTool` | 延迟工具发现 |
| `CronCreateTool` | 定时触发器创建 |
| `RemoteTriggerTool` | 远程触发器 |
| `SleepTool` | 主动模式等待 |
| `SyntheticOutputTool` | 结构化输出生成 |

### 2. 命令系统 (`src/commands/`)

用户面向的斜杠命令，以 `/` 前缀调用。

| 命令 | 描述 |
|---|---|
| `/commit` | 创建 git 提交 |
| `/review` | 代码审查 |
| `/compact` | 上下文压缩 |
| `/mcp` | MCP 服务器管理 |
| `/config` | 设置管理 |
| `/doctor` | 环境诊断 |
| `/login` / `/logout` | 认证 |
| `/memory` | 持久内存管理 |
| `/skills` | 技能管理 |
| `/tasks` | 任务管理 |
| `/vim` | Vim 模式切换 |
| `/diff` | 查看更改 |
| `/cost` | 检查使用成本 |
| `/theme` | 更改主题 |
| `/context` | 上下文可视化 |
| `/pr_comments` | 查看 PR 评论 |
| `/resume` | 恢复上一个会话 |
| `/share` | 分享会话 |
| `/desktop` | 桌面应用交接 |
| `/mobile` | 移动应用交接 |

### 3. 服务层 (`src/services/`)

| 服务 | 描述 |
|---|---|
| `api/` | Anthropic API 客户端、文件 API、引导 |
| `mcp/` | 模型上下文协议服务器连接和管理 |
| `oauth/` | OAuth 2.0 认证流程 |
| `lsp/` | 语言服务器协议管理器 |
| `analytics/` | 基于 GrowthBook 的功能标志和分析 |
| `plugins/` | 插件加载器 |
| `compact/` | 对话上下文压缩 |
| `policyLimits/` | 组织政策限制 |
| `remoteManagedSettings/` | 远程管理设置 |
| `extractMemories/` | 自动内存提取 |
| `tokenEstimation.ts` | 令牌计数估算 |
| `teamMemorySync/` | 团队内存同步 |

### 4. 桥系统 (`src/bridge/`)

连接 IDE 扩展 (VS Code, JetBrains) 与 Claude Code CLI 的双向通信层。

- `bridgeMain.ts` — 桥主循环
- `bridgeMessaging.ts` — 消息协议
- `bridgePermissionCallbacks.ts` — 权限回调
- `replBridge.ts` — REPL 会话桥
- `jwtUtils.ts` — 基于 JWT 的认证
- `sessionRunner.ts` — 会话执行管理

### 5. 权限系统 (`src/hooks/toolPermission/`)

在每次工具调用时检查权限。要么提示用户批准/拒绝，要么根据配置的权限模式自动解决 (`default`, `plan`, `bypassPermissions`, `auto` 等)。

### 6. 功能标志

通过 Bun 的 `bun:bundle` 功能标志进行死代码消除：

```typescript
import { feature } from 'bun:bundle'

// 非活跃代码在构建时完全剥离
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

值得注意的标志：`PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`

---

## 关键文件详情

### `QueryEngine.ts` (~46K 行)

LLM API 调用的核心引擎。处理流式响应、工具调用循环、思考模式、重试逻辑和令牌计数。

### `Tool.ts` (~29K 行)

定义所有工具的基础类型和接口 — 输入模式、权限模型和进度状态类型。

### `commands.ts` (~25K 行)

管理所有斜杠命令的注册和执行。使用条件导入为每个环境加载不同的命令集。

### `main.tsx`

基于 Commander.js 的 CLI 解析器 + React/Ink 渲染器初始化。在启动时，并行化 MDM 设置、钥匙串预取和 GrowthBook 初始化以加快启动速度。

---

## 技术栈

| 类别 | 技术 |
|---|---|
| 运行时 | [Bun](https://bun.sh) |
| 语言 | TypeScript (严格) |
| 终端 UI | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| CLI 解析 | [Commander.js](https://github.com/tj/commander.js) (额外类型) |
| 模式验证 | [Zod v4](https://zod.dev) |
| 代码搜索 | [ripgrep](https://github.com/BurntSushi/ripgrep) (通过 GrepTool) |
| 协议 | [MCP SDK](https://modelcontextprotocol.io), LSP |
| API | [Anthropic SDK](https://docs.anthropic.com) |
| 遥测 | OpenTelemetry + gRPC |
| 功能标志 | GrowthBook |
| 认证 | OAuth 2.0, JWT, macOS Keychain |

---

## 值得注意的设计模式

### 并行预取

通过在开始重模块评估之前并行预取 MDM 设置、钥匙串读取和 API 预连接来优化启动时间。

```typescript
// main.tsx — 在其他导入之前作为副作用触发
startMdmRawRead()
startKeychainPrefetch()
```

### 延迟加载

重模块 (OpenTelemetry ~400KB, gRPC ~700KB) 通过动态 `import()` 延迟到实际需要时。

### 代理群

通过 `AgentTool` 生成子代理，`coordinator/` 处理多代理编排。`TeamCreateTool` 启用团队级并行工作。

### 技能系统

在 `skills/` 中定义的可重用工作流，并通过 `SkillTool` 执行。用户可以添加自定义技能。

### 插件架构

内置和第三方插件通过 `plugins/` 子系统加载。

---

## 免责声明

此仓库存档了 **2026-03-31** 从 Anthropic 的 npm 注册表泄露的源代码。所有原始源代码归 [Anthropic](https://www.anthropic.com) 所有。
