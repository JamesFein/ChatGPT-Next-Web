# CLAUDE.md

本文件为Claude Code（claude.ai/code）在此仓库工作时提供指导。

## 项目概览

**NextChat** 是一款轻量级高效的AI助手网页应用，支持多个LLM提供商（OpenAI、Claude、Google Gemini、DeepSeek等15+个）。具备现代化聊天UI、提示词模板、制品、插件、实时聊天和MCP（Model Context Protocol）支持等功能。

技术栈：Next.js 14 + React 18 + TypeScript + Zustand + Tauri（桌面应用）

## 开发命令

### 基础命令
- **`yarn install`** - 安装依赖
- **`yarn dev`** - 启动开发服务器（同时监听mask变动）。访问 http://localhost:3000
- **`yarn build`** - 生产构建（先构建mask，再构建Next.js）
- **`yarn start`** - 运行生产服务器（需要先执行build）
- **`yarn lint`** - 运行ESLint
- **`yarn test`** - 以监听模式运行Jest测试
- **`yarn test:ci`** - 运行一次测试（用于CI/CD）
- **`yarn mask`** - 构建mask配置文件（如果mask有变动，在dev/build前必须运行）
- **`yarn mask:watch`** - 监听mask文件并自动重新构建

### 桌面应用命令
- **`yarn app:dev`** - 以开发模式启动Tauri桌面应用
- **`yarn app:build`** - 构建桌面应用发行版

### 特殊构建模式
- **`yarn export`** - 导出为静态站点（设置 BUILD_MODE=export）
- **`yarn export:dev`** - 静态导出模式下的开发

### 环境配置
在项目根目录创建 `.env.local`：
```
OPENAI_API_KEY=<你的密钥>
# 可选：代理的BASE_URL
BASE_URL=https://your-proxy/api
```

## 架构设计

### 目录结构概览

```
app/
├── components/          # React UI组件（聊天、设置、侧边栏等）
├── client/              # LLM提供商抽象层
│   ├── api.ts          # 主API接口和平台实现路由
│   ├── controller.ts   # 请求/响应生命周期管理
│   └── platforms/      # 提供商实现（openai.ts、anthropic.ts、google.ts等）
├── api/                # Next.js API路由（后端）
│   ├── [provider]/[...path]/route.ts  # 动态路由分发器
│   ├── openai.ts       # OpenAI特定服务器逻辑
│   ├── anthropic.ts    # Claude API逻辑
│   ├── common.ts       # 共享请求处理
│   ├── auth.ts         # 认证/API密钥验证
│   ├── config/         # 服务器配置端点
│   └── webdav/         # WebDAV同步端点
├── store/              # Zustand状态管理
│   ├── chat.ts         # 聊天消息、对话、历史记录
│   ├── config.ts       # 用户设置、模型偏好
│   ├── access.ts       # API密钥、认证、访问控制
│   ├── mask.ts         # 提示词模板
│   ├── plugin.ts       # 插件系统
│   ├── prompt.ts       # 自定义提示词
│   ├── sd.ts           # Stable Diffusion配置
│   ├── sync.ts         # WebDAV同步状态
│   └── update.ts       # 应用更新检查
├── utils/              # 工具函数
│   ├── format.ts       # 文本格式化、JSON显示
│   ├── token.ts        # Token计数
│   ├── store.ts        # 存储持久化帮助函数
│   └── model.ts        # 模型过滤和收集
├── config/             # 配置
│   ├── client.ts       # 客户端配置加载
│   └── server.ts       # 服务器配置（环境变量）
├── locales/            # i18n多语言翻译
├── masks/              # Mask（提示词模板）定义
└── mcp/                # Model Context Protocol支持
    ├── actions.ts      # MCP工具执行
    └── utils.ts        # MCP工具函数
```

### 数据流架构

**聊天请求流程**：
1. 用户在UI发送消息 → 存储到chat store
2. `getClientApi()` 根据模型配置确定使用哪个提供商平台
3. 聊天请求发送到 `/api/[provider]/[...path]` → Next.js路由
4. 路由认证请求（API密钥验证）
5. 路由调用相应平台API（`openai.ts`、`anthropic.ts`等）
6. 平台处理器转换请求并转发给外部LLM API
7. 响应流回客户端，UI通过 `onUpdate` 回调更新

**状态管理** (Zustand)：
- `useChatStore` - 活跃对话、消息、主题
- `useAccessStore` - API密钥、提供商选择、成本追踪
- `useAppConfig` - 主题、语言、温度、token限制等
- `useMaskStore` - 提示词模板
- `usePluginStore` - 插件注册表和状态

### 核心概念

**提供商（Providers）**: LLM服务（OpenAI、Claude、Google等）- 定义在 `/app/client/platforms/`

**模型（Models）**: 单个模型名称（gpt-4、claude-3-opus等）- 模型列表生成处理基于配置的模型包含/排除

**Mask（面具）**: 可重用的提示词模板，包含系统提示和参数。由mask构建脚本从 `/app/masks/` 定义构建而成。

**插件（Plugins）**: 外部API集成（搜索、计算器等）- 从 `/app/store/plugin.ts` 加载

**制品（Artifacts）**: 在单独窗口中渲染的HTML/代码内容 - 由制品显示组件管理

**MCP** (Model Context Protocol): 工具和资源系统 - 通过 `/app/mcp/actions.ts` 执行

## 重要实现细节

### 请求/响应模式
- 使用 `ChatOptions` 接口配合回调：`onUpdate`（流式）、`onFinish`（完成）、`onError`、`onController`（中止）
- Controllers实现 `ChatControllerPool` 以管理多个并发请求
- 平台实现解析不同的响应格式（OpenAI JSON、流式SSE等）

### Token管理
- 客户端在发送前使用 `estimateTokenLength()` 估算token
- 服务器端计算响应的token数量
- 接近限制时自动压缩聊天历史
- 不同模型采用不同的token计数方式

### 多提供商请求
- 每个提供商有自己的API端点、认证和请求格式
- 公共请求处理在 `/app/api/common.ts`
- 认证基于环境变量按提供商进行
- 支持通过 `BASE_URL` 环境变量或Azure特定配置覆盖BaseURL

### Mask构建系统
Mask构建系统编译提示词模板：
```bash
yarn mask          # 构建一次
yarn mask:watch    # 监听并在变动时重新构建
```
Mask是导出模板定义的TypeScript文件，构建为运行时使用的JSON。

### 测试
- Jest配置用于单元/组件测试
- 测试文件在 `/test` 目录
- 运行单个测试：`yarn test -- --testNamePattern="specific test name"`
- 无内置E2E测试（在dev服务器上手动测试）

## 配置

### 环境变量（服务器端）
影响行为的关键变量：
- `OPENAI_API_KEY` - OpenAI认证
- `ANTHROPIC_API_KEY` - Claude认证
- `CUSTOM_MODELS` - 添加/移除/重命名模型（格式：`+modelname,-modelname,name=displayName`）
- `ENABLE_MCP` - 启用MCP功能（设置为 `true`）
- `HIDE_USER_API_KEY` - 从用户隐藏API密钥输入
- `DISABLE_GPT4` - 隐藏GPT-4模型
- 完整列表见README.md（~30+个提供商变量）

### 客户端配置
从 `/app/config/client.ts` 加载：
- 默认模型选择
- 温度和采样参数
- 最大token限制
- 功能开关

## 常见开发任务

### 添加新的聊天提供商
1. 创建 `/app/client/platforms/newprovider.ts` 实现平台API类
2. 在 `/app/client/api.ts` 中添加提供商case来路由请求
3. 创建 `/app/api/newprovider.ts` 实现服务器端认证和代理逻辑
4. 在 `/app/constant.ts` 的 `ModelProvider` 枚举中添加提供商
5. 在README和 `/app/config/server.ts` 中添加API密钥环境变量

### 添加新模型
1. 提供商的模型列表通过API动态获取或在平台文件中硬编码
2. 使用 `CUSTOM_MODELS` 环境变量在不修改代码的情况下添加模型
3. 格式：`+modelname` 添加，`-modelname` 隐藏

### 修改聊天行为
- 消息处理逻辑在 `/app/store/chat.ts`
- UI在 `/app/components/chat.tsx` 和 `input.tsx`（如果存在）
- 系统提示和参数在 `ModelConfig` store中

### 调试请求
- 检查 `next dev` 控制台的路由日志（`[OpenAI Route]`等）
- 浏览器DevTools网络选项卡显示向 `/api/[provider]/v1/...` 的请求
- 在 `/app/api/auth.ts` 检查API密钥验证

## 构建模式

- **`standalone`**（默认）- Next.js独立构建，可用 `yarn start` 运行
- **`export`** - 静态HTML导出用于静态托管（Vercel等）
- **应用导出** - 使用 `BUILD_APP=1` 标志的桌面Tauri应用构建

## 已知模式

- **流式处理**：平台实现处理SSE/流式格式转换
- **Controllers**：每个请求都获得AbortController用于取消
- **回调**：通过流式回调更新UI，通过onFinish发送最终消息
- **持久化**：Zustand store通过 `createPersistStore` 自动持久化到localStorage/IndexedDB
- **图标**：使用next.config.mjs中配置的 `@svgr/webpack` 进行SVG导入
- **i18n**：根据浏览器语言加载 `/app/locales/` 中的翻译
