# AI 原生完整链路实现

> 本文说明 Summer Checkin 如何把一次用户输入从“请求进入”推进到“模型选择、工具执行、知识检索、长期/短期记忆注入、流式响应、消息持久化和 token 记账”。  
> 重点不是罗列 AI 功能，而是说明每个技术点对应的实现模块、数据边界、失败兜底和验证方式。

## 一、先看完整链路

```text
用户消息
  │
  ├─ 认证：Better Auth Cookie → userId
  │
  ├─ /api/ai/route.ts
  │    ├─ 识别协议：ui / text
  │    ├─ 识别场景：agent / plan / doc
  │    ├─ 创建或校验 Conversation
  │    ├─ 保存当前 user message
  │    │
  │    └─ buildConversationContext()
  │         ├─ AgentState：滚动摘要、workingMemory、todoList
  │         ├─ 最近原始消息：ConversationMessage
  │         ├─ Scene Context：当前计划/文档/学习状态
  │         └─ Memory v2：稳定画像 + 查询相关记忆 → memory_context
  │
  ├─ create*Tools() 创建工具
  │    └─ pickTools(surface) 按场景 fail-closed 裁剪工具
  │
  ├─ streamTextWithFallback("high", ...)
  │    ├─ 选择 HIGH 模型链
  │    ├─ 首个 chunk 探测真实 provider 响应
  │    ├─ 额度/限流/网络/模型不可用 → 冷却或跳过并降级
  │    ├─ streamText 多步循环，最多 10 步
  │    ├─ LLM 生成 tool-call
  │    ├─ 工具 execute() 读取/写入业务数据库
  │    └─ tool-result 回到下一步模型调用
  │
  ├─ RAG（仅在模型决定调用 searchKnowledgeBase 时触发）
  │    ├─ query → embedding
  │    ├─ PostgreSQL + pgvector 召回
  │    ├─ cross-encoder rerank
  │    └─ 检索结果作为 tool-result 返回模型
  │
  ├─ stream → 浏览器
  │    ├─ agent：UI Message Stream，保留 text/tool parts
  │    └─ studio：纯文本流，保持历史协议兼容
  │
  ├─ onEnd
  │    ├─ 保存 assistant message：content + parts + rawParts
  │    ├─ recordUsage() 写入 tokenusage
  │    └─ scheduleExtraction() 异步排入长期记忆候选
  │
  └─ WebSocket 聊天室是另一条轻量链路
       server/index.ts → server/ai.ts
       LOW 模型链 → 文本流 → 广播 → chatmessage 持久化
```

## 二、技术点与实现模块总表

| 技术点 | 入口/核心模块 | 关键函数或对象 | 数据表/外部依赖 | 主要兜底 |
|---|---|---|---|---|
| 模型池降级 | `src/lib/model-pool.ts` | `streamTextWithFallback`、`completionsWithFallback`、`embeddingWithFallback`、`rerankWithFallback` | Agnes、阿里云百炼、`tokenusage` | 按错误类型冷却/跳过/换模型 |
| 工具调用 | `src/app/api/ai/route.ts`、`src/lib/tools/` | `streamText`、`stopWhen`、`pickTools`、各 `create*Tools` | Plan、Task、Todo、Checkin、Conversation、Memory 等 | `safeExecute`、场景白名单、最多 10 步 |
| RAG 检索 | `src/lib/rag/`、`src/lib/tools/rag-tools.ts` | `searchKnowledge`、`searchSimilarChunks`、`rerankChunks` | `document`、`documentchunk`、`knowledgedoc`、pgvector | 重排失败回退向量序；向量失败返回空/错误边界 |
| 长期记忆 v2 | `src/lib/memory/` | `scheduleExtraction`、`processCandidate`、`commitCandidate`、`retrieveMemories` | `memory`、`memoryevent`、`memorycandidate` | 门控、校验、幂等、候选重试、生命周期维护 |
| 会话短期记忆 | `src/lib/context/` | `buildConversationContext`、`maybeCompressConversation`、`updateAgentStateWithRetry` | `conversation`、`conversationmessage`、`agentstate` | 保留原文、摘要失败保留旧摘要、乐观锁重试 |
| token 记账 | `src/lib/usage.ts` + `model-pool.ts` | `recordUsage`、`getTodayUsage`、`assertInteractiveUsageAllowed` | `tokenusage`、`user.vip`、`AI_TOKEN_LIMIT` | 交互面早检；记录失败不阻断主流程 |
| 流式持久化 | `src/app/api/ai/route.ts`、`src/lib/ai/parts.ts` | `toUIMessageStream`、`toTextStream`、`saveAssistantMessage` | `conversationmessage.parts/rawParts` | 中断保留已生成内容；UI 与 DB 使用同源 parts |
| 聊天室 AI | `server/index.ts`、`server/ai.ts` | `handleAI`、`streamTextWithFallback("low")` | `chatmessage`、虚拟 AI user | 失败文案收尾、广播 `ai:done`、消息限流与幂等 |

## 三、统一 AI 请求入口：`src/app/api/ai/route.ts`

### 3.1 入口负责什么

`POST /api/ai` 是智能体页、计划工作台、文档工作台共用的 AI 入口。它不是简单的“把 messages 转发给模型”，而是承担了整条链路的编排职责：

1. `getAuthUser()` 验证当前用户，未登录返回 401。
2. 解析 `protocol`：
   - `ui`：AI SDK UI Message Stream，保留文本、工具调用和工具结果；智能体页使用。
   - `text`：纯文本流，兼容现有 studio 面板。
3. 根据 `studioContext` 计算 `surface`：`agent`、`plan`、`doc`。
4. 校验已有会话必须属于当前用户，或创建新 `Conversation`。
5. 把本轮用户消息写入 `conversationmessage`。
6. 调用 `buildConversationContext()` 组装 system prompt、短期记忆、长期记忆、场景数据和历史消息。
7. 创建所有候选工具，再由 `pickTools()` 按场景白名单裁剪。
8. 通过 `streamTextWithFallback("high", ...)` 执行多步模型循环。
9. 将流返回浏览器，并在 `onEnd` 中持久化完整 assistant message、记账和排队记忆抽取。

### 3.2 为什么入口要统一编排

认证、限额、上下文、记忆、工具白名单、模型池和持久化如果分别散落在三个页面，很容易出现以下漂移：

- 智能体页有记忆，文档工作台没有记忆；
- UI 协议保存了 tool parts，文本协议只保存最后一步文本；
- 一个场景可以写入不属于当前对象的计划或待办；
- 模型已经超出用户额度，但请求仍然执行；
- 前端看到的流式内容和刷新后数据库里的内容不一致。

当前实现把这些横切能力集中在 `/api/ai`，再通过 `surface` 做受控差异化。

## 四、模型池降级

实现文件：`src/lib/model-pool.ts`

### 4.1 四类模型链

| 链 | 用途 | 当前策略 |
|---|---|---|
| HIGH | 智能体、计划/文档工作台等高质量交互 | Agnes 优先，失败后进入阿里云多个模型快照，再以 Flash 兜底 |
| LOW | 聊天室、标题、记忆抽取、计划拆分、摘要 | Agnes 优先，随后阿里云 Flash 链 |
| Embedding | 知识库和长期记忆向量化 | 固定输出 1024 维，模型失败后切换 embedding 链 |
| Rerank | RAG 召回后的 cross-encoder 精排 | 独立重排链和独立 endpoint/protocol |

模型不是只按“第一个失败就换下一个”处理，而是先分类错误：

- `isQuotaError()`：额度耗尽，标记模型 exhausted；
- `isRateLimitError()`：限流，进入短暂冷却；
- `isModelUnavailableError()`：模型不存在或未开通，跳过；
- `isTransientNetworkError()`：超时、连接重置、DNS/5xx 等，短冷却后降级；
- 参数、协议或响应结构错误：根据链路决定继续尝试或熔断，避免把稳定错误无限重复。

模型状态是进程内状态：进程重启后耗尽/冷却记录会丢失。这是当前实现的一个明确边界，不是持久化熔断系统。

### 4.2 非流式降级

`completionsWithFallback()` 适用于标题生成、记忆抽取、会话摘要、Agent 后台步骤等非流式调用：

```text
getChain(tier)
  → createClientFor(entry)
  → 调用 provider
  → 成功：记录 usage，返回 data + model
  → 可降级错误：标记状态，继续下一个 entry
  → 不可恢复错误：抛给上层
```

调用方只需要声明档位和业务回调，不需要重复写 provider 选择、错误分类和 usage 记录。

### 4.3 流式降级的关键难点

`streamTextWithFallback()` 不能只看 `streamText()` 是否返回对象，因为 AI SDK 可能把 provider 的 403/错误作为流中的 `error` part 发出，而不是让 Promise reject。

因此实现了 `peelFirstChunk()`：

1. 读取流开始部分；
2. `start` 不算 provider 成功；
3. 直到出现真实 provider 响应的 chunk；
4. 如果首 chunk 是错误，抛给模型池；
5. 模型池切换下一个模型后，再把首 chunk 和剩余流合并返回。

这保证“额度耗尽 → 降级”发生在流已经返回客户端之前，避免客户端收到空回复。

流已经开始返回后，如果后续 tool step 或下一次模型调用失败，不能无条件重放整轮，因为工具可能已经产生数据库副作用。实现选择通过 `onError` 调用 `noteModelFailure()`，至少记录模型冷却/耗尽状态，将影响留给下一次请求处理。

### 4.4 维度契约

`EMBEDDING_DIMENSION = 1024` 同时约束：

- embedding API 请求显式传 `dimensions: 1024`；
- 返回结果写库前检查实际向量长度；
- Prisma schema 使用 `Unsupported("vector(1024)")`；
- PostgreSQL 必须安装并启用 pgvector；
- `toPgVector()` 负责把数组序列化为 PostgreSQL `::vector` 可解析的字面量。

任何一环不满足，都会导致写入失败或检索不可用。此前的 `type "vector" does not exist` 就是数据库扩展未启用，而不是模型代码问题。

## 五、工具调用：让模型能够操作产品

实现入口：`src/app/api/ai/route.ts`、`src/lib/tools/`。

### 5.1 工具调用闭环

```text
第 1 步：模型理解用户意图
  → 选择工具 + 生成参数
  → AI SDK 执行 tool.execute()
  → 工具读写数据库并返回结构化结果
第 2 步：模型读取 tool-result
  → 继续调用其他工具，或生成自然语言结果
  → 最多 10 步
```

`stopWhen: isStepCount(10)` 是循环保险丝，不是业务编排器。它避免模型在多个工具之间无限跑，但如果第 10 步仍然是 `tool-calls`，代码会记录告警，提示回答可能被截断。

### 5.2 工具工厂与用户隔离

工具通过工厂函数创建，把 `userId` 闭包注入：

| 工厂 | 主要能力 |
|---|---|
| `createStudyTools(userId, options)` | 创建计划、读取计划/打卡/统计/记忆，计划拆分相关能力 |
| `createRAGTool(userId)` | `searchKnowledgeBase`、`listKnowledgeDocs` |
| `createContextTools(userId, conversationId)` | `searchConversationHistory`、`updateAgentState` |
| `createAgentTools(userId)` | Agent Workflow：任务拆分、获取任务、更新任务状态等 |
| `createTodoTools(userId)` | 今日待办的增删改查 |
| `createStudioTools(userId, target)` | 只修改当前计划或当前文档 |

工具执行函数内部还要按 `userId` 查询和写入；闭包只是第一层边界，数据库条件和关系约束是第二层边界。

### 5.3 场景白名单：fail-closed

`src/lib/tools/router.ts` 维护 `SURFACE_POLICY`：

- `agent`：全量能力，包括跨对象写入工具；
- `plan`：读工具 + 当前计划的 `updatePlanInfo`/`updateDocument`；
- `doc`：读工具 + 当前文档的 `updateDocument`。

工作台不拿到 `deleteTodo`、`updateTaskStatus`、`breakdownPlanTasks` 等与当前编辑对象无关的写工具。工具白名单是 fail-closed：新建一个工具后不会自动暴露给任何场景，必须显式加入策略。

提示词分组也和工具白名单放在同一处维护。原因是只裁工具、不裁 prompt，会让模型反复调用一个当前场景根本不存在的工具。

### 5.4 工具结果如何进入前端和数据库

AI SDK 的完整 part 流包含：

- `text`；
- `tool-call`；
- `tool-result`；
- 工具执行状态和错误。

智能体页使用 UI Message Stream，前端可以展示工具调用过程；数据库同时保存：

- `content`：兼容标题生成和旧逻辑按纯文本读取；
- `parts`：面向界面展示的压缩摘要；
- `rawParts`：保留后续重建 tool-call/tool-result 所需的原始结构，但会做大小裁剪。

文本协议的 studio 仍把全量 part 流投影为纯文本，以保持历史兼容。

## 六、RAG 检索

实现目录：`src/lib/rag/`；工具入口：`src/lib/tools/rag-tools.ts`。

### 6.1 RAG 不是每轮自动执行

当前主对话链路不会对每一条消息主动生成 query embedding。模型只有在判断用户问题确实需要知识库内容时，才调用 `searchKnowledgeBase`。

这样做的直接收益：

- 寒暄和普通聊天不白烧 embedding 额度；
- 不把无关文档片段强行塞进每轮上下文；
- 检索行为可以被 tool-call 和日志观察。

`listKnowledgeDocs` 与 `searchKnowledgeBase` 刻意分开：

- “我的知识库有哪些文档”是完整列举问题；
- “某个概念在哪篇资料里”是语义检索问题。

不能用前者的语义检索去猜后者的完整清单。

### 6.2 文档写入链路

```text
上传 md/pdf/docx/txt
  → 文件边界与类型校验
  → PDF/Word 由 Python 脚本提取文本
  → Markdown/文本安全分块
  → embeddingWithFallback()
  → 写入 documentchunk.embedding::vector
  → knowledgedoc 保存原文及来源信息
```

相关实现：

- `src/lib/knowledge-upload-guard.ts`：上传边界；
- `src/lib/knowledge-upload.ts`：上传、解析和分块编排；
- `src/lib/rag/chunk.ts`：分块；
- `src/lib/rag/client.ts`：调用 embedding；
- `src/lib/rag/retriever.ts`：pgvector SQL 召回；
- `src/app/api/knowledge/documents/route.ts`：HTTP 上传入口。

### 6.3 查询链路

```text
searchKnowledge(query, userId)
  1. embedText(query)
  2. searchSimilarChunks(queryEmbedding, recallSize, sourceFilter, userId)
  3. PostgreSQL pgvector 计算相似度，按 userId 隔离
  4. 召回最多 20 条候选（默认 topK=5）
  5. rerankChunks(query, candidates)
  6. 返回 content/sourceName/relevanceScore
```

结果对象区分：

- `recallScore`：向量余弦相似度；
- `score`：最终分数，可能是 rerank 分数；
- `scoreSource`：`rerank` 或 `vector`；
- `searchedChunks`：该用户知识库总 chunk 数；
- `recalledChunks`：本次实际送入后续处理的候选数；
- `rerankModel`：实际使用的重排模型。

这避免了把两个不同量纲的分数混在同一字段里，导致调用方无法判断重排是否真正执行。

### 6.4 重排失败如何降级

`src/lib/rag/rerank.ts` 使用真正的 cross-encoder，而不是“再次 embedding 后重新算同一个余弦”。

- 召回成功、重排成功：按 cross-encoder 分数排序；
- 额度、限流、超时或网络抖动：本次退化为向量序，下次继续尝试；
- 参数/协议/响应结构等稳定错误：当前进程内关闭重排，退化为向量序，避免每次搜索重复付出失败请求；
- 不同 provider 的 endpoint 和请求 envelope 由 `model-pool.ts` 固化并通过纯函数测试。

RAG 的设计原则是：重排是质量增强，不应成为知识库不可用的单点故障。

## 七、长期记忆 v2

实现目录：`src/lib/memory/`。

### 7.1 数据模型

| 表 | 作用 |
|---|---|
| `memory` | 当前长期记忆实体；包含 kind、content、importance、confidence、有效期、状态和 embedding |
| `memoryevent` | 记忆变更事件日志；记录 add/update/supersede/expire/archive/delete 等动作 |
| `memorycandidate` | 从对话抽取出的候选任务；保存原始 LLM 输出、校验后的 operations、状态和重试信息 |
| `usermemory` | v1 旧表，作为迁移/回滚资产保留，v2 是当前主路径 |

`memory` 的 `status` 区分 `active`、`superseded`、`expired`、`archived`、`deleted`，因此“删除”不等同于立即丢失历史。

### 7.2 抽取链路

每轮 assistant message 成功落库后，`saveAssistantMessage()` 调用 `scheduleExtraction()`：

```text
assistant message 落库
  → decideExtraction() 门控
  → messageId 唯一约束创建 memorycandidate
  → fire-and-forget processCandidate()
  → LOW 模型抽取 JSON
  → extractJsonText() 解析
  → validateRawOperations() 校验操作
  → commitCandidate() 事务提交 memory + memoryevent + candidate 状态
```

### 7.3 门控、幂等和异步化

- **门控**：`src/lib/memory/gate.ts` 判断这条消息是否值得抽取，避免每轮闲聊都调用 LLM。
- **异步化**：抽取不阻塞用户流式响应；失败写入 candidate 状态，后续队列可补偿。
- **幂等**：`memorycandidate.messageId` 唯一，同一条 assistant message 不重复抽取。
- **抢占**：`pending/failed` 通过原子更新抢占为 `running`；过期的 `running` 可被队列重新放回 pending。
- **事务**：`commitCandidate()` 把记忆变更、事件写入和 candidate=done 放在同一事务中。
- **可审计**：记忆不是无痕地覆盖，事件表保留 before/after、actor、evidence 和来源消息。

### 7.4 记忆召回

`retrieveMemories()` 是两路召回：

1. **稳定画像**：按 `goal`、`preference` 等类型取重要且近期确认的记忆；
2. **查询相关**：对当前用户消息生成 embedding，在 active memory 中做 pgvector 召回。

随后使用 `selectByQuotas()` 按任务类型配额选择，避免某一类记忆占满上下文。短消息如“你好”“谢谢”“继续”不会触发 query embedding，降低无意义成本。

召回失败不会阻塞主回答：结果中记录 `embeddingFailed`，仍可以把稳定画像或空结果交给上下文构建器。

### 7.5 生命周期

`runMemoryMaintenance()` 每日维护：

1. 继续处理候选队列；
2. 给缺少 embedding 的 active memory 补向量；
3. 将过期目标转为 `expired`；
4. 将长期未确认的低价值记忆转为 `archived`；
5. 超过保留期的 deleted 记忆物理清理，但事件日志保留。

## 八、会话短期记忆和上下文工程

实现目录：`src/lib/context/`；核心入口：`src/lib/context/builder.ts`。

### 8.1 上下文的四层组成

`buildConversationContext()` 并行准备四类信息：

- `AgentState`：当前会话摘要、workingMemory、todoList；
- `Scene Context`：当前用户、计划、文档、学习状态等业务现场；
- `Memory Context`：长期记忆 v2 的 `<memory_context>`；
- 最近原始消息：从 `conversationmessage` 加载，保留工具 parts。

最终发送给模型的结构近似为：

```text
system：通用行为规则 + 用户名 + 当前场景 + memory_context
messages：较早 conversation_summary（system）
           + 最近原始对话（user/assistant/tool parts）
           + 当前用户消息
           + Agent Status Bar（system）
```

### 8.2 Rolling Summary

`maybeCompressConversation()` 的原则：

- 最近窗口内的消息保留原文，不参与压缩；
- 更早消息和旧摘要一起生成新的完整摘要；
- 原始 `ConversationMessage` 永不删除；
- tool-call/tool-result 未完成时，不跨越该消息设置压缩边界；
- 摘要失败时返回旧 state，不阻塞主对话；
- 保存摘要和 `compressedThroughMessageId` 使用 `AgentState.version` 乐观锁。

摘要本身使用 LOW 模型链，并通过 `completionsWithFallback()` 记为 `agent-bg`，不会被计入用户交互式精力额度。

### 8.3 AgentState 并发控制

`updateAgentState()` 通过 `updateMany where id + version` 实现乐观锁：

- 更新成功：version+1；
- 更新数量为 0：说明其他请求先写，返回冲突；
- `updateAgentStateWithRetry()` 重读一次再重试；
- 避免无条件 update 覆盖并发请求的最新 workingMemory/todoList。

### 8.4 历史消息兼容

历史 assistant 消息可能只有 `content`，也可能有 `rawParts`。`toModelMessages()` 优先恢复完整 parts；如果 parts 损坏，则退化为纯文本历史，不让单条坏消息拖垮整轮会话。

## 九、token 记账与额度控制

实现文件：`src/lib/usage.ts`、`src/lib/model-pool.ts`；数据表：`tokenusage`。

### 9.1 记账位置

模型池是统一记账边界：

- 非流式：`completionsWithFallback()` 在成功模型返回后读取 response usage；
- 流式：`streamTextWithFallback()` 包装 `onEnd`，读取 `event.usage`；
- embedding/rerank：属于系统能力，当前不写入用户交互 token 精力条；
- 记录字段包含 `userId`、`surface`、`tier`、`model`、输入/输出/总 token。

这样可以保证降级之后只记录最终成功模型，不会把一次失败尝试重复算成用户消耗。

### 9.2 额度边界

`COUNTED_SURFACES = ["agent", "studio", "chatroom"]`：

- 用户主动发起的智能体、工作台和聊天室请求计入每日额度；
- 标题、记忆抽取、计划拆分、摘要、后台 Agent 等系统开销仍然记账，但不计入用户精力条；
- `AI_TOKEN_LIMIT=0` 表示不限量；
- `user.vip=true` 绕过每日限制，但仍然保留用量记录。

交互式请求会在模型调用前执行 `assertInteractiveUsageAllowed()`，超限时抛出 `UsageLimitError`，返回“没精力了，宕机了”，不会先创建无意义的会话或继续烧模型费用。

### 9.3 记录失败不阻断主流程

`recordUsage()` 自身捕获数据库写入异常。用量观测失败时，主回答仍尽量完成。这是可用性优先的选择，但也意味着需要后续增加告警，否则记账故障可能静默发生。

## 十、从实现看出的关键设计原则

### 10.1 AI 是有副作用的应用服务

模型不仅返回文本，还能：

- 创建/修改计划、任务和待办；
- 查询学习数据和知识库；
- 更新会话 AgentState；
- 生成长期记忆候选并写入数据库；
- 消耗 token 额度。

因此项目把工具白名单、用户隔离、幂等、事务、事件日志和额度控制放在 AI 链路中，而不是把模型当成纯函数。

### 10.2 质量增强能力必须可降级

- 模型主链失败 → 换模型；
- embedding 失败 → 当前能力返回失败边界，不污染数据库；
- rerank 失败 → 回到向量序；
- 长期记忆抽取失败 → 不影响本轮回答，候选可重试；
- 摘要失败 → 使用旧摘要和最近原文；
- 历史 parts 损坏 → 退化纯文本；
- token 记账失败 → 不阻断用户响应，但应记录告警。

这使 AI 增强能力不会轻易变成主业务单点故障。

### 10.3 数据结构同时服务于运行和审计

- `content` 服务于兼容读取；`parts/rawParts` 服务于 UI 和 tool replay；
- `memory` 服务于当前召回；`memoryevent` 服务于可审计变更；
- `tokenusage` 服务于成本观测；`surface/tier/model` 服务于归因；
- `score/recallScore/scoreSource` 服务于 RAG 结果解释。

## 十一、验证与测试证据

### 已有测试

- 模型池：`tests/model-pool-chain.test.ts`、`tests/model-pool-peel.test.ts`；
- RAG：`tests/rag-chunk.test.ts`、`tests/rag-search.test.ts`、`tests/rag-rerank.test.ts`、`tests/rag-rerank-timeout.test.ts`；
- 记忆：`tests/memory-*.test.ts`；
- 上下文：`tests/context-*.test.ts`；
- Agent：`tests/agent-*.test.ts`；
- 工具：`tests/tools-*.test.ts`；
- WebSocket：`tests/websocket-*.test.ts`、`tests/message-idempotency.test.ts`；
- Markdown 安全：`tests/markdown-sanitize.test.ts`。

### 当前验证边界

- 纯逻辑测试覆盖较好，能验证错误分类、排序、配额、状态机和数据转换；
- 真实 provider 的模型质量、网络降级、token usage 完整性和工具副作用仍缺少稳定的端到端环境；
- RAG 有评测脚本和离线数据，但当前资料没有给出统一的 Recall@K、NDCG、人工评分、延迟和成本基线；
- Memory v2 有验证脚本和事件模型，但长期运行后的归档、重嵌入、重试和并发冲突缺少线上演练证据；
- 当前工作区曾观察到测试依赖缺失，完整测试链不能只依赖本地已安装的 `node_modules`，应在干净 `npm ci` 环境验证。

## 十二、后续建议

| 优先级 | 建议 | 验收证据 |
|:---:|---|---|
| P0 | 把模型池、Better Auth、pgvector、Prisma schema 和数据库实际结构纳入部署前 smoke test | 新环境可自动检查 extension、关键表/列、认证接口和向量写入 |
| P0 | 为模型降级和 token 记账补 provider mock/contract test | 可验证 403、429、超时、空流、后续 step 失败和最终用量归因 |
| P1 | 建立 RAG 质量基线 | 每次模型/分块/重排变更输出 Recall@K、排序指标、延迟和 token 成本 |
| P1 | 建立 Memory v2 长期运行测试 | 覆盖重复抽取、候选卡死恢复、事件一致性、归档/过期/物理清理 |
| P1 | 将工具 schema 和场景白名单做自动一致性检查 | 新增工具但未加入 policy 或 prompt 时 CI 失败 |
| P1 | 增加 AI 链路可观测性 | 每次请求可关联 conversationId、model、tier、tool calls、latency、tokens、fallback reason |
| P2 | 统一 `ui` 与 `text` 两套协议 | studio 迁移到 UI Message Stream 后删除兼容分支，减少两套落库逻辑 |

## 十三、总结

Summer Checkin 的“AI 原生”不是单点接入模型，而是已经形成一条包含运行时编排、工具副作用、知识检索、记忆管理、上下文压缩、流式持久化和成本控制的应用链路。

当前最成熟的是“代码层的完整性”：每个能力都有明确模块、数据库边界和失败兜底；当前最需要补齐的是“运行层的可证明性”：需要用干净依赖、真实数据库、模拟 provider、线上指标和回放数据证明这条链路在多环境、长时间和故障状态下仍然成立。
