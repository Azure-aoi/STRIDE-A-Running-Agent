# Running Agent 当前 Agent 框架


## 当前大框架

```text
浏览器三页面
    ↓
Express API
    ↓
数据层：文件上传 / 运动平台 MCP / SQLite
    ↓
确定性指标：跑量、配速、心率、负荷、恢复节奏
    ↓
Context Builder：28 天汇总 + 最近 10 次活动
    ↓
本地规则 或 外部大模型 API
    ↓
回答写入 SQLite 并显示在 AI 教练页面
```

核心实现位于 `server.js`、`src/api/`、`src/agent-runtime.js`、`src/agent-tools.js`、`src/context-engine.js`、`src/model-gateway.js`、`src/coach.js`、`src/db.js`、`src/coros.js` 和 `src/training-plan.js`。`server.js` 只负责中间件、路由注册和生命周期；数据、COROS 和模型接口已拆分到 `src/api/`。当前使用有界 Agent Loop：外部模型优先使用原生 function calling，不支持时降级到 JSON 工具协议，本地模式使用规则规划器；三者共享最多 8 步、60 秒模型超时、120 秒 Run 超时、取消传播和本地保底。

## 代码文件位置

项目根目录：`running-agent/`

```text
running-agent/
├─ server.js                 Node.js/Express 组合根：中间件、路由注册和生命周期
├─ package.json              依赖和启动脚本
│
├─ public/                   前端代码
│  ├─ index.html             页面结构与弹窗
│  ├─ styles.css             页面样式与响应式布局
│  └─ app.js                 路由、交互、图表和 API 调用
│
├─ src/                      后端业务代码
│  ├─ api/
│  │  ├─ data-routes.js      训练数据、活动详情、导入和演示数据 API
│  │  ├─ coros-routes.js     高驰授权、同步和连接 API
│  │  └─ model-routes.js     模型连通性测试 API
│  ├─ db.js                  SQLite 表结构和数据操作
│  ├─ coros.js               高驰 MCP、OAuth 和数据同步
│  ├─ secret-store.js        高驰令牌加密存储
│  ├─ model-gateway.js       模型 API 调用与错误处理
│  ├─ coach.js               Prompt、上下文和本地教练分析
│  ├─ agent-api.js           Agent、会话、记忆、日历和任务 API
│  ├─ agent-runtime.js       有界 Agent Loop、SSE 和运行审计
│  ├─ agent-store.js         会话、记忆、Run、审计清理、日历和任务数据层
│  ├─ memory-extraction.js   模型长期记忆抽取、解析和去重写入
│  ├─ agent-tools.js         工具注册、执行和计划确认
│  ├─ local-planner.js       本地规则规划与动态工具选择
│  ├─ context-engine.js      上下文预算、摘要与信息选择
│  ├─ knowledge-rag.js       本地训练知识检索
│  ├─ auth.js                可选访问口令、会话和登录限流
│  ├─ origin-guard.js        修改类 API 的 Origin 校验
│  ├─ tool-schema.js         工具参数 JSON Schema 严格校验
│  ├─ job-runner.js          后台同步、重试和恢复
│  ├─ skill-loader.js        训练 Skill 加载
│  ├─ training-plan.js       训练计划生成
│  ├─ plan-constraints.js    长期记忆与档案到计划约束
│  ├─ metrics.js             训练指标计算
│  ├─ parsers.js             CSV、GPX、TCX 文件解析
│  └─ seed.js                演示数据
│
├─ test/                     自动化测试
├─ scripts/evaluate-agent.mjs 30 条 Agent 评估脚本
├─ skills/                   训练 Skill 和参考规则
├─ docs/                     项目文档
├─ data/                     SQLite 数据库、加密密钥和 Agent 评估用例
├─ samples/                  示例跑步数据
├─ screenshots/              历史界面截图
└─ node_modules/             第三方依赖
```

`data/master.key` 是令牌加密密钥，必须和 `data/running-agent.db` 一起备份并保密。`node_modules` 是安装生成的依赖目录，不需要手动修改。

## 启动
```powershell
cd running-agent
$env:PORT = "4186"
node server.js
```

服务默认只监听 `127.0.0.1`；需要局域网访问时显式设置 `$env:HOST = "0.0.0.0"`。所有修改状态的 `/api/` 请求都会经过 Origin 校验。

## 能力实现情况

| Agent 能力 | 状态 | 当前实现与边界 |
|---|---|---|
| 大模型接入 | 已实现 | 支持 OpenAI Responses、Anthropic Messages、OpenAI-compatible；可接入 DeepSeek、Kimi、千问和自定义模型。 |
| Prompt 工程 | 已实现 | 有角色、证据边界、安全限制、结构化上下文、工具协议、固定回答格式和训练 Skill；Prompt 内容保存版本哈希。 |
| 上下文管理 | 已实现基础能力 | 按预算组装训练指标、最近活动、会话、长期记忆和知识检索结果；支持摘要、截断、Token 估算和淘汰原因记录。 |
| 短期记忆 | 已实现 | 会话独立保存消息，最近对话会传入模型；上下文接近预算时压缩旧消息并保留摘要。 |
| 长期记忆 | 已实现基础能力 | 支持跑者档案、偏好、目标、伤病和历史决策；使用 SQLite FTS5 与中文关键词兜底检索。外部模型回答后自动抽取用户明确陈述的长期事实，去重后以 `auto` 来源写入；本地分析不自动写入。 |
| Agent Loop | 已实现 | 外部模型优先使用原生 function calling（OpenAI Responses、Anthropic Messages、OpenAI-compatible），接口不支持工具时自动降级到 JSON 工具协议；本地模式使用规则规划器根据已完成观察选择下一步。三者共享 8 步上限、超时、取消、重试和本地保底。 |
| 规划能力 | 部分实现 | 本地规则可生成 7 天模板；模型可依据 Skill 一次性生成计划。尚未实现 Plan-and-Solve 的多步执行和根据新结果动态重规划。 |
| 工具使用 | 已实现 | 外部模型优先通过原生 function calling 选择注册工具，并支持单轮多个工具调用；不支持原生工具时降级到结构化 JSON 协议。本地模式使用确定性工具流程，工具结果会回传并进入下一轮。 |
| MCP 协议 | 部分实现 | 已实现高驰 OAuth、工具调用和运动数据同步；尚未把 MCP 工具目录接入模型的通用工具选择循环。 |
| Reflection | 已实现基础能力 | 外部模型生成初稿后执行事实和约束检查；本地模式使用确定性规则校验训练模板。 |
| RAG | 已实现基础能力 | 本地训练 Skill 使用关键词、中文二元组、特征哈希向量和余弦相似度进行混合检索；不依赖外部向量数据库。 |
| 多 Agent 协作 | 未实现 | 当前是单体 Express 服务；没有 A2A、ANP、角色分工或子 Agent 调度。 |
| 评估体系 | 已实现基础能力 | 87 项自动化测试和 30 条 Agent 固定评估集，覆盖事实关键词、工具路径、误报无数据、回答长度、会话、长期记忆注入与属性级替代、上下文预算、个人心率档案、训练决策、外部模型计划强制预览、SSE 续传、原生工具调用与降级、工具 Schema、取消传播、审计清理和确认写入。 |
| 可观测性 | 已实现 | 保存 Run、步骤、完整上下文、Prompt 版本、模型、估算 Token、耗时、错误和工具结果；审计数据默认保留 90 天，可通过 `AUDIT_RETENTION_DAYS` 调整。 |

## 结论

当前产品已经升级为：**带确定性指标、长期记忆、知识检索、工具选择、有界循环和运行审计的跑步训练 Agent**。文件导入、高驰同步和本地分析仍可离线运行；外部模型可以自主选择注册工具并执行多步分析。

当前仍保留的边界：训练知识向量是本地特征哈希表示，不是外部 Embedding 模型；高驰 MCP 只接入活动和课程写入，不是通用 MCP 工具目录；当前仍是单 Agent，没有 A2A、ANP 或子 Agent 协作。

## 已实现的功能

### 前端

| 功能 | 页面与交互 | 依赖接口 | 验收标准 |
|---|---|---|---|
| 多会话 | AI 教练左侧增加会话列表；支持新建、切换、重命名、删除和分页 | `GET/POST/PATCH/DELETE /api/sessions`、`GET /api/sessions/:id/messages` | 刷新后仍停留当前会话；会话之间消息和上下文隔离 |
| 短期记忆展示 | 会话顶部显示消息数、摘要生成状态和当前上下文占用 | `GET /api/sessions/:id/context` | 用户可以判断模型本轮会记住哪些内容 |
| 长期记忆管理 | 设置页增加“跑者档案”和“长期记忆”两个标签；支持新增、编辑、删除、调整重要性 | `GET/PUT /api/profile`、`GET/POST/PATCH/DELETE /api/memory` | 修改后下一轮 Agent 请求立即生效 |
| Agent 执行轨迹 | 回答上方显示“理解任务、读取数据、调用工具、生成计划、检查结果”等阶段 | `GET /api/agent/runs/:id/events` | 阶段状态实时变化；失败时显示具体步骤，不显示内部思维链 |
| 工具结果卡片 | 根据工具名称渲染活动列表、指标摘要、计划草稿、同步结果和写入结果 | 事件中的 `toolName`、`inputSummary`、`outputSummary` | 每张卡片可展开查看结构化结果和错误 |
| 取消与重试 | 运行时显示取消按钮；失败后支持重试或切换模型 | `POST /api/agent/runs/:id/cancel`、`POST /api/agent/runs/:id/retry` | 取消后停止后续工具调用；重试创建新的 Run |
| 计划预览与确认 | 写入外部平台前展示开始日期、周数、课程列表和变更差异 | `POST /api/tools/preview-plan-write`、`POST /api/tools/publish-plan` | 未经二次确认不得写入；确认记录与 Run 关联 |
| 上下文调试 | 开发者抽屉展示系统提示版本、选中活动、记忆、Token 占比和截断原因 | `GET /api/agent/runs/:id/context` | 普通用户默认不可见，开启调试开关后可用 |
| 模型运行状态 | 显示模型、首字延迟、总耗时、Token、费用、重试和降级原因 | Run 状态事件 | 每次回答都能追溯到实际模型和调用结果 |
| 训练日历 | 周视图展示课程、状态和完成情况；支持打卡、跳过、RPE 和备注 | `GET /api/training/calendar`、`POST /api/training/calendar/:date/check-in` | 课程状态可持久化，并能被下一次 Agent 分析读取 |

### 后端

#### 1. 核心数据表

```text
sessions(id, user_id, title, summary, created_at, updated_at)
messages(id, session_id, role, content, model_label, token_count, run_id, created_at)
memory_items(id, user_id, type, content, importance, source, last_used_at, created_at)
agent_runs(id, session_id, status, model, context_json, usage_json, error, started_at, ended_at)
agent_steps(id, run_id, step_no, type, tool_name, input_json, output_json, status, duration_ms)
training_sessions(id, user_id, date, workout_json, status, rpe, notes, completed_at)
```

`messages` 增加 `session_id`，不能再把所有问题放在同一个全局会话中。`memory_items.type` 至少支持 `profile`、`goal`、`preference`、`injury` 和 `decision`。

#### 2. 会话记忆

每次请求按当前 `session_id` 读取最近 10 条消息。输入上下文超过预算的 70% 时，将较早消息压缩为会话摘要，保留目标、约束、结论和未解决问题，再附带最近 4 条原始消息。摘要版本和最后覆盖的消息位置写入 `sessions.summary`。

#### 3. 长期记忆检索

长期记忆先用 SQLite `FTS5` 完成关键词检索，再结合重要性、新近性和当前问题重排。每次最多注入 5 条高相关记忆。没有可靠检索结果时不强行注入，避免无关偏好污染回答。

#### 4. Agent Runtime

新增状态机：`queued → planning → acting → observing → reflecting → completed`，失败时进入 `failed`，用户取消时进入 `cancelled`。默认最多 8 步、单次模型调用 60 秒、整个 Run 120 秒；连续两次工具失败或达到最大步数时停止并返回可解释错误。

每一步执行以下流程：

```text
组装上下文 → 调用模型 → 解析工具调用 → 校验参数
→ 执行工具 → 保存 Observation → 检查是否完成
```

#### 5. 工具注册

首批工具固定为以下十项：

| 工具 | 类型 | 用途 |
|---|---|---|
| `getRecentActivities` | 只读 | 查询最近活动 |
| `getTrainingMetrics` | 只读 | 计算跑量、配速、心率、负荷和恢复节奏 |
| `getTrainingCalendar` | 只读 | 读取训练安排、完成状态和 RPE |
| `getRunnerProfile` | 只读 | 读取目标、PB、可用时间和伤病限制 |
| `searchMemories` | 只读 | 检索长期记忆 |
| `searchTrainingKnowledge` | 只读 | 检索本地训练知识 |
| `generateTrainingPlan` | 只读 | 生成计划草稿 |
| `syncActivitySource` | 写入 | 同步外部运动数据 |
| `previewPlanWrite` | 只读 | 生成外部计划写入预览 |
| `publishPlan` | 写入 | 经确认后写入训练计划 |

每个工具必须声明参数 JSON Schema、超时、只读或写入类型、结果大小限制和错误格式。模型只能调用注册表中的工具。

#### 6. 上下文引擎

输入预算建议按以下比例分配：系统提示 15%、长期记忆与跑者档案 20%、近期对话 35%、活动与指标 30%。预算可配置，并预留输出空间。每次上下文构建需要记录候选信息、实际入选信息、Token 数和淘汰原因。

#### 7. 流式 API

新增：

```text
POST /api/agent/runs
GET  /api/agent/runs/:id/events
POST /api/agent/runs/:id/cancel
POST /api/agent/runs/:id/retry
```

`events` 使用 SSE，至少包含 `run.started`、`stage.changed`、`tool.started`、`tool.completed`、`message.delta`、`run.completed` 和 `run.failed` 事件。客户端断开后服务端应停止后续模型调用和工具执行。

#### 8. 权限与审批

只读工具自动执行；写入工具必须先生成预览，再由后端签发一次性 `confirmationToken`，用户确认后才能执行。审计记录必须包含用户、会话、Run、工具参数、结果和时间。

#### 9. 模型路由与容错

允许为“快速问答”和“训练规划”分别配置模型。`401/403` 不重试；`429/5xx/超时` 最多重试两次并支持主备模型。所有降级必须在前端显示，不能静默切换到质量差异较大的模型。

#### 10. 后台任务

新增 `jobs` 表保存同步、导入和长计划任务，使用幂等键避免重复写入。任务支持排队、进度、取消、失败重试和服务重启后的恢复。

#### 11. 评估与追踪

建立不少于 30 条的固定评估集，覆盖无数据、历史数据、高负荷、伤病限制和计划调整。检查工具调用是否正确、数值是否来自数据库、计划是否违反训练间隔、同一会话是否保持约束。每次 Run 保存 Prompt 版本、模型、工具轨迹、Token、费用、耗时和错误。
