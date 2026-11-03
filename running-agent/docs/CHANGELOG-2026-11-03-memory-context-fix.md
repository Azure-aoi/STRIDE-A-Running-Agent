# STRIDE OS 长期记忆上下文修复日志（2026-11-03）

## 概要

| 项目 | 内容 |
|---|---|
| 问题 | 用户保存的长期记忆未稳定进入外部模型的系统上下文 |
| 影响 | 与当前问题没有关键词重合的记忆可能完全不被模型读取 |
| 修复状态 | 已完成 |
| 数据库结构 | 未修改 |
| 数据迁移 | 不需要 |
| 测试结果 | `npm test` 共 67 项通过，0 失败 |

## 问题现象

用户在“跑者档案与记忆”中保存了：

```text
类型：profile
内容：周一我跑不了
来源：manual
```

当用户询问“给我接下来七天课表”时，外部模型没有读取这条长期记忆。

## 根因

原实现位于 `src/context-engine.js`：

```js
const memories = searchMemories(question, 5);
```

该检索只返回与当前问题关键词或中文二元组匹配的记忆。问题“给我接下来七天课表”和记忆“周一我跑不了”没有共同关键词，因此检索结果为空。

虽然代码已经通过 `listMemories(100)` 读取了全部记忆并保存到 `planMemories`，但 `planMemories` 只用于本地计划生成和本地保底分析，没有写入发送给外部模型的 `systemPayload`。

这导致：

- 外部模型只能看到关键词命中的少量记忆。
- 本地分析和计划约束可以看到全部记忆。
- 同一个功能在本地路径和外部模型路径上的行为不一致。

## 修改内容

### 1. 新增上下文记忆合并和预算控制

文件：`src/context-engine.js`

新增以下限制：

```js
const MAX_CONTEXT_MEMORIES = 40;
const MAX_CONTEXT_MEMORY_CHARS = 4000;
```

新增 `selectContextMemories(relevant, all)`：

1. 先放入与当前问题相关的记忆。
2. 再合并数据库中保存的全部长期记忆。
3. 按记忆 ID 去重。
4. 最多保留 40 条、总计 4000 字符。

该设计保留了相关记忆的优先顺序，同时保证无关键词重合的长期记忆仍然进入模型上下文。

### 2. 调整上下文构建逻辑

修改前：

```js
const memories = searchMemories(question, 5);
```

修改后：

```js
const planMemories = listMemories(100);
const relevantMemories = searchMemories(question, 5);
const memories = selectContextMemories(relevantMemories, planMemories);
```

最终使用的 `memories` 会同时进入：

- `selectedContext`，用于运行审计和上下文查看。
- `systemPayload`，用于外部模型的系统提示。

### 3. 增加长期记忆强制遵守提示

在系统提示中新增：

```text
profile 和 memories 是用户主动保存的长期事实与训练约束。必须主动遵守，
即使当前问题没有出现相同关键词，也不得忽略其中的伤病、可用时间、
目标或个人偏好。
```

该提示与结构化上下文同时生效，减少模型即使收到记忆却不主动使用的情况。

### 4. 增加回归测试

文件：`test/agent.test.js`

新增测试一：无关键词重合也必须注入记忆。

```text
记忆：周一我跑不了
问题：给我接下来七天课表
验证：context.memories 和 context.systemPrompt 均包含该记忆
```

新增测试二：外部模型必须实际收到长期记忆。

```text
记忆：周二晚上加班，不安排强度课
验证：模拟模型收到的 system message 包含该记忆
```

后一个测试覆盖了“内部变量存在，但未真正发送给外部模型”的场景。

## 修改文件

| 文件 | 修改类型 | 说明 |
|---|---|---|
| `src/context-engine.js` | 修改 | 合并相关记忆和全部长期记忆，增加预算限制和系统提示 |
| `test/agent.test.js` | 修改 | 增加无关键词重合和外部模型注入测试 |
| `docs/CHANGELOG-2026-11-03-memory-context-fix.md` | 新增 | 本修复日志 |

## 验证结果

执行：

```powershell
npm test
```

结果：

```text
tests 67
pass 67
fail 0
```

修复后服务已重启，健康检查返回：

```text
GET http://127.0.0.1:4173/api/health
200 OK
```

实际页面地址：

```text
http://localhost:4173
```

## 行为变化

修改前：

- 只有与问题关键词命中的记忆进入外部模型上下文。
- 不相关但重要的长期约束可能被遗漏。

修改后：

- 所有已保存的长期记忆在预算内都会进入模型上下文。
- 与问题相关的记忆排在前面。
- 超过 40 条或 4000 字符时，优先保留相关和高重要性记忆。
- 跑者档案仍与长期记忆分开保存，并继续完整注入上下文。

## 兼容性与边界

- 未修改 `memory_items`、`memory_fts` 或 `runner_profile` 表结构。
- 未修改长期记忆的新增、编辑、删除和自动抽取逻辑。
- 未修改模型 API 协议和工具调用协议。
- JSON Tool Calling 和原生 Tool Calling 共用同一个系统上下文。
- 长期记忆仍受上下文预算限制，不是无限量注入。

## 相关运行处理（无代码变更）

排查模型 API 接入时发现，服务曾经运行在受限网络环境中，外部模型请求统一返回 `MODEL_NETWORK_ERROR`。

通过联网权限重新启动服务后，项目对 DeepSeek 测试接口可以收到预期的 `401` 响应，错误码变为 `MODEL_AUTH_FAILED`，确认模型网络链路已恢复。

该处理只涉及服务进程重启，没有修改源码、数据库或模型配置。

## 结论

本次修复解决的是长期记忆读取和上下文注入问题，不是记忆写入问题。现有数据库中的长期记忆无需迁移，后续新会话会在下一次请求时使用新的上下文构建逻辑。
