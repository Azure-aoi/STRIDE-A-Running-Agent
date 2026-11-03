# STRIDE OS 改动归档（2026-11-03）

本文档归档本轮针对 Agent 运行时、长期记忆、模型接入、活动详情页和后端结构的改动，便于后续维护和回溯。

## 总览

| 优先级 | 改动 | 状态 |
|---|---|---|
| P1 | 外部模型回答后自动抽取并写入长期记忆 | 已完成 |
| P1 | Agent 运行审计默认保留 90 天并自动清理 | 已完成 |
| P2 | OpenAI Responses / Anthropic Messages / OpenAI-compatible 原生 tool calling | 已完成 |
| P2 | 单次活动详情页与路线轨迹 | 已完成 |
| P2 | 后端路由拆分与旧聊天接口清理 | 已完成 |

验证结果：

- `npm test`：66 项通过，0 失败。
- `npm run evaluate:agent`：30 条固定用例全部通过，得分 100。
- 活动详情接口、GPX 轨迹、深链接、COROS 状态、模板下载和旧接口 404 均已做 HTTP 冒烟测试。

---

## 一、自动长期记忆（P1）

### 行为

- 仅外部模型运行成功后触发；本地分析模式不自动写入记忆。
- 从用户原话中抽取未来仍然有效的个人事实，类型限定为 `goal`、`preference`、`injury`、`decision`、`profile`。
- 禁止抽取本轮训练数据、模型建议、临时情绪、猜测和医疗诊断。
- 每次最多 3 条，使用第三人称短句；自动去重后以 `source: "auto"` 写入。
- 记忆抽取失败不会导致整次运行失败；用户取消仍会中断运行。

### 实现

- 新增 `src/memory-extraction.js`
  - `parseMemoryCandidates()`：兼容代码围栏和多层包裹，限制条数、类型、内容长度和重要度范围。
  - `saveExtractedMemories()`：按去除空白和标点后的文本做完全相同/包含关系去重。
  - `extractAndSaveMemories()`：串联模型抽取和落库。
- `src/model-gateway.js`
  - 新增 `extractAgentMemories()`，使用独立记忆抽取 Prompt，`maxTokens: 400`，要求只输出 JSON 数组。
  - `providerError()` 额外保存 `providerMessage`，供原生工具降级判断使用。
- `src/agent-runtime.js`
  - 新增 `captureRunMemories()`，在外部模型回答完成后、Run 完成前执行。
  - 记录 `type: "memory"` 的 Agent 步骤，`stepNo = MAX_STEPS + 5`；有新增记忆时推送 `memory.created` SSE 事件。
- 新增 `test/memory-extraction.test.js`
  - 覆盖代码围栏解析、无效输出、3 条上限、类型回退、内容长度过滤、重要度裁剪、自动来源、精确/包含去重。

---

## 二、Agent 审计保留（P1）

### 行为

- `agent_runs` 和 `agent_steps` 默认保留 90 天。
- 服务启动时清理一次，之后每 24 小时清理一次；定时器 `unref()`，不阻碍进程退出。
- 只删除过期的 Run 和步骤，不影响会话、长期记忆、训练日历和活动数据。
- 环境变量 `AUDIT_RETENTION_DAYS` 可调整保留天数；无效值或非正数回退为 90 天。

### 实现

- `src/agent-store.js`
  - 新增 `pruneAgentAudit({ days = 90 })`：先删除过期 Run 的步骤，再删除过期 Run，返回删除数量和截止时间。
- `server.js`
  - 启动时调用 `pruneAgentAudit()`，并注册每日清理定时器。
- `test/agent.test.js`
  - 新增保留窗口测试，验证仅清理 91 天前的 Run 和步骤，新数据保留。

---

## 三、原生 tool calling（P2）

### 行为

- 外部模型优先使用各厂商原生工具调用协议：
  - OpenAI Responses API：`tools` + `function_call` / `function_call_output`。
  - Anthropic Messages API：`tools` + `tool_use` / `tool_result`。
  - OpenAI-compatible Chat Completions：`tools` + `tool_calls` / `role: "tool"`。
- 支持单轮返回多个工具调用，按顺序执行并回传结果。
- 原生请求失败且错误为 400/422、错误文本涉及 tool/function/unsupported 时，自动降级到原有 JSON 工具协议。
- 仅在尚未执行任何工具前允许降级，避免重复执行有副作用的工具。
- 原生工具调用仍受 8 步上限、60 秒模型超时、120 秒 Run 超时、取消传播和本地保底约束。

### 实现

- `src/model-gateway.js`
  - 新增 `generateNativeAgentDecision()` 和各协议请求适配。
  - 新增中性消息格式转换：`toChatMessages()`、`toAnthropicMessages()`、`toResponsesInput()`。
  - 新增 `parseToolArguments()`；非法 JSON 参数回退为 `{}`，交给工具 Schema 校验并回传错误。
  - 新增 `toolsUnsupportedError()`，统一产出 `MODEL_TOOLS_UNSUPPORTED`。
  - 抽取 `withModelRetry()`，原生决策和 JSON 文本调用共享重试与备用模型逻辑。
- `src/agent-runtime.js`
  - 原 `runExternalAgent()` 更名为 `runJsonExternalAgent()`。
  - 新增 `runNativeExternalAgent()`：原生决策、多工具执行、观察回传、步骤审计和 SSE 事件。
  - 新增 `runExternalAgent()` 包装器：原生优先，必要时安全降级。
  - 抽取 `reflectFinalAnswer()`，原生和 JSON 路径共享反思逻辑。
- 测试
  - `test/model-gateway.test.js`：三种协议的工具调用往返测试。
  - `test/agent.test.js`：原生工具调用成功路径；工具不支持时降级 JSON 协议。

---

## 四、活动详情页（P2）

### 行为

- 训练记录列表行可点击或用键盘 Enter/空格进入详情。
- 详情路由为 `/activities/:id`，支持浏览器前进后退和直接访问深链接。
- 展示距离、时长、平均配速、平均/最大心率、步频、累计爬升、训练负荷、来源、导入文件和备注。
- 有轨迹数据时渲染 SVG 路线，标记起点和终点；无轨迹时显示空状态。
- 返回按钮和侧边栏“训练记录”均可回到列表。

### 实现

- `src/db.js`
  - 抽取 `activitySelect` 和 `mapActivityRow()`，新增 `getActivity(id)`；保持 `route` 懒解析。
- `src/api/data-routes.js`
  - 新增 `GET /api/activities/:id`，不存在时返回 404 `ACTIVITY_NOT_FOUND`。
- `public/index.html`
  - 新增 `data-view="activity"` 详情视图。
- `public/app.js`
  - 新增详情路由识别、详情加载与渲染、`renderRouteMap()`、时长/时间格式化、列表点击与键盘事件。
- `public/styles.css`
  - 新增详情指标卡、路线图、详细指标列表和移动端响应式样式。

---

## 五、后端拆分与清理（P2）

### 结构变化

`server.js` 现在只负责：

- Express 中间件、JSON 限制、缓存头和 Origin 校验。
- 路由注册、后台任务恢复、审计清理和自动同步定时器。
- 静态文件、SPA 回退、统一错误处理和进程生命周期。

新增 `src/api/` 路由模块：

- `src/api/data-routes.js`：健康检查、总览、活动列表/详情、来源、导入模板、文件上传、导入删除和演示数据重置。
- `src/api/coros-routes.js`：高驰状态、OAuth 开始/回调、工具列表、同步和断开连接。
- `src/api/model-routes.js`：模型连通性测试。

### 清理内容

- 移除未被前端使用的旧接口：
  - `GET /api/agent/messages`
  - `DELETE /api/agent/messages`
  - `POST /api/agent/chat`
- 移除对应数据层函数：`saveMessage()`、`listMessages()`、`clearMessages()`。
- 移除旧链路代码：
  - `src/model-gateway.js` 的 `generateModelAnalysis()`。
  - `src/coach.js` 的 `buildCoachContext()`、`buildUserPrompt()` 和 `titleForRun()`。
- 新增 `/api/*` 统一 404 JSON 响应，未知接口返回 `API_NOT_FOUND`。

### 兼容性说明

- 前端原本就只使用 `/api/agent/runs` 系列接口，本次移除不影响页面功能。
- 数据库中的旧 `messages` 表未删除，避免破坏已有本地数据；只是不再读写。

---

## 六、此前完成的 P0/P1 修复（一并归档）

- 工具参数严格 JSON Schema 校验：新增 `src/tool-schema.js`，非法参数返回 400 `TOOL_ARGUMENTS_INVALID`，便于模型重试。
- 取消传播：Run 取消会中断工具执行和 COROS 网络请求，统一使用 `RUN_CANCELLED`。
- COROS 数据正确性：
  - 只同步跑步类活动（100–103），忽略步行、骑行、游泳、力量等记录。
  - 修正纯日期和带时区日期的解析，统一使用东八区。
  - 跨来源去重：开始时间 ±3 分钟、距离 ±1.5%、时长 ±3%。
- 数据库性能：
  - 为记忆、Run、步骤、训练日历、任务和确认令牌增加索引。
  - 活动路线改为懒解析，避免总览批量解析轨迹 JSON。
- 评估隔离：
  - `scripts/evaluate-agent.mjs` 为每条用例创建并删除独立会话。
  - 增加 7 天计划结构检查：必须经过 `previewPlanWrite`，日期为 7 天且不重复。

---

## 七、变更文件清单

### 新增

- `src/memory-extraction.js`
- `src/api/data-routes.js`
- `src/api/coros-routes.js`
- `src/api/model-routes.js`
- `test/memory-extraction.test.js`
- `docs/CHANGELOG-2026-11-03.md`

### 修改

- `server.js`
- `src/agent-store.js`
- `src/agent-runtime.js`
- `src/model-gateway.js`
- `src/db.js`
- `src/coach.js`
- `src/agent-tools.js`
- `src/coros.js`
- `src/tool-schema.js`
- `public/index.html`
- `public/app.js`
- `public/styles.css`
- `test/agent.test.js`
- `test/model-gateway.test.js`
- `test/tool-schema.test.js`
- `test/dedupe.test.js`
- `test/coros.test.js`
- `scripts/evaluate-agent.mjs`
- `README.md`
- `docs/AGENT_FRAMEWORK_AUDIT.md`

## 八、新增配置与行为变更

| 项目 | 说明 |
|---|---|
| `AUDIT_RETENTION_DAYS` | Agent 审计保留天数，默认 90 |
| 自动长期记忆 | 仅外部模型成功后写入，本地分析不写入 |
| 原生工具调用 | 默认启用；接口不支持时自动降级到 JSON 工具协议 |
| 深度链接 | 支持 `/activities/:id` |
| 旧聊天接口 | `/api/agent/chat` 和 `/api/agent/messages` 已移除 |
