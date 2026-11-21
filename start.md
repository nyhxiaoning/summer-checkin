# 项目分析:Summer Checkin

> 生成日期:2026-11-08 · 基于对仓库的只读阅读(未修改任何代码)

## 1. 项目是做什么的

**Summer Checkin(夏签)** 是一个全栈学习成长平台,口号是「让每一次专注,都留下生长的痕迹」。核心功能:

- **🏝️ 学习小岛打卡**:每日签到打卡,3D 学习岛屿(Three.js)随累计专注时长成长,连续天数 + 年度热力图
- **📋 计划与任务**:计划 → 任务 → 打卡三方联动;「文档工作室」用 Markdown 写计划,AI 自动拆解成任务;独立今日待办
- **🍅 专注番茄钟**:自定义时长(默认 25+5),完成自动写入学习记录,有沉浸模式
- **🤖 AI 智能体**:分析学习数据生成个性化计划(关键步骤人工确认)、文档拆任务、长期记忆、每周报告;智能体页为 "Rainforest Explorer"
- **📚 知识库 RAG**:上传 md/pdf/docx/txt,pgvector 向量检索(HNSW 索引在库内完成),基于你的文档回答
- **💬 实时聊天室**:WebSocket 群聊,引用回复,`@AI` 唤起 AI 伙伴流式回复(Markdown/代码高亮),游标分页 + 消息窗口
- **👤 账号体系**:Better Auth 注册/登录,头像阿里云 OSS 预签名直传、历史头像可回滚
- **✨ 场景主题**:雨林/雪日/暖云三套主题 + 环境音

## 2. 技术栈

| 层 | 技术 |
|---|---|
| 框架 | **Next.js 16.2(App Router)** + React 19.2 + TypeScript 5 |
| ⚠️ 特殊注意 | 本项目的 Next 是**非训练数据里的版本**:`middleware.ts` 已改名为 **`src/proxy.ts`**;构建默认 **Turbopack**;开启 `reactCompiler`;异步 Request API 为破坏性变更。写代码前必须读 `node_modules/next/dist/docs/`(见 AGENTS.md) |
| 样式 | Tailwind CSS v4 + shadcn/ui + `class-variance-authority` |
| 3D/动效 | Three.js(@react-three/fiber/drei)、GSAP、Motion(按需动态加载) |
| 数据库 | PostgreSQL 16 + **pgvector**(`vector(1024)` embedding,HNSW 索引)+ Prisma 7(schema 走 `prisma-client` 生成器,输出到 `prisma/generated/`) |
| 后端 | Next.js API Routes(24 个 `route.ts`)+ Server Components 直连数据库 |
| 实时 | **独立 WebSocket sidecar**(`server/`,`ws` 库,:3001),Nginx 把 `/ws` 反代过去;握手期 Cookie 鉴权、心跳保活 |
| AI | OpenAI 兼容 API,**模型池**(`src/lib/model-pool.ts`,LOW/HIGH 两档 + 降级链,默认 Agnes 中国站;Embedding 走阿里百炼)、Vercel AI SDK、Zod |
| 认证 | Better Auth(session cookie) |
| 文件存储 | 阿里云 OSS(头像预签名直传) |
| 测试 | Vitest(43 个单测文件,纯逻辑;集成测试单独配置需真实 PG) |
| 部署 | Docker Compose(nginx + app + postgres)、GitHub Actions CI |
| 包管理 | **npm**(package-lock.json / Dockerfile `npm ci` / CI 均用 npm) |

## 3. 目录结构

```
summer-checkin/
├── src/
│   ├── app/                    # App Router
│   │   ├── page.tsx            # 落地页(未登录),已登录 redirect /dashboard
│   │   ├── layout.tsx          # 根布局(html/body、全局样式)
│   │   ├── proxy.ts            # ⚠️ Next 16 的 middleware(路由守卫,未登录跳 /login)
│   │   ├── (auth)/             # 登录 / 注册
│   │   ├── (dashboard)/        # 业务主区(自带 layout.tsx:requireAuth + TopNav)
│   │   │   ├── checkin/        # 小岛打卡
│   │   │   ├── dashboard/      # 仪表盘
│   │   │   ├── plans/          # 计划(含 [id]、[id]/studio、new)
│   │   │   ├── docs/           # 阅读 + 知识库
│   │   │   ├── statistics/     # 个人统计主页
│   │   │   ├── profile/        # 个人资料
│   │   │   └── actions.tsx     # Server Actions(退出登录等)
│   │   ├── agent/              # AI 智能体页(独立于 dashboard 组)
│   │   └── api/                # 24 个 route.ts(chat/ai/knowledge/plans/todos/oss/...)
│   ├── components/             # 按域分包:chatroom/ island/ plans/ studio/ agent/
│   │   │                       #   focus-room/ statistics/ landing/ layout/ ui(shadcn)/
│   ├── lib/                    # 业务逻辑:agent/ ai/ context/ memory/ rag/ tools/ studio/
│   │   │                       #   model-pool.ts、auth-utils.ts、prisma.ts、usage.ts...
│   ├── context/                # React 全局状态(scene-context 等)
│   ├── config/ styles/ types/  # 配置、全局样式、共享类型
├── server/                     # WebSocket sidecar 独立进程(index.ts + room/auth/rate-limit...)
├── prisma/                     # schema.prisma + seed.ts + 18 个手工 SQL 脚本 + generated/
├── tests/                      # Vitest 单测(纯逻辑) + tests/integration/(需真实 PG)
├── scripts/                    # 部署、RAG/记忆 eval、验证、密码重置、PDF/Word 提取...
├── eval/                       # RAG / memory 评测数据
├── nginx/                      # 反代配置(80 → app:3000 + /ws → :3001)
├── Dockerfile / docker-compose.yml
├── .github/workflows/ci.yml    # typecheck + lint + test(不跑 next build)
├── context.md                  # 历史任务规格文档(上下文工程实施任务)
└── README.md                   # 最完整的文档
```

## 4. 启动、构建、测试命令

### 首次准备

```bash
npm install                     # 注意:用 npm,不是 pnpm
cp .env.example .env            # 填 DATABASE_URL、AGNES_API_KEY、BETTER_AUTH_SECRET 等
npx prisma generate             # 生成客户端到 prisma/generated/
npx prisma db push              # 建表(本项目不用 migrate,走 db push + 手工 SQL)
npx prisma db seed              # 种子数据
```

数据库必须是 **pgvector/pgvector:pg16** 镜像(embedding 列是 `vector(1024)`)。

### 日常开发(两个进程)

```bash
npm run dev        # Next.js :3000
npm run ws:dev     # WebSocket sidecar :3001(聊天室和 @AI 依赖它)
```

### 检查与测试

```bash
npm test           # 全部单测(vitest,纯逻辑,无需数据库/API Key)
npm run test:watch # 监听模式
npm run test:db    # 集成测试(需真实 PostgreSQL)
npm run typecheck  # tsc --noEmit
npm run lint       # eslint
npm run check      # typecheck + lint + test 一把梭(CI 同款)
```

### 构建与部署

```bash
npm run build && npm run start   # 生产构建/启动(需真实环境变量)
docker compose up -d --build      # 一键部署(nginx + app + postgres)
bash scripts/deploy.sh            # 构建镜像并上传部署
```

CI(`.github/workflows/ci.yml`)只跑 `prisma generate` → `next typegen` → `typecheck` → `lint` → `test`,**刻意不跑 `next build`**(RSC 直连数据库,需要真实环境变量;构建交给 Dockerfile)。

### 其他实用脚本

```bash
npm run eval:rag / eval:memory     # RAG / 记忆评测
npm run verify:memory / verify:context / verify:rere...  # 各类验证(tsx scripts/*.ts)
npm run memory:reprocess            # 记忆重处理
```

## 5. 新增一个页面应该从哪里开始

1. **先读内置文档**:`node_modules/next/dist/docs/01-app/01-getting-started/03-layouts-and-pages.md`(AGENTS.md 强制要求——这是 Next 16,与训练数据不同)。
2. **决定路由组**:
   - 需要登录 + 顶部导航 → 放 `src/app/(dashboard)/<你的路由>/page.tsx`(自动继承 `(dashboard)/layout.tsx` 的 `requireAuth()` + TopNav,未登录自动跳 `/login`)
   - 公开页面 → 直接 `src/app/app/<路由>/page.tsx`,注意 `src/proxy.ts` 会把不在白名单的路径重定向到登录
3. **写页面**:默认导出 React 组件;本项目惯例是 **Server Component 直接 `await requireAuth()` + `prisma.*` 查数据**(参考 `src/app/(dashboard)/statistics/page.tsx`),交互部分再拆 `"use client"` 组件放进 `src/components/<域>/`。
4. **加导航入口**(如果是主功能页):编辑 `src/components/layout/top-nav.tsx` 里的 `STATIC_NAV` / `STATIC_FULL_NAV` 数组。
5. **需要后端接口** → 在 `src/app/api/<资源>/route.ts` 加 Route Handler;**需要 Server Action** → 放 `(dashboard)/actions.tsx` 或就近的 `actions.tsx`。
6. **数据模型变更** → 改 `prisma/schema.prisma` 后 `npx prisma generate` + `npx prisma db push`;**非结构性变更(加索引、加列默认值等)写手工 SQL 放 `prisma/*.sql`**(本项目的迁移策略,见 Dockerfile 注释)。
7. **补测试** → 纯逻辑放 `tests/*.test.ts`(默认配置);需要数据库的放 `tests/integration/`(单独配置,不进默认 `npm test`)。
8. **跑 `npm run check`** 确认 typecheck + lint + test 全绿。

## 6. 当前项目有哪些明显维护风险

1. **⚠️ 双包管理器状态混乱(当前最紧迫)**:仓库里同时存在已跟踪的 `package-lock.json` 和**未跟踪的 `pnpm-lock.yaml` + `pnpm-workspace.yaml`**。而 Dockerfile、CI、README 全部按 npm(`npm ci`)。更糟的是 `pnpm-workspace.yaml` 内容是未完成的占位符(`'set this to true or false'`),一旦被 pnpm 读到会出问题。应删除这两个 pnpm 文件,或全仓统一迁移到 pnpm。
2. **Next.js 16 认知风险**:AGENTS.md 明确警告「这不是你认识的 Next.js」。`middleware.ts` → `proxy.ts`、Turbopack 默认、异步 Request API 等破坏性变更,任何凭训练数据记忆写代码的 AI/新人都会踩坑。依赖内置文档的强制约定目前只写在 AGENTS.md 里,新人未必读。
3. **数据库迁移无版本管理**:没有 `prisma/migrations/`,只有 `prisma db push` + 18 个**手工 SQL 脚本**。多环境(本地/生产/CI)极易出现 schema 漂移——没人能确定哪个环境执行过哪些 SQL。建议引入 `prisma migrate` 或至少做一份「SQL 执行台账」。
4. **CI 不跑 `next build`**:类型检查靠 `next typegen`,但 RSC 页面里需要真实数据库的代码路径、构建期错误(如 server-only 模块被拉进客户端)只能在 Docker 构建或部署时才发现。README 已注明原因,但这是构建期问题的盲区。
5. **巨型组件**: `chat-room.tsx` **1410 行**、`markdown-studio.tsx` 1152 行、`model-pool.ts` 979 行、`agent/runtime.ts` 874 行。聊天室虽然用注释记录了「不做虚拟化」的量化结论(可贵),但单文件体量让改动风险和 review 成本都很高。
6. **测试只覆盖纯逻辑**:43 个单测全是 node 环境纯逻辑(分片、净化白名单、Zod 边界、限流等),**没有组件测试、没有 E2E**;集成测试只有 1 个文件且不在 CI 跑。前端交互(尤其聊天室滚动窗口、3D 岛屿)回归只能靠人工。
7. **安全敏感面集中在白名单**:`markdown-renderer` 开着 `rehype-raw`,输入包含 AI 输出、用户导入 md、他人聊天消息——XSS 防线完全靠 `markdown-sanitize-schema` 白名单 + 测试锁住。这块测试是生命线,任何白名单改动都必须过 `tests/markdown-sanitize.test.ts`。
8. **双进程架构的运维耦合**:聊天室依赖独立 sidecar(`ws:dev` / 容器),本地忘开第二个进程时「聊天室发不出消息」这类问题排查成本高;WebSocket 握手鉴权、Origin 白名单、限流逻辑分布在 `server/` 与 `src/lib/` 两侧。
9. **文档漂移**:README 徽章写 Prisma 5,实际是 Prisma 7;`context.md` 是某次任务的规格文档留在仓库根部,容易被误认为现行设计文档;导航/功能描述散落在 README 大段表格里,没有单一的架构决策记录(ADR)。
10. **依赖面广**:Three.js + GSAP + Motion + highlight.js + KaTeX + rehype 全家桶 + ai SDK…升级任何一个都可能连锁反应(尤其 Markdown 渲染管线插件顺序),而 CI 只能挡住类型层面的破坏。

## 7. 结论

这是一个**功能密度相当高的个人全栈项目**:Next.js 16 App Router + Prisma/Postgres/pgvector + 独立 WebSocket sidecar + 模型池降级的 AI 能力,文档(README)和测试意识(净化白名单回归、纯逻辑单测)在个人项目里属于较好水平。当前最值得优先处理的是 **pnpm/npm 混用的脏状态** 和 **数据库迁移台账缺失**;写任何页面/接口前,先读 `node_modules/next/dist/docs/` 里的对应指南。
