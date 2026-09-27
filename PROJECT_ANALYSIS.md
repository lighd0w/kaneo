# Kaneo 项目分析报告

> 仓库: https://github.com/usekaneo/kaneo · 本地路径: `C:\2code\kaneo`
> 分析基于 main 分支最新代码（v2.28.3，2026-09-26）

---

## 1. 项目概述

**Kaneo** 是一个开源（MIT）、可自托管的项目管理平台，定位类似 Linear / Trello / Planka 的轻量替代品。核心理念是"少即是多"——作者认为大多数项目管理工具的问题在于功能过多而非过少，因此 Kaneo 刻意保持界面简洁、性能优先、自托管友好。

| 项目 | 信息 |
|---|---|
| 许可证 | MIT（© 2024 Andrej Acevski） |
| 当前版本 | v2.28.3 |
| 创建时间 | 2024-12-30 首次提交 |
| 提交总数 | 3,142+ |
| 代码规模 | ~1,637 个 TS/TSX 文件，约 19.2 万行（不含生成文件） |
| 主要贡献者 | Andrej Acevski（核心作者）、Tin、justin、MonsPropre 等 |
| 商业模式 | 开源 + 托管云（cloud.kaneo.app，含 Creem 计费）双轨 |
| 文档 | kaneo.app/docs（Mintlify 站点，随仓库维护） |

**功能范围**：工作区/项目/看板/任务（子任务、关联、标签、优先级、截止日期）、日历与甘特图、Backlog、时间跟踪、评论与活动流、通知（多渠道投递）、全局搜索、团队协作与邀请、细粒度权限角色、集成（GitHub/GitLab/Gitea/Slack/Discord/Mattermost/Telegram/通用 Webhook/ntfy/Gotify）、iCal 日历订阅、MCP 服务器（AI 工具接入）、S3 附件上传、可选中开启的云计费。

---

## 2. 技术栈总览

| 层 | 技术 |
|---|---|
| 后端 | Hono 4 + `@hono/zod-openapi` + `@hono/node-server` + `@hono/node-ws`（Node 24+） |
| 前端 | React 19 + Vite（含 React Compiler）、TanStack Router v1（文件路由）、TanStack Query v5 |
| UI | Tailwind CSS v4、shadcn/ui 风格组件（Base UI + Radix UI）、Lucide 图标、Tiptap v3 富文本、dnd-kit 拖拽、framer-motion |
| 状态 | 服务端状态用 TanStack Query；本地 UI 状态用 Zustand v5；会话用 Better Auth（nanostores） |
| 认证 | Better Auth 1.6（邮箱 OTP/魔法链接/密码、GitHub/Google/Discord OAuth、API Key、设备授权、匿名访客） |
| 数据库 | PostgreSQL 16 + Drizzle ORM（`pg` 驱动，启动时自动跑迁移） |
| 实时 | 自建 WebSocket 中心 + 进程内事件总线；可选 Redis（ioredis，支持单机/哨兵/集群）做跨实例扇出 |
| 存储 | S3 兼容对象存储（预签名 URL 直传，可选） |
| 邮件 | SMTP（`@kaneo/email` 包，模板化） |
| 后台任务 | croner 定时任务 + Postgres `job_lease` 表做 leader 选举 |
| 可观测 | Sentry（Node + React + 性能采样），sentry/ 目录内有告警和仪表盘的配置即代码 |
| 工程 | pnpm 10 workspace + Turborepo、Biome（lint/format）、commitlint、husky |
| 测试 | Vitest（单元 + DB 集成 + 存储集成）、Playwright E2E（含 BrowserStack） |
| 部署 | Docker 单镜像（nginx 反代同端口）、独立 api/web 镜像、Docker Compose、Coolify、Helm Chart |
| AI 工程化 | 内置 MCP HTTP 端点 + npm 包 `@kaneo/mcp`、AGENTS.md 代理操作手册、CodeRabbit 代码评审、AI skills/ 目录 |

---

## 3. Monorepo 结构

pnpm workspace + Turborepo 编排（`turbo build/dev/lint/test/typecheck`）。

```
kaneo/
├── apps/
│   ├── api/        # @kaneo/api — Hono 后端（444 个 TS 文件，核心）
│   ├── web/        # @kaneo/web — React 前端（749 个 TS 文件，最大）
│   ├── site/       # @kaneo/site — Next.js 官网与文档宿主
│   └── docs/       # Mintlify 文档源（111 个 MDX + OpenAPI 规范）
├── packages/
│   ├── libs/               # @kaneo/libs — 共享类型化 Hono RPC 客户端
│   ├── permissions/        # @kaneo/permissions — 权限词汇表与内置角色
│   ├── email/              # @kaneo/email — SMTP 与邮件模板
│   ├── mcp/                # @kaneo/mcp — 发布到 npm 的 stdio MCP 服务器
│   ├── planka-import/      # PLANKA → Kaneo 迁移工具
│   └── typescript-config/  # 共享 tsconfig
├── charts/kaneo/           # Helm Chart
├── i18n/                   # 20 种语言 JSON（en-US 为源，含 zh-CN）
├── tests/
│   ├── api/                # ~100 个 API 单元测试
│   ├── api-integration/    # PostgreSQL 集成测试
│   ├── storage-integration/ # S3 存储集成测试
│   └── e2e/                # Playwright 冒烟测试（独立 workspace）
├── skills/                 # AI 协作技能（动画设计、原型等）
├── plans/                  # 动效改进计划文档
├── sentry/                 # Sentry 告警/仪表盘配置
└── scripts/                # i18n 校验、OpenAPI 校验、发布脚本
```

---

## 4. 后端架构（apps/api）

### 4.1 入口与启动流程

入口在 [apps/api/src/index.ts](apps/api/src/index.ts)，流程：

1. `instrument.ts` 初始化 Sentry（仅在配置了 DSN 时）。
2. `createApp()` 构建根 Hono 应用：客户端 IP 中间件（只信任 `x-kaneo-client-ip`）、全局错误处理（5xx 才报 Sentry）、CORS（凭据模式）、gzip 压缩、`createNodeWebSocket` 注入 WS（payload 上限 64 字节）。
3. 内层 `OpenAPIHono` 承载全部 API，挂载在 `/api` 前缀下。
4. **启动任务序列**：等待数据库就绪 → 手写数据迁移 → **Drizzle `migrate()` 自动跑 `apps/api/drizzle/` 下 55 个 SQL 迁移** → 列迁移补丁 → 种子默认工作区角色 → 初始化集成插件 → 启动定时任务 → 初始化 WebSocket 扇出适配器。
5. `@hono/node-server` 监听 1337 端口，注册 WebSocket，优雅关停（停调度器、停 WS、等邮件队列排空）。

### 4.2 路由组织：OpenAPI 优先

核心模式在 [apps/api/src/openapi.ts](apps/api/src/openapi.ts)（仅 82 行）：

- `apiRouter()` 工厂返回带 `defaultHook` 的 `OpenAPIHono`，把第一个 Zod 校验错误转成 `400 field: message`。
- 每个功能模块一个目录，内部结构统一：`index.ts`（`createRoute({...})` 路由定义）+ `schema.ts`（请求 Zod 模式）+ `response.ts`（响应 Zod 模式，`.openapi("Name")` 使其成为可复用组件）+ `controllers/`（每操作一个文件）。
- 全部路由自动生成 OpenAPI 3.1 文档（`GET /openapi`），并提交到 `apps/docs/openapi.json` 供文档站使用——**CI 会在规范漂移时失败**，保证文档与代码一致。
- 前端类型化客户端直接消费该 Hono 应用的 `AppType`，实现端到端类型安全。

已注册的路由组（[apps/api/src/index.ts](apps/api/src/index.ts)）：`/project`、`/task`、`/column`、`/comment`、`/activity`、`/time-entry`、`/label`、`/search`、`/notification`、`/custom-field`、`/workflow-rule`、`/external-link`、`/task-relation`、`/calendar-feed`、`/invitation`、`/workspace`、`/user`、`/admin`、`/billing`、`/config`、`/instance`，以及 7 个集成路由（github/gitea/gitlab/slack/mattermost/discord/telegram/generic-webhook）和 MCP 路由。

认证顺序设计：公开路由（健康检查、公开看板、Git 提供方 webhook、iCal 订阅）先注册；`/auth/*` 交给 Better Auth 处理（带 API key 桥接）；MCP 在全局认证门之前挂载（自己做 Bearer 校验）；其余全部走全局认证中间件 `authenticateApiRequest`（会话 Cookie / Bearer 会话 / Bearer API Key / `x-api-key`），并通过 `AsyncLocalStorage` 把 `initiatorId` 注入事件上下文，使实时广播能排除发起者自己的客户端。

### 4.3 数据库层

- [apps/api/src/database/schema.ts](apps/api/src/database/schema.ts)（1287 行，约 40 张表）：Better Auth 表（user/session/account/apikey 等）+ 领域表（workspace、project、column、task、task_relation、label、activity、time_entry、integration、notification、custom_field、workflow_rule、calendar_feed、job_lease 等）。
- ID 用 `cuid2`；关系定义在 [relations.ts](apps/api/src/database/relations.ts)（421 行），支持 `db.query.*` 关系查询。
- `db` 默认导出是一个惰性 Proxy，规避 dotenv 初始化顺序问题。
- 迁移工作流：`pnpm --filter @kaneo/api db:generate` 生成 SQL → 人工检查 → 提交；服务器启动时自动应用。另有若干手写幂等迁移（`utils/migrate-*.ts`）处理历史数据修补。
- Better Auth 适配器（[auth-adapter.ts](apps/api/src/database/auth-adapter.ts)）重写 update 逻辑，用 `pg_advisory_xact_lock` 防止把最后一个实例管理员降级。

### 4.4 认证与权限模型

[apps/api/src/auth.ts](apps/api/src/auth.ts) 配置 Better Auth，映射 `organization → workspace`、`member → workspace_user`，启用插件：anonymous（可关的访客访问）、magicLink、emailOTP、organization、genericOAuth（OIDC/PKCE）、bearer、api-key（限速 100/分）、deviceAuthorization（白名单 `kaneo-cli`/`kaneo-mcp`）、admin（实例级管理员）。

**权限模型**（`@kaneo/permissions` 包 + [require-workspace-permission.ts](apps/api/src/utils/require-workspace-permission.ts)）：

- 自定义 AccessControl 语句：`project`、`task`、`label`、`workspace` 等资源维度。
- 内置角色 `viewer / member / admin`（**动态**——作为 `workspace_role` 表的行存储，可在每个工作区自定义编辑）+ 静态 `owner`（不可被编辑掉，保护创建者）。
- 请求时检查顺序：API Key scope → 实例管理员旁路 → 成员角色 → 角色 statements（优先读数据库行，回退编译内置）。
- 工作区隔离由 `workspaceAccess.fromProject/fromTask/fromLabel/...` 中间件族完成：从参数/请求体/数据库反查 workspaceId，再校验访问权，防止跨工作区越权（包括批量操作的交叉校验）。

**注册策略**高度环境可配：`DISABLE_REGISTRATION`、`DISABLE_PASSWORD_REGISTRATION`、`DISABLE_LOGIN_FORM`、`DISABLE_WORKSPACE_CREATION`、一次性邮箱域名拦截、Turnstile 验证码（云模式）、首位非访客用户自动成为实例管理员等。

### 4.5 实时机制：事件总线 → WebSocket → 客户端缓存

三段式设计（[events/index.ts](apps/api/src/events/index.ts)、[ws/index.ts](apps/api/src/ws/index.ts)）：

1. **事件总线**：进程内 `EventEmitter`。所有 mutation 调用 `publishEvent(eventType, data)`，`AsyncLocalStorage` 自动附带 `initiatorId`（`userId:windowId`）。
2. **WS 中心**：两类端点——`GET /api/ws/:projectId`（看板协作）与 `GET /api/ws/user`（个人通知）。订阅约 19 种领域事件并翻译成 `TASK_UPDATED`/`TASK_CREATED`/`TASK_MOVED` 等消息。同项目消息按 100ms 批量合并去重后广播。
3. **扇出适配器**：单实例用内存实现；配置 Redis 后用 `RedisBroadcastAdapter`（发布/订阅模式，valibot 校验消息、带实例 origin 防重复投递），支持多 API 实例水平扩展。

**安全**（[ws/security.ts](apps/api/src/ws/security.ts)）：升级时校验 Origin 白名单（Cookie 无法跨源携带，原生客户端必须带 Bearer/x-api-key）；入站消息只接受 ≤64 字节的 `{"type":"ping"}` 保活，其余一律断开。

前端对应模式（AGENTS.md 强调的约定）：**影响实时状态的 mutation 必须同时考虑事件发布 → WS 投递 → 客户端缓存失效** 三个环节。前端 WS hook 收到消息后调用 `queryClient.invalidateQueries`，不做乐观本地存储。

### 4.6 后台任务与集成

**调度器**（[scheduler/](apps/api/src/scheduler/)）：croner 注册 4 个定时任务（截止提醒、项目 webhook 提醒、席位对账、试用提醒），每个都包 Sentry check-in；多实例协调用 Postgres `job_lease` 表 + 租约 TTL 做 leader 锁——**不引入额外基础设施**。

**集成插件系统**（[plugins/](apps/api/src/plugins/)）：统一 `IntegrationPlugin` 接口 + 注册表，订阅 `task.created`、`task.status_changed`、`comment.created` 等事件并分发到每个启用的集成行。GitHub 插件最完整：GitHub App + octokit，webhook 签名验证，双向同步（issue ↔ task）。Git 提供方的 webhook 接收路由注册在全局认证门之前（各自验签）。

**MCP 服务器**（[mcp/](apps/api/src/mcp/)）：每个实例自带 `POST /api/mcp`（Streamable HTTP 传输），完整实现 OAuth 2.0 动态客户端注册、授权 + 同意页、PKCE token 交换、RFC 8414/9728 元数据端点。这意味着 Claude/Cursor 等 AI 工具可以直接通过 OAuth 连接任意 Kaneo 实例管理任务——这在同领域产品里相当超前。

**其他**：S3 预签名直传（10MB 上限、MIME 白名单、5 分钟 TTL）+ 资产访问鉴权 + 清理任务；计费（Creem）只在云模式启用，webhook 免认证但验签；邮件通过有界内存队列异步投递（SMTP 延迟不阻塞 HTTP 响应，优雅关停时排空）。

---

## 5. 前端架构（apps/web）

### 5.1 框架与构建

- **React 19 + Vite + React Compiler**（babel preset 自动 memo 化，见 [vite.config.ts](apps/web/vite.config.ts)）。
- **TanStack Router v1** 文件式路由，`autoCodeSplitting: true` 按路由自动分包，路由树由 `routeTree.gen.ts` 生成。
- **TanStack Query v5** 管理全部服务端状态：单一 QueryClient（[query-client/index.ts](apps/web/src/query-client/index.ts)），`refetchOnWindowFocus/Mount: false`、有界重试、401 全局处理（自动跳登录）、Sentry 网络错误限流上报。
- **Zustand v5**（persist 中间件）存本地偏好（主题、视图模式、紧凑模式等）。
- 应用根 [main.tsx](apps/web/src/main.tsx)：ErrorBoundary → QueryClientProvider → ThemeProvider → AuthProvider → I18nProvider → KeyboardShortcutsProvider。

### 5.2 类型化 API 客户端（前后端契约的关键）

类型化客户端在共享包 [packages/libs/src/hono.ts](packages/libs/src/hono.ts)：用 Hono 的 `hc<AppType>(apiUrl)` 直接从后端 Hono 应用类型推导出完全类型安全的 RPC 客户端。前端三层组织：

```
fetchers/<domain>/   # 薄封装：调用 client.project[":id"].$get()，检查 response.ok，失败抛 HttpError；
                     # 请求参数类型用 InferRequestType 从路由推导并再导出
hooks/queries/<domain>/use-get-*.ts    # useQuery 封装，queryKey 有统一约定（如 ["projects", workspaceId, id]）
hooks/mutations/<domain>/use-*.ts      # useMutation 封装，onSuccess 批量 invalidate 相关 queryKey
```

AGENTS.md 明确禁止绕过该客户端另建未类型化请求层——这是整个项目前后端契约一体化的基石。看板用有界分页（[load-board-pages.ts](apps/web/src/fetchers/task/load-board-pages.ts)）逐页拉取并按 id 去重合并，保持缓存形状稳定。

### 5.3 路由与页面

```
公开：/ (落地页) · /auth/*（登录/注册/找回/OTP） · /device*（设备授权） · /mcp/authorize ·
      /public-project/$projectId · /invitation/accept/$inviteId
认证后（routes/_layout/_authenticated，beforeLoad 守卫重定向登录）：
  /dashboard · /dashboard/invitations · /dashboard/onboarding
  /dashboard/settings/account|admin|projects/$projectId|workspace
  /dashboard/workspace/$workspaceId（主页/成员/搜索）
  /dashboard/workspace/$workspaceId/project/$projectId/{index,board,backlog,calendar,gantt,task/$taskId}
```

### 5.4 组件与 UI 体系

- **shadcn/ui 风格**（new-york 风格、zinc 基色），63 个 UI 原语在 [components/ui/](apps/web/src/components/ui/)，构建在 **Base UI（@base-ui/react）+ Radix UI** 之上，`cva` + `tailwind-merge` 管理类名。
- **Tailwind CSS v4**（`@tailwindcss/vite` 插件，CSS-first 配置，`@custom-variant dark` 类策略暗色主题）。
- 看板：[components/kanban-board/](apps/web/src/components/kanban-board/)，`@dnd-kit` 拖拽排序；另有列表视图、Backlog 视图、日历、甘特图。
- 富文本：**Tiptap v3**（任务描述、评论），配 markdown 扩展、表格、任务列表、代码高亮（shiki/lowlight），dompurify 消毒。
- 实时：[use-project-websocket.ts](apps/web/src/hooks/use-project-websocket.ts) 与 [use-user-websocket.ts](apps/web/src/hooks/use-user-websocket.ts)，30s ping 保活（应对 Cloudflare 100s 空闲断连）、最多 5 次指数退避重连、disposed 标志防跨项目事件泄漏；收到消息后按领域批量失效 Query 缓存。
- 动效：framer-motion，显式处理 `prefers-reduced-motion`。

### 5.5 i18n

`react-i18next` + 按命名空间组织的 20 种语言 JSON（根 [i18n/](i18n/) 包，`en-US.json` 是源）。惰性按 locale 分块加载；locale 解析顺序：用户偏好 → 浏览器语言 → 默认；带陈旧分块恢复机制。所有用户可见文案必须是静态 i18n key，`scripts/i18n/check.mjs` 在 CI 校验完整性。

---

## 6. 部署形态

| 方式 | 说明 |
|---|---|
| **单容器（推荐）** | [Dockerfile.kaneo](Dockerfile.kaneo)：多阶段构建，最终镜像 = API(Node) + 前端静态文件 + **nginx 同端口反代**（5173 同时服务 UI 和 `/api`），非 root 用户运行，自带健康检查；入口脚本在启动时把 `KANEO_API_URL` 等替换进构建产物。只需外挂 PostgreSQL。 |
| 分离镜像 | `ghcr.io/usekaneo/api`（1337）+ `ghcr.io/usekaneo/web`，适合前后端分开部署 |
| Docker Compose | [compose.yml](compose.yml)：kaneo + postgres:16，含健康检查与依赖顺序；Redis 可选注释块 |
| Coolify | 专用 [compose.coolify.yml](compose.coolify.yml) |
| Kubernetes | [charts/kaneo](charts/kaneo) Helm Chart，CI 校验 |
| drim | 官方 CLI 一键部署（自动 HTTPS、数据库、配置） |
| Redis | 仅在多 API 实例时需要（实时消息跨实例扇出）；单实例零额外依赖 |

环境变量集中在根 [.env](.env.sample)：`DATABASE_URL`、`AUTH_SECRET`、`KANEO_CLIENT_URL`、SMTP、OAuth 凭据等；可选 S3、Redis、Turnstile、Sentry。注意 `engines: node >= 24`（根 package.json）。

---

## 7. 质量保障与工程实践

**测试金字塔**：
- 单元测试：Vitest，API 约 100 个测试文件在 [tests/api/](tests/api/)，前端 104 个测试与组件同目录放置。
- 集成测试：[tests/api-integration/](tests/api-integration/)（真实 PostgreSQL）、[tests/storage-integration/](tests/storage-integration/)（真实 S3）。
- E2E：[tests/e2e/](tests/e2e/) Playwright 冒烟测试（TLS 拓扑 + BrowserStack 云真机）。

**CI/CD**（22 个 GitHub workflow）：
- `ci.yml` 常规检查；`nightly.yml` 夜间扫描；`browser-tests.yml`/`browserstack.yml` 浏览器测试。
- **发布是手动且有节制的**：从 main 手动触发 Release workflow，按 Conventional Commits 解析版本号，先构建三套 GHCR 镜像、校验 Helm chart，全部成功后才打 tag/发 Release/写 CHANGELOG——**绝不会出现 tag 指向未发布镜像**。发布说明脚本把每个 commit 回溯到 PR，按 PR 折叠去重。
- AI 协作深度融入：CodeRabbit 自动评审（`.coderabbit.yaml`）、UI 视觉评审 workflow、`pr-title.yml` 强制 conventional commits、甚至有防"未检查代码的机器人"的 honeypot 机制（AGENTS.md 要求外部 AI 贡献者添加认错文件）。

**代码质量**：Biome 统一 lint/format（2.5.7）；TypeScript 7.0.2（`turbo typecheck` 全仓校验）；commitlint + husky；pnpm overrides 大量固定依赖版本修复已知漏洞；`.trufflehog-exclude` + 密钥扫描。

**可观测性即代码**：[sentry/](sentry/) 内有 `alerts.json`、`dashboards.json`——告警与仪表盘配置随仓库版本化，前后端均有 Sentry 接入（5xx 才上报，带限流）。

---

## 8. AI 工程化亮点（值得借鉴）

这个仓库是我见过的对 AI 协作开发支持最完善的社区项目之一：

1. **AGENTS.md 操作手册**：不是营销文档，而是真正的运营规则——架构边界（"API 是授权的唯一权威，UI 隐藏按钮不等于鉴权"）、变更必须检查的完整表面清单（路由→客户端→hook→缓存失效→事件→WS→权限→迁移→翻译→部署）、验证策略（"用覆盖变更行为的最小证明"）、发布流程、领域术语表。
2. **内置 MCP 服务器**：每个实例自带 MCP 端点 + 完整 OAuth，AI 工具可安全接入任意实例。
3. **skills/ 目录**：动画设计、原型制作等 AI 技能集（配合 `skills-lock.json` 版本锁定）。
4. **plans/ 目录**：动效改进的结构化计划文档，展示"计划先行"的工作流。
5. **自动化防滥用**：honeypot 文件机制识别未实际检查代码的 AI 机器人贡献。

---

## 9. 亮点与观察

### 优点

- **类型安全贯穿全栈**：Zod 定义 → OpenAPI 生成 → Hono AppType → 前端 `hc<AppType>` RPC 客户端 → `InferRequestType/ResponseType`，一条链路无断点，且 CI 强制 OpenAPI 规范不漂移。
- **单实例优先的自托管体验**：不强制 Redis/S3/SMTP，单容器单数据库即可跑全功能；Redis 只为水平扩展预留。
- **实时架构干净**：事件总线与 WS 分层，批量合并广播，发起者回声抑制，内存/Redis 双适配器。
- **权限模型精细且务实**：动态角色（数据库行）+ 静态 owner 保护 + API Key scope + 实例管理员旁路的清晰检查顺序。
- **迁移纪律好**：Drizzle 迁移 + 启动自动应用 + 手写幂等补丁，明确要求"数据库变更必须对已有安装生效"。
- **测试覆盖全面**：单元/集成/存储/E2E 四层，测试位置规划合理（API 测试集中在根 tests/，前端同目录）。

### 需要注意的点

- **规模已不小**：约 19 万行 TS，前端 749 文件 + 后端 444 文件，二次开发需先理解其 fetcher→hook→invalidation 约定与事件面检查清单。
- **React Compiler + React 19** 较新，部分第三方库兼容性需留意。
- **云/自托管双轨代码**（billing、Turnstile、cloud 判断）散布在关键路径，自托管部署通常不触发但阅读代码时需分辨。
- **Node >= 24 / pnpm 10 / TS 7** 工具链很激进，本地环境需匹配。
- **AGENTS.md 中的贡献者规则**（如 honeypot 文件）是给外部 AI 贡献者的，理解其意图即可，不必在本仓库套用。
- Windows 本地开发未在文档中重点说明（官方推荐 drim/Docker），建议用 WSL2 或 Docker Compose 跑开发环境。

---

## 10. 关键文件索引

| 关注点 | 文件 |
|---|---|
| 后端入口/路由注册 | [apps/api/src/index.ts](apps/api/src/index.ts) |
| 路由工厂/OpenAPI | [apps/api/src/openapi.ts](apps/api/src/openapi.ts) |
| Better Auth 配置 | [apps/api/src/auth.ts](apps/api/src/auth.ts) |
| 数据库 schema / 关系 | [apps/api/src/database/schema.ts](apps/api/src/database/schema.ts) · [relations.ts](apps/api/src/database/relations.ts) |
| 权限词汇表 | [packages/permissions/src/index.ts](packages/permissions/src/index.ts) |
| 权限校验中间件 | [apps/api/src/utils/require-workspace-permission.ts](apps/api/src/utils/require-workspace-permission.ts) |
| 事件总线 / WS 中心 | [apps/api/src/events/index.ts](apps/api/src/events/index.ts) · [apps/api/src/ws/index.ts](apps/api/src/ws/index.ts) |
| 调度器 / leader 锁 | [apps/api/src/scheduler/index.ts](apps/api/src/scheduler/index.ts) |
| 集成插件注册 | [apps/api/src/plugins/index.ts](apps/api/src/plugins/index.ts) |
| MCP 服务器 | [apps/api/src/mcp/index.ts](apps/api/src/mcp/index.ts) |
| 类型化客户端 | [packages/libs/src/hono.ts](packages/libs/src/hono.ts) |
| 前端入口 | [apps/web/src/main.tsx](apps/web/src/main.tsx) |
| QueryClient | [apps/web/src/query-client/index.ts](apps/web/src/query-client/index.ts) |
| 前端 WS hooks | [apps/web/src/hooks/use-project-websocket.ts](apps/web/src/hooks/use-project-websocket.ts) |
| 看板组件 | [apps/web/src/components/kanban-board/index.tsx](apps/web/src/components/kanban-board/index.tsx) |
| 单镜像 Dockerfile | [Dockerfile.kaneo](Dockerfile.kaneo) |
| AI 操作手册 | [AGENTS.md](AGENTS.md) |
| 环境搭建指南 | [ENVIRONMENT_SETUP.md](ENVIRONMENT_SETUP.md) |
