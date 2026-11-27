# Summer Checkin 项目整体复盘

> 复盘日期：2026-11-21（Asia/Shanghai）  
> 复盘范围：2026-07-07 初始提交至当前 HEAD `9ff7c81`；覆盖代码、配置、测试、文档、Prisma/SQL、Docker 配置和 Git 历史。  
> 证据标记：**已证实** = 可由当前代码、测试、配置或 Git 记录直接支持；**合理推断** = 多个信号推导但没有直接验收证据；**资料缺失** = 当前资料无法确认。
> 重点增补：2026-11-27 追加“数据库从 0 到 1”及 3 个核心技术专项，并重新执行本地检查；原始复盘时间范围与 HEAD 保持不变。

## 一、项目概览

| 项目名称 | 复盘日期 | 项目周期 | 当前状态 |
| --- | --- | --- | --- |
| Summer Checkin | 2026-11-21 | 2026-07-07 — 2026-11-21 | 功能持续迭代；本地开发链路已跑通，生产验收资料不完整 |

### 核心结论

- **解决了什么问题（已证实）**：把“学习计划、任务、打卡、专注计时、统计、AI 辅助、知识库和多人聊天室”集中到一个学习成长平台中。依据：`README.md` 的功能总览、`src/app/` 路由和 `src/components/` 模块。
- **最重要的成果（已证实）**：形成了 AI 原生的完整链路：模型池降级、工具调用、RAG 检索、长期记忆 v2、会话短期记忆、token 记账和 WebSocket AI 聊天。依据：`src/lib/model-pool.ts`、`src/lib/rag/`、`src/lib/memory/`、`src/lib/context/`、`server/ai.ts`、`src/lib/usage.ts`。
- **最大价值（合理推断）**：项目不只是把 AI 接到页面上，而是把 AI 行为接入了可追踪的数据模型、限额、失败兜底和用户上下文，具备继续演化为个人学习操作系统的基础。
- **最大风险或教训（已证实）**：功能迭代速度超过了数据库迁移、依赖锁定、线上验收和文档同步速度；最近的 pgvector 类型缺失和 Better Auth schema mismatch 都需要人工排查后才能恢复。

### 整体评价

| 维度 | 评价 | 关键依据 |
|---|---|---|
| 目标达成 | 部分达成 | 功能面覆盖完整；但生产环境运行、备份、监控和真实用户验收证据缺失 |
| 交付质量 | 中 | 380 个测试断言通过、lint 通过，关键安全/幂等/隔离逻辑有测试；完整测试仍有 2 个 suite 因依赖缺失失败 |
| 执行效率 | 中 | Git 在约 4 个月内完成 86 个提交和多次垂直功能迭代；同时产生了结构漂移和重复人工迁移成本 |
| 长期价值 | 高（合理推断） | RAG、记忆、上下文、模型降级、评测脚本和部署文档都已沉淀为可复用资产 |

## 二、项目文档总索引

| 优先级 | 模块 | 位置 | 状态 | 摘要 |
|:---:|---|---|---|---|
| P0 | 项目入口 | `README.md`、`start.md` | 已有但有漂移 | 有功能、安装、启动和部署入口；README 的 Prisma 徽章与实际版本不一致 |
| P0 | 项目使命 | `README.md`、`000面试整理项目留下来的资产.md` | 部分已有 | 能说明产品能力和可沉淀资产；缺少正式目标、用户和成功指标 |
| P0 | 需求与验收 | `context.md`、`docs/superpowers/specs/` | 部分已有 | AI 上下文和向量检索有设计规格；缺少全项目验收清单 |
| P0 | 架构设计 | `README.md`、`docker-compose.yml`、`Dockerfile`、`nginx/nginx.conf` | 已有但分散 | HTTP、WebSocket、PostgreSQL、pgvector、OSS 的关系可读；缺少一份单独架构图/运行时契约 |
| P0 | 决策记录 | `docs/superpowers/specs/2026-11-12-vector-search-switch-design.md`、`context.md` | 部分已有 | 关键 AI/RAG 决策有文档；数据库和认证升级决策主要散落在复盘文档 |
| P0 | 测试说明 | `README.md`、`tests/`、`vitest.integration.config.mts` | 部分已有 | 有单测和集成测试目录；缺少统一的测试矩阵与真实环境验收说明 |
| P0 | 复盘记录 | `1112复盘文档和部署文档.md`、本文件 | 已有 | 已记录数据库、Docker、Better Auth 和部署问题；仍需持续维护执行台账 |
| P1 | 领域模型 | `prisma/schema.prisma`、`src/lib/memory/types.ts`、`src/lib/context/` | 部分已有 | 领域结构主要体现于代码和注释；缺少术语表和跨模块不变量清单 |
| P1 | 接口契约 | `src/app/api/**/route.ts`、`server/protocol.ts` | 部分已有 | 路由和 WebSocket 协议在代码中；缺少面向调用者的接口文档和错误码表 |
| P1 | 运维与安全 | `docker-compose.yml`、`scripts/deploy.sh`、`nginx/nginx.conf` | 部分已有 | 有部署脚本、Origin 校验、限流和 Nginx 配置；监控、备份、HTTPS、回滚未形成闭环 |
| P1 | 变更历史 | Git log、`prisma/*.sql` | 已有但不可审计 | Git 提交较丰富；数据库脚本没有统一执行台账，且部分脚本是一次性/不可重复执行 |

## 三、重点模块导航

### 1. 项目目标与成果（P0）

- **原始目标（已证实）**：从初始暑假打卡辅助工具，逐步扩展为包含计划、专注、AI、知识库和聊天室的全栈学习成长平台。依据：最早提交信息、`README.md`、Git 历史中的功能提交。
- **当前成果（已证实）**：
  - 24 个 Next.js API route 文件；
  - 28 个 Prisma model（包含 Better Auth 的 `Verification`）；
  - 独立 WebSocket sidecar，承载聊天室、AI 流式回复、握手鉴权、心跳和按用户限流；
  - PostgreSQL 16 + pgvector，向量列约束为 `vector(1024)`；
  - AI 模型池支持 HIGH/LOW 降级链，RAG 支持向量召回和 cross-encoder 重排；
  - 长期记忆 v2 与会话级短期记忆已经落地代码、数据表和验证脚本。
- **成功标准（资料缺失）**：没有发现统一的用户留存、学习时长、回答质量、检索 Recall@K、P95 延迟、错误率或线上可用性目标，因此无法判断产品目标是否达到，只能判断工程能力已覆盖。

### 2. 需求与约束（P0）

#### 已证实的核心需求

1. 用户注册/登录/会话管理和新用户引导模板克隆。
2. 计划 → 任务 → 打卡的学习闭环，另有待办和番茄钟。
3. AI 计划生成、文档拆任务、工具调用、周报和通知。
4. 用户隔离的知识库上传、分块、embedding、向量检索和重排。
5. 长期记忆与会话短期记忆，用于上下文增强。
6. 单房间 WebSocket 聊天、引用回复、消息幂等、按用户限流和 `@AI`。

#### 关键约束

- Next.js 16 使用项目规则中强调的非传统约定；新增代码前必须阅读 `node_modules/next/dist/docs/`，依据：`AGENTS.md`。
- Node/Docker/CI/部署均按 npm 走；仓库当前同时出现 `pnpm-lock.yaml` 和 `pnpm-workspace.yaml`，形成包管理约束冲突。
- 向量维度必须是 1024；`text-embedding-v4`/`v3` 是当前模型池中符合该约束的主要模型，依据：`src/lib/model-pool.ts` 与 pgvector 设计文档。
- 数据库结构目前采用 `prisma db push` + `prisma/*.sql`，没有 Prisma migrations，依据：`prisma.config.ts`、`Dockerfile`、`start.md`。

#### 未满足或未证实项

- 生产数据库是否已经执行全部必要 SQL：**资料缺失**。
- 线上真实数据下的 RAG 质量、AI 降级质量和 WebSocket 稳定性：**资料缺失**。
- 认证、密钥、备份、监控和回滚的生产演练：**资料缺失**。

### 3. 架构与关键实现（P0）

#### 运行时拓扑

```text
浏览器
  ├─ HTTP/HTTPS → Nginx → Next.js app:3000
  │                         └─ Prisma → PostgreSQL + pgvector
  ├─ /ws       → Nginx → WebSocket sidecar:3001
  │                         ├─ Better Auth Cookie 鉴权
  │                         ├─ Origin 校验 / 心跳 / 用户级限流
  │                         └─ 聊天消息 + AI 流式回复
  └─ 头像直传 → 阿里云 OSS（应用只签发预签名 URL）
```

#### 关键模块

| 模块 | 主要位置 | 作用 |
|---|---|---|
| Web UI / API | `src/app/`、`src/components/` | 页面、Server Components、API Routes、学习功能和管理界面 |
| 认证 | `src/lib/auth.ts`、`src/lib/auth-client.ts`、`src/app/api/auth/[...all]/route.ts` | Better Auth + Prisma adapter + Cookie session |
| AI 模型池 | `src/lib/model-pool.ts` | provider、HIGH/LOW 分档、网络错误/额度错误分类、降级和用量记录 |
| Agent | `src/lib/agent/`、`src/lib/tools/` | 计划、任务、打卡、记忆和知识库工具调用工作流 |
| RAG | `src/lib/rag/`、`src/app/api/knowledge/` | 上传、分块、embedding、pgvector 召回、重排和结果格式化 |
| Memory v2 | `src/lib/memory/`、`prisma/add-memory-v2.sql` | 抽取、验证、选择、生命周期、事件日志和上下文注入 |
| Context | `src/lib/context/` | AgentState、滚动摘要、历史回溯、场景数据和状态栏 |
| WebSocket | `server/` | 握手、房间、协议、消息幂等、限流和 AI 回复 |
| 部署 | `Dockerfile`、`docker-compose.yml`、`docker-entrypoint.sh`、`nginx/` | 构建 standalone 镜像，启动 app + sidecar，并由 Nginx 反代 |

#### 最大架构风险

数据库结构变更没有单一、版本化、可回滚的事实来源。`schema.prisma`、18 个手工 SQL、部署手册和实际环境可能不同步；本次 `vector` 类型缺失和 Better Auth 字段缺失就是该风险的直接表现。

### 4. 决策与权衡（P0）

| 决策主题 | 最终方案 | 未采用方案 | 重新评估条件 | 证据 |
|---|---|---|---|---|
| 向量检索 | PostgreSQL pgvector + HNSW，应用通过 SQL 召回 | 应用层全量拉取后用 JS 计算余弦 | 数据规模或托管数据库限制使 pgvector 不可用 | `prisma/migrate-to-pgvector.sql`、RAG 代码、向量设计文档 |
| AI 供应商容错 | 模型池按档位降级，网络/额度错误继续尝试 | 单一模型失败即报错 | 供应商能力、价格或质量发生变化 | `src/lib/model-pool.ts`、2026-10/11 Git 提交 |
| WebSocket 结构 | 独立 sidecar，与 Next 主应用分进程 | 将 WS 与页面进程混在同一入口 | 需要水平扩展、共享房间状态或统一观测时 | `server/index.ts`、`docker-entrypoint.sh` |
| 聊天消息窗口 | 游标分页 + 最多约 100 条 DOM 窗口 + 锚点补偿 | 直接虚拟化不定高聊天行 | 真实性能指标或 UI 结构改变 | `README.md`、`chat-room.tsx` 注释、`scripts/bench-render.tsx` |
| 数据库迁移 | 本地 `db push`，部署库执行手工 SQL | 当前未采用 `prisma migrate deploy` | 多环境数量增加、需要审计/回滚时 | `Dockerfile`、`start.md`、`prisma/*.sql` |

### 5. 约束与不变量（P0）

| 约束类别 | 关键约束 | 验证方式 | 状态 |
|---|---|---|---|
| 数据 | embedding 与数据库 vector 均为 1024 维 | `scripts/verify-pgvector.ts`、schema、RAG 测试 | 满足于代码层；生产数据抽样资料缺失 |
| 数据隔离 | RAG、文档、记忆、对话和学习数据按 `userId` 约束 | route 代码、`knowledge-isolation.test.ts`、Prisma relations | 部分已验证 |
| 认证 | Better Auth 所需模型必须与数据库同步 | `/api/auth/get-session` 运行验证；schema 与 `db push` | 当前本地已验证；线上资料缺失 |
| WebSocket | Origin 校验、Cookie 鉴权、按用户限流、心跳清理 | `websocket-origin.test.ts`、`websocket-rate-limit*.test.ts`、代码 | 已验证关键纯逻辑；真实多连接压力资料缺失 |
| 消息 | `clientId` 对同一用户幂等，重复请求不重复广播/触发 AI | `message-idempotency.test.ts`、`server/index.ts` | 已验证关键逻辑 |
| AI 额度 | agent/studio/chatroom 计入用户限额，系统后台消耗仍记账 | `src/lib/usage.ts`、usage tests | 已验证纯函数；真实计费数据资料缺失 |
| 安全 | Markdown `rehype-raw` 输入必须经过白名单净化 | `markdown-sanitize.test.ts`、`markdown-sanitize-schema.ts` | 2 个相关 suite 当前因 `unified` 依赖缺失未能运行 |

### 6. 成功经验（P0）

- **把失败模式写进代码契约**：模型池区分额度错误、网络错误和可继续降级的错误；Agent 还修复过“失败伪装成成功”的问题，依据 Git 提交 `7389cac`、`a928ff8`、`41bc6ce`。
- **关键数据链路有幂等和隔离意识**：聊天室消息使用 `clientId` 幂等；知识库同名覆盖采用事务化处理；检索结果显式记录 `scoreSource`，避免把 rerank 分数和向量分数混为一谈。
- **对性能问题做过量化而非只凭感觉**：聊天室保留了虚拟化方案的实测结论，并通过窗口化 + 锚点补偿降低跳动，依据 `README.md`、`chat-room.tsx` 和 `scripts/bench-render.tsx`。
- **AI 功能有评测/验证资产**：`eval/rag/`、`eval/memory/` 和多份 `scripts/verify-*.ts` 说明项目已经开始把 AI 质量从手工体验转向可复验流程。
- **安全边界有显式防线**：WebSocket 在握手前校验路径和 Origin；Markdown 渲染有白名单测试；Cron/后台入口有 fail-closed 相关提交和测试。

### 7. 失败与事故（P0）

| 问题 | 影响 | 根本原因 | 修复措施 | 自动阻止方式 | 状态 |
|---|---|---|---|---|---|
| PostgreSQL 缺少 `vector` 类型 | `npx prisma db push` 无法建表 | 连接到了不带 pgvector 的其他 PostgreSQL 容器/端口 | 切换到项目 pgvector 容器，启用 `vector` extension 后重新 push | 启动前检查镜像、端口和 `pg_extension`；增加环境自检 | 本地已修复；尚无自动门禁 |
| Better Auth schema mismatch | `npm run dev` 加载认证配置失败 | Better Auth 升级后 `Account`/`Verification` 结构没有同步到 schema/数据库 | 增加 4 个 Account 字段和 `Verification` 模型，执行 generate + db push，重启旧 Next 进程 | 将认证 schema 检查加入 CI/部署前 smoke test | 本地已验证；线上资料缺失 |
| Markdown 测试依赖缺失 | 2 个 suite 无法启动，380 个断言中 380 个已运行断言通过 | 当前 `node_modules`/lockfile 与 package.json 依赖状态不一致，缺少 `unified` 等包 | 尚未执行修复 | `npm ci` 后在 CI 强制运行完整 test；依赖变更必须同步 lockfile | 待处理 |
| 类型检查依赖/类型缺失 | `npm run typecheck` 失败 | 缺少 `dotenv`、`unified`、`remark-*` 类型/包，且 Three.js 存在类型问题 | 尚未执行修复 | 干净环境 CI 通过后才允许合并 | 待处理 |
| 数据库迁移漂移 | 新环境、线上环境可能缺表/缺列/缺索引，且错误可能在启动时才暴露 | 没有迁移版本、执行台账和部署前 schema smoke test | 已形成较详细手工 SQL 和部署文档，但未形成自动化闭环 | 版本化 migration 或至少维护幂等台账与 schema diff | 待处理 |

### 8. 领域与接口（P1）

#### 核心领域概念（已证实）

`User` 是所有学习与 AI 数据的归属根；`Plan` 包含 `PlanTask`；`Checkin` 记录学习行为；`StudyRecord` 记录番茄钟真实时长；`Document`/`DocumentChunk`/`KnowledgeDoc` 构成文档与知识库；`Conversation`/`ConversationMessage`/`AgentState` 构成对话和短期上下文；`Memory`/`MemoryEvent`/`MemoryCandidate` 构成长记忆 v2；`TokenUsage` 负责 AI 用量观测；`ChatMessage` 是聊天室持久化消息。

#### 接口资料状态

API 路由和 WebSocket 协议实际存在于 `src/app/api/` 与 `server/protocol.ts`，但没有独立的 OpenAPI/接口契约文档。特别是流式 AI 响应、工具调用 parts、WebSocket 错误码、幂等冲突和 `scoreSource` 需要补成稳定契约，否则前后端改动容易只在运行时暴露。

### 9. 运维与安全（P1）

- **部署方式（已证实）**：Docker Compose 管理 `postgres`、`app`、`nginx` 三个服务；`Dockerfile` 使用 Node 22 Alpine 多阶段构建；入口脚本后台启动 WebSocket，前台运行 Next standalone。
- **当前可观察性（已证实）**：应用有 console 日志、WebSocket `/health`、Postgres healthcheck；没有发现指标、集中日志、错误追踪、告警或 SLO 配置。
- **最大安全风险（已证实/合理推断）**：`scripts/deploy.sh` 会把 `.env` 通过 `scp` 上传服务器；Compose 使用环境变量注入密钥；仓库虽然忽略 `.env`，但部署流程仍依赖人工保护服务器和传输链路。复盘只记录风险，不复制任何凭据。
- **数据库风险（已证实）**：Compose 数据卷持久化，但未发现自动备份、备份校验和恢复演练脚本。
- **网络风险（已证实）**：Nginx 配置监听 80，当前文档仍把 HTTPS 列为路线图项目；线上是否已有外部 TLS 终止层，资料缺失。
- **生产故障入口（部分已有）**：`docker compose logs -f`、部署脚本和 `1112复盘文档和部署文档.md`；没有统一的“故障 → 检查 → 回滚”Runbook。

### 10. 遗留问题与风险（P1）

| 风险或问题 | 影响 | 可能性 | 应对方案 | 负责人 | 状态 |
|---|---|---|---|---|---|
| 数据库没有版本化迁移和执行台账 | 线上 schema 漂移、无法可靠回滚 | 高 | 引入 Prisma migrate 或建立带版本/环境/校验结果的 SQL 台账 | 待指定 | 未开始 |
| npm 与 pnpm 文件并存 | 本地、CI、Docker 解析不同依赖树 | 高 | 保留 npm 单一方案，移除或明确隔离 pnpm 文件，并在干净环境重装验证 | 待指定 | 未开始 |
| 关键依赖未被完整测试发现 | CI/新机器可能在安装或测试阶段失败 | 高 | 对齐 `package.json` 与 `package-lock.json`，执行干净 `npm ci && npm run check` | 待指定 | 未开始 |
| 生产监控、备份、恢复证据缺失 | 故障发现晚、数据恢复不可验证 | 中-高 | 增加 DB 备份/恢复演练、HTTP/WS/AI/DB 监控和告警 | 待指定 | 未开始 |
| 文档与实际结构漂移 | 新人按错误步骤初始化或误判表结构 | 高 | 更新 README 的 Prisma 版本、表数量、Better Auth 初始化和线上迁移说明 | 待指定 | 未开始 |
| 超大模块维护成本高 | 改动影响范围和 review 成本上升 | 中 | 按领域拆分 `chat-room.tsx`、`markdown-studio.tsx`、`model-pool.ts`、`agent/runtime.ts` | 待指定 | 暂缓 |
| AI 质量指标缺少生产基线 | 无法判断模型升级/降级是否损伤学习效果 | 中 | 建立离线集 + 线上采样，跟踪 RAG Recall@K、rerank 命中、回答人工评分和成本 | 待指定 | 未开始 |

## 四、专项实战说明：核心技术实现

### 4.1 数据库从 0 到 1：从打卡数据到 AI 学习平台的数据底座（重点）

#### 解决的问题与核心价值

数据库不是本项目的被动存储层，而是学习闭环、AI 上下文、知识检索、消息一致性和用量治理的共同事实来源。它需要同时守住四类不变量：用户数据隔离、业务写入一致性、AI 状态可追溯、并发操作幂等。任何一类失效，都会分别导致跨用户数据泄露、知识库新旧版本混杂、记忆无痕覆盖、重复消息或重复任务。

#### 0→1 演化过程（已证实）

| 阶段 | 时间与证据 | 数据设计变化 | 解决的问题与代价 |
|---|---|---|---|
| 0. 打卡工具起点 | `c26d0a9`（2026-07-09）的 `schema.prisma` | MySQL；9 个模型：认证、计划、打卡、学习记录、徽章、AIHistory | 快速支撑单体打卡产品；领域简单，但 AI 历史只是问答日志，缺少任务、知识和上下文结构 |
| 1. 学习与 AI 领域扩张 | 2026-07 至 2026-08 的 schema 提交 | 增加任务、待办、对话、Agent 运行/步骤、长期记忆、文档与知识分片；从 MySQL 迁至 PostgreSQL | 数据开始按领域拆分；迁移主要依赖 schema 同步和人工操作，技术债随功能增长 |
| 2. 生产化存储 | `docker-compose.yml`、`d7f7366` 之后的配置 | PostgreSQL 16 容器、命名 volume、Prisma；应用和 WebSocket 共用数据库 | 支持 JSON、事务和后续 pgvector；部署环境开始成为 schema 的一部分 |
| 3. 向量检索入库 | `9c76d78`、`prisma/migrate-to-pgvector.sql` | embedding 从 `jsonb` 一次性转换为 `vector(1024)`；启用 pgvector；为文档与 v1 记忆建立 HNSW | 消除应用层最多 1000 条全量拉取/JS 余弦的静默截断；引入数据库扩展、原生 SQL和维度契约 |
| 4. AI 状态可追溯 | `7deb4c9`、`1c52752` | Memory v2 的实体/事件/候选队列；AgentState 的摘要、工作记忆、待办和 version 乐观锁 | 记忆从“可覆盖文本”升级为带生命周期与事件审计的状态；模型数量和迁移复杂度上升 |
| 5. 数据库裁决并发 | `c23552f`、`add-chat-message-idempotency.sql`、`add-agentrun-daily-unique.sql` | `(userId, clientId)` 与 `(userId, runDate)` 唯一约束；P2002 作为并发胜负结果 | 把“先查再写”的竞态转成数据库原子裁决；要求客户端稳定复用幂等键、应用正确识别唯一冲突 |
| 当前形态 | `prisma/schema.prisma` | PostgreSQL + Prisma 7 + pgvector；28 个模型、18 个手写 SQL 文件 | 已形成覆盖业务、AI、实时协作和运营的统一数据层；最大短板是迁移治理尚未版本化闭环 |

#### 当前领域模型全景（已证实）

| 领域 | 模型 | 关键关系/职责 |
|---|---|---|
| 身份认证 | `User`、`Session`、`Account`、`Verification` | `User` 是主要聚合根；Session/Account 级联删除；与 Better Auth schema 强绑定 |
| 学习闭环 | `Plan`、`PlanTask`、`Todo`、`Checkin`、`StudyRecord` | 计划拆任务，任务完成可产生打卡；真实专注时长以 StudyRecord 为准，Checkin.hours 已是遗留口径 |
| Agent 执行 | `AgentRun`、`AgentStep`、`Conversation`、`ConversationMessage`、`AgentState` | 运行与步骤可追踪；消息同时保留纯文本、展示 parts 和受限 rawParts；AgentState 一对一保存会话压缩状态 |
| 长期记忆 | `UserMemory`、`Memory`、`MemoryEvent`、`MemoryCandidate` | v1 保留作回滚点；v2 拆出实体、不可变事件和异步候选队列，支持生命周期、来源和向量召回 |
| 文档与 RAG | `Document`、`KnowledgeDoc`、`DocumentChunk` | 编辑文档、知识原文、检索分片分开存；分片 embedding 为 `vector(1024)` |
| 实时与通知 | `ChatMessage`、`Notification`、`Announcement` | 聊天持久化/引用/幂等；用户通知与全站公告分离 |
| 模板、资产与成本 | `PlanTemplate`、`DocumentTemplate`、`AvatarChange`、`TokenUsage` | 新用户模板克隆、头像历史、按用户/入口/模型记录 AI token |

这里最值得保留的设计不是“表多”，而是数据职责逐渐从页面结构中独立出来：知识原文与分片分离、对话文本与工具 parts 分离、记忆实体与事件分离、运行与步骤分离、打卡与真实计时分离。这些拆分让检索、回滚、审计和统计各自拥有稳定入口。

#### 从 schema 到运行的完整链路

```text
业务/API/WS 入口
  → src/lib/prisma.ts 或 server/db.ts 创建 Prisma Client + PrismaPg adapter
  → 普通字段走 Prisma CRUD；vector/部分锁语义走参数化 raw SQL
  → schema.prisma 提供模型、关系、普通索引和唯一约束
  → prisma/*.sql 补充 pgvector、HNSW、部分唯一索引、回填与细粒度 ALTER
  → PostgreSQL 16 + pgvector 容器持久化到 postgres_data
  → 单测/集成测试/校验 SQL 验证隔离、幂等、维度、索引和回填结果
```

- **客户端装配（已证实）**：`src/lib/prisma.ts` 使用 Prisma 7 的 `PrismaPg` adapter，并在开发环境把 client 缓存在 `globalThis`，避免热更新重复建连接。WebSocket sidecar 通过 `server/db.ts` 使用同一生成 client 和数据库。
- **生成物管理（已证实）**：Prisma Client 输出到 `prisma/generated/`，不放入 `src/`，避免每次 generate 重写大量文件并触发 Next 开发模块图整体失效。
- **关系与删除策略（已证实）**：多数用户私有数据通过 `userId` 归属，并在用户删除时级联；聊天消息的用户和引用使用 `SetNull`，保留公共聊天历史。并非所有 `userId` 都有外键，例如 `AvatarChange` 只有索引，因此“全库均由外键保证用户归属”不能成立。
- **索引策略（已证实）**：普通访问路径围绕 `userId + 状态/时间/来源`；向量查询使用与 `<=>` 余弦距离匹配的 HNSW `vector_cosine_ops`；Memory v2 的“同用户、同 normalizedKey 只能有一条 active”由 Prisma 无法表达的部分唯一索引守护。
- **一致性策略（已证实）**：知识库覆盖写采用事务和 transaction-level advisory lock；Memory v2 的实体变更与事件在事务中提交；AgentState 用 version 条件更新实现乐观锁；聊天室和每日 Agent 运行依赖唯一约束裁决并发。
- **兼容策略（已证实）**：新增消息 `parts/rawParts`、对话 `surface/refId` 等字段采用 nullable/默认值，让历史行自然降级；v1 `UserMemory` 不删除，为 Memory v2 提供回滚点。

#### 关键设计与权衡

1. **Prisma 负责主模型，SQL 负责数据库特性。** 这让常规 CRUD 保持类型安全，同时能使用 pgvector、HNSW、advisory lock 和部分唯一索引。代价是事实来源被拆成 `schema.prisma` 与 18 个 SQL 文件，开发者不能只看其中一边。
2. **数据库约束优先于应用层“先查后写”。** `(userId, clientId)`、`(userId, runDate)` 和 Memory active key 都把并发冲突交给 PostgreSQL 原子裁决，比先查询再插入可靠；代价是错误处理必须区分目标 P2002 与其他数据库错误。
3. **慢且易失败的外部调用放在事务外。** 知识库先分片并生成 embedding，再进事务替换旧分片，避免长事务和 embedding 失败导致旧数据先被删；代价是两次同名上传需要 advisory lock 在写阶段串行化。
4. **保留旧数据，而不是强行重塑历史。** Memory v2 回填不伪造 normalizedKey/validTo，旧表保留，历史对话字段允许 null；这降低迁移风险，但会长期存在双模型和兼容分支。
5. **当前迁移策略偏向“快速演进”。** 本地 `db push`、存量库手写 SQL能降低单人项目早期成本；当环境增多或需要审计、回滚时，它已经不再经济，应重新评估为版本化 migration。

#### 初始化、变更与排障实战

**全新本地库（现有文档链路，尚未在本次执行）：**

```bash
docker compose up -d postgres
npx prisma generate
npx prisma db push
npx prisma db seed
```

前提是 `DATABASE_URL` 指向带 pgvector 的 PostgreSQL 16。首次建库后还必须确认 `vector` extension 和 Prisma 无法表达的索引是否存在；仅执行 `db push` 不能证明 HNSW、部分唯一索引和历史回填均已完成。

**已有数据库变更：** 不应把 `db push` 当成无差别生产迁移。仓库现状是按变更选择 `prisma/*.sql`，使用 `psql -v ON_ERROR_STOP=1` 执行，并运行脚本末尾的校验查询。部分脚本幂等，`migrate-to-pgvector.sql` 明确是一次性的；执行顺序和目标环境必须进入台账，不能靠文件名猜测。

**最小检查清单：**

```sql
SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';
SELECT table_name, column_name, udt_name
FROM information_schema.columns
WHERE column_name = 'embedding';
SELECT tablename, indexname, indexdef
FROM pg_indexes
WHERE indexname LIKE '%hnsw%' OR indexname LIKE '%clientId%' OR indexname LIKE '%runDate%';
```

还应检查 Better Auth 的关键表/列、28 个预期模型对应的表、Memory v2 的回填行数与 import 事件数，并通过 `/api/auth/get-session`、知识库上传/检索和 WebSocket 重试做应用层 smoke test。以上是建议验收路径，不代表本次已经连接真实数据库执行。

#### 验证与验收状态

| 验证项 | 证据与本次状态 | 结论 |
|---|---|---|
| schema 静态一致性 | 已检查 `schema.prisma`、Prisma 配置、Docker 和 18 个 SQL 文件 | 当前代码设计已证实；实际环境是否一致资料缺失 |
| 知识库事务顺序 | `tests/knowledge-upload.test.ts` 验证锁先于删除、失败回滚等调用关系 | 单测层已覆盖 |
| 用户隔离与真实 pgvector | `tests/integration/knowledge-isolation.test.ts` 覆盖真实 PostgreSQL 上同名文件隔离、覆盖和检索 | 测试资产已存在；本次 Docker daemon 未运行，未执行 |
| 消息幂等 | `tests/message-idempotency.test.ts` 覆盖首次写、P2002 重读、载荷冲突 | 本次包含在 380 个通过断言中 |
| 会话乐观锁 | `tests/context-agent-state.test.ts` 覆盖 version 冲突不覆盖新状态 | 本次包含在 380 个通过断言中 |
| pgvector 迁移 | SQL 内含扩展、类型、索引、行数、维度校验；另有 `scripts/verify-pgvector.ts` | 验证工具已存在；本次未连接数据库执行 |
| 完整质量门禁 | 2026-11-27 执行 `npm run check` | typecheck 先失败，完整门禁未通过；lint 单独执行 0 error/36 warnings，测试 41/43 suite 通过、380 assertions 通过 |

#### 风险与改进

- **P0：没有单一、版本化、可审计的迁移链。** `prisma.config.ts` 声明了 `prisma/migrations`，仓库却没有该迁移目录；Docker 启动脚本明确不执行 migration。验收标准：空库和一份脱敏旧库都能用同一版本化流程升级，迁移记录、schema diff、校验和回滚均可追踪。
- **P0：部署不会自动证明 schema 可用。** Compose 只等待 `pg_isready`，这只能证明进程接收连接，不能证明 pgvector、认证列、HNSW 和唯一约束齐全。验收标准：部署前/启动后 smoke test 自动失败关闭，并输出缺失对象而非真实凭据。
- **P1：备份不是恢复能力。** SQL 中的一次性备份表与 Docker volume 只能降低部分操作风险，不能替代定期外部备份、保留策略和恢复演练。验收标准：可恢复到临时库并记录 RPO/RTO、行数和关键查询校验。
- **P1：遗留列和弱关系会继续制造语义债。** `Checkin.hours/mood`、`StudyRecord.checkinId` 命名、`AvatarChange.userId` 无外键等都依赖注释和调用方纪律。验收标准：建立字段级数据字典，逐项决定迁移、重命名、加约束或正式保留。
- **P1：raw SQL 边界需要集中治理。** 当前 raw SQL 主要为参数化值和必要的 pgvector/锁操作，但分散在 RAG、Memory、History 和脚本中。验收标准：建立统一封装/审查清单，并用集成测试覆盖 userId 条件、向量维度和事务边界。

### 4.2 RAG 与知识库：事务化覆盖、租户隔离和库内向量召回

- **解决的问题与核心价值**：把用户上传的 md/pdf/docx/txt 转成可检索知识，同时保证同名文档更新不出现新旧分片混合、不同用户的同名文件不互相覆盖。
- **当前实现（已证实）**：API 提取文本后进入 `src/lib/knowledge-upload.ts`；先做大小限制与分片，再在事务外批量 embedding；事务内取得 `(userId, sourceName)` advisory lock，删除旧 `DocumentChunk`、以 raw SQL 写入 `vector(1024)`、upsert `KnowledgeDoc`。检索由 `src/lib/rag/retriever.ts` 使用 `<=>` 做 pgvector 召回，再进入 rerank，并保留 `scoreSource` 区分分数来源。
- **关键设计与权衡**：原文和分片分表，支持阅读与检索各自演进；embedding 事务外计算保护旧数据，事务内覆盖保证原子可见；数据库内 HNSW 取代 JS 全量余弦，解决超过 1000 个 chunk 后的静默漏检。代价是强依赖 1024 维模型、pgvector 和 raw SQL 契约。
- **实战使用方式**：上传前校验文本/分片上限；上传失败先看文本提取、embedding 数量是否与 chunk 一致，再看 transaction/advisory lock 和向量 cast；检索异常按“用户 chunk 数 → query embedding → pgvector rows → rerank → scoreSource”逐段定位。
- **验证与验收**：单测覆盖分片、向量字面量和覆盖事务；真实数据库集成测试覆盖跨用户隔离与同名覆盖；`eval/rag/` 和 `scripts/eval-rag.ts` 提供质量评测资产。本次只执行默认测试，集成测试因 Docker 未运行而未执行，线上 Recall@K/P95 资料缺失。
- **风险与改进**：P0 是维度/扩展/schema 漂移导致整条链路不可用；P1 是缺少生产检索质量与延迟基线。改进验收应同时包含 schema smoke、隔离集成测试、固定评测集 Recall@K/NDCG、rerank 增益和 P95 延迟。

### 4.3 长期记忆与会话上下文：可追溯状态而非“无限聊天记录”

- **解决的问题与核心价值**：长期对话不能无限塞入模型窗口，也不能让模型抽取出的偏好无痕覆盖。项目把“原始消息、滚动摘要、工作记忆、长期记忆实体、变更事件、抽取候选”拆成不同持久化层。
- **当前实现（已证实）**：`ConversationMessage` 永久保留原始文本，并保存展示 parts/受限 rawParts；`AgentState` 保存 summary、workingMemory、todoList 和压缩边界；`Memory` 保存 active/superseded/expired 等生命周期，`MemoryEvent` 记录 before/after/evidence，`MemoryCandidate` 作为抽取队列。`src/lib/context/` 负责上下文组装，`src/lib/memory/` 负责抽取、验证、检索、写入与维护。
- **关键设计与权衡**：AgentState 使用 version 乐观锁，冲突时不覆盖新摘要；Memory active key 使用部分唯一索引，实体变更与事件同事务；v1 `UserMemory` 保留为只读回滚点。代价是链路和表数量明显增加，且 Memory v1/v2 的切换状态需要文档化。
- **实战使用方式**：上下文异常先检查 conversation 的 surface/refId 绑定，再检查 rawParts 是否能恢复工具调用、AgentState 压缩边界是否前进、MemoryCandidate 是否卡在 pending/running/failed，最后检查 MemoryEvent 是否存在对应操作证据。
- **验证与验收**：默认测试覆盖 AgentState 解析上限和乐观锁；仓库有 `eval/memory/`、`verify-memory*.ts`、`verify-context.ts`。本次没有真实模型/API/数据库环境，因此“记忆提升回答质量”和“生产候选队列无积压”均为资料缺失。
- **风险与改进**：P1 建立抽取准确率、冲突修正率、注入命中率、候选积压和事件完整率；明确 v1 停写/回滚/最终下线条件；增加“Memory 写入必有 Event”的真实数据库约束或集成测试。

### 4.4 WebSocket 聊天室：鉴权、消息幂等与 AI 流式隔离

- **解决的问题与核心价值**：独立 sidecar 承担多人实时消息与 `@AI` 流式回复，必须避免未登录连接、跨站握手、重复广播、重复触发 AI，以及某次 AI 异常拖垮整个进程。
- **当前实现（已证实）**：Nginx `/ws` 转发到 `server/index.ts`；握手阶段校验路径、Origin 和 Better Auth Cookie；连接后有心跳、用户级限流和断线清理。客户端为发送生成 `clientId`，数据库 `(userId, clientId)` 唯一约束处理并发重试；相同 key 不同载荷返回 `idempotency_conflict`。AI 回复作为独立异步任务，末端 catch 隔离未捕获 rejection，并持久化 ChatMessage、token 用量与 AI 角色。
- **关键设计与权衡**：sidecar 与 Next 分进程，职责清楚且可独立处理长连接；当前房间/连接状态仍在单进程内存中，尚不适合直接水平扩展。数据库唯一约束比进程内去重可靠，但需要每次重试复用 clientId。
- **实战使用方式**：故障按 Nginx upgrade 头 → `/ws` 路径与 Origin → Cookie session → sidecar `/health` → Postgres 写入 → 广播/AI 流逐层检查；重复消息重点检查 clientId 是否复用以及唯一索引是否存在。
- **验证与验收**：单测覆盖 Origin、限流、协议和 P2002 幂等；默认测试本次通过相关断言。真实多连接压力、断线重连、代理超时和生产 AI 流稳定性资料缺失。
- **风险与改进**：P1 增加并发连接/消息压测、连接数与错误码指标、sidecar 单独健康与重启告警；若需要多副本，重新设计共享房间广播、跨实例限流和 AI 任务归属。

## 五、后续行动

| 优先级 | 行动项 | 负责人 | 截止日期或触发条件 | 验收标准 | 状态 |
|:---:|---|---|---|---|---|
| P0 | 固化数据库迁移策略与 schema 基线 | 待指定 | 下一次线上 schema 变更前 | 空库和脱敏旧库均能按同一版本化流程升级；有执行台账、schema diff、校验 SQL 和回滚说明 | 未开始 |
| P0 | 把 Better Auth、pgvector 和 Prisma schema smoke test 加入部署前检查 | 待指定 | 下次部署前 | 自动确认 `vector` extension、关键认证表/列、Prisma Client 和 `/api/auth/get-session` 可用 | 未开始 |
| P0 | 修复依赖安装与完整检查链路 | 待指定 | 合并新功能前 | 干净目录执行 `npm ci && npm run typecheck && npm run lint && npm test` 全部通过；当前 2 个缺依赖 suite 不再失败 | 未开始 |
| P1 | 建立生产备份与恢复演练 | 待指定 | 上线前/每月 | 备份可生成、可恢复到临时库，并记录最近一次成功时间和恢复耗时 | 未开始 |
| P1 | 补齐接口和错误码契约 | 待指定 | 下次前后端协议变更前 | API、流式响应、WS 消息/错误、幂等行为和 RAG 结果字段有可审阅文档或契约测试 | 未开始 |
| P1 | 建立 AI 质量与成本基线 | 待指定 | 模型池再次升级前 | RAG、记忆、Agent 和聊天室各有最小评测集、质量指标和 token/延迟记录 | 未开始 |
| P1 | 收敛文档入口 | 待指定 | 下一次环境变更后 | README、`start.md`、部署复盘文档的版本、表数量、启动命令和迁移说明一致 | 未开始 |
| P1 | 建立数据库字段字典与遗留字段处置清单 | 待指定 | 下一次领域模型调整前 | `Checkin.hours/mood`、`StudyRecord.checkinId`、弱外键字段均有明确语义、保留/迁移决定和验证查询 | 未开始 |
| P2 | 拆分超大组件和服务模块 | 待指定 | 发生跨模块高频改动时 | 单文件职责收敛，拆分后现有测试和关键行为不回归 | 暂缓 |

## 六、复盘结论

- **一句话总结**：Summer Checkin 已从打卡工具演进为功能密度较高的 AI 学习平台，工程上最值得肯定的是 AI 失败兜底、数据隔离、幂等和评测意识；最需要补齐的是多环境数据库治理和可重复交付。
- **最值得保留的经验**：把 AI 当作有成本、有失败、有上下文和有数据副作用的系统来设计，而不是只包一层模型调用；模型池、token 账本、记忆事件、RAG scoreSource、WS 幂等和安全净化都体现了这一点。
- **最需要避免的错误**：先把功能推进到数据库和线上，再依赖人工记忆补迁移；`vector` 类型缺失、Better Auth 字段缺失和依赖缺失都说明部署前验证应该自动化。
- **下次需要验证的假设**：模型降级是否保持可接受质量；RAG/记忆是否真实提升学习结果；生产数据能否在故障后恢复；线上 schema 是否与当前代码完全一致。
- **需要更新的文档**：建议后续更新 `README.md`、`start.md` 和 `1112复盘文档和部署文档.md`，并新增数据库执行台账、生产 Runbook、API/WS 契约和 AI 质量基线。当前本次复盘只创建主复盘文件，未创建这些配套文档。

## 七、审计边界与证据缺口

- 未读取或复制 `.env` 中的真实密钥、Token、数据库密码或个人隐私；只检查了变量名和配置引用位置。
- 未访问线上 ECS、线上数据库、云厂商控制台、真实用户数据、生产日志和外部监控，因此线上状态、备份、TLS、密钥轮换、真实性能和真实 AI 质量均不能视为已验证。
- 未执行生产部署、数据库迁移、代码修改或 Git 提交；本次只运行了本地只读审计和既有检查命令，并创建本复盘文档。
- 2026-11-27 本地验证记录：`npm run lint` 为 0 error / 36 warnings；`npm test` 运行 43 个 suite，其中 41 个通过、2 个因缺少 `unified` 等依赖未启动，已运行测试共 380 个通过；`npm run typecheck` 因 `dotenv`、`unified`、`remark-*`、Three.js 类型等缺口失败，因此 `npm run check` 未通过。
- 本次 Docker daemon 未运行，`docker compose ps` 无法连接；因此没有执行 `test:db`、pgvector 校验脚本或任何真实数据库写入，数据库集成与线上状态均不能写成已验证。
- 文档冲突：`README.md` 仍写 Prisma 5、27 张表；当前 `package.json`/lockfile 使用 Prisma 7，当前 schema 有 28 个 model（新增认证 `Verification`）。这属于文档漂移，不应按 README 数字判断当前数据库真实状态。
