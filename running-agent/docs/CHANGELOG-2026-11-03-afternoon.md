# STRIDE OS 2026-11-03 下午修改总日志

## 记录范围

- 日期：2026-11-03
- 时区：Asia/Shanghai
- 记录时间：12:00–16:14
- 项目目录：`C:\Users\16157\Desktop\r\running-agent(1)\running-agent`

本文件是当天下午修改的总日志。`docs/CHANGELOG-2026-11-03-memory-context-fix.md`、`docs/CHANGELOG-2026-11-03-training-decision-policy.md`、`docs/CHANGELOG-2026-11-03-chat-dialog-fixes.md` 和 `docs/CHANGELOG-2026-11-03-agent-review-fixes.md` 保留为专题明细。

## 最终状态

| 项目 | 结果 |
|---|---|
| 自动化测试 | `87/87` 通过 |
| Agent 固定评估 | `30/30` 通过 |
| 评估得分 | `100` |
| 服务状态 | 正常运行 |
| 服务地址 | `http://localhost:4173` |
| 数据库结构 | 已自动迁移，无数据删除 |
| 模型网络 | 正常，错误密钥返回预期的 `MODEL_AUTH_FAILED` |

## 一、模型 API 接入排查

### 问题

项目最初在受限网络中启动，外部模型请求统一返回：

```text
MODEL_NETWORK_ERROR
```

排查确认不是 Base URL 或 API Key 配置导致，而是运行进程没有外部网络权限。

### 处理

- 在联网权限下重新启动 Node 服务。
- 使用无效测试 Key 探测 DeepSeek。
- 返回从 `MODEL_NETWORK_ERROR` 变为预期的 `401 / MODEL_AUTH_FAILED`，确认网络链路恢复。

### 代码影响

本次只涉及运行进程重启，没有修改模型配置、API Key 或模型协议代码。

## 二、长期记忆读取修复

### 问题

`buildAgentContext()` 原来只使用 `searchMemories(question, 5)`：

- 只注入与当前问题关键词匹配的记忆。
- “给我接下来七天课表”无法命中“周一我跑不了”。
- `planMemories` 虽然保存了全部记忆，但没有进入外部模型的系统上下文。

### 修复

- 相关记忆优先，同时合并全部长期记忆。
- 增加上下文记忆预算：最多 40 条、总计 4000 字符。
- 系统提示明确要求模型主动遵守长期记忆。
- 增加无关键词重合测试，以及外部模型实际收到记忆的测试。

### 关键文件

- `src/context-engine.js`
- `test/agent.test.js`

## 三、训练决策策略调整

### 新决策边界

训练计划只依据：

- 最近周跑量。
- 距离比赛天数。
- 可训练日和不可训练日。
- 伤病限制。
- 用户明确偏好。

心率不再用于：

- 模板选择。
- 强度课次数。
- 取消强度课。
- 跑量增减。
- 恢复周判定。

### 周跑量分档

| 最近周跑量 | 模板 |
|---|---|
| 低于 15 km | 基础速度建立周 |
| 15–50 km | 稳态进阶周 |
| 达到 50 km | 比赛专项进阶周 |

### 比赛日期规则

- 距离比赛超过 30 天：每周至少两次速度或节奏刺激。
- 距离比赛 30 天以内：减少短间歇和阈值堆叠，增加比赛配速跑与专项长距离。
- 比赛前 14 天内进一步缩短比赛配速课和长距离。
- 比赛配速支持全马、半马、10K 和 5K 目标。

### 心率档案

- 默认读取最近 30 个训练日。
- 同一天多个活动先合并，按时长加权。
- 档案包含样本天数、原始活动数、平均值、中位数、常见区间和近期偏移。
- 档案固定标记为 `advisory-only`。
- 高中位数只提供课中建议，不用于提前删除强度。

### 当前实际数据结果

```text
最近周跑量：46.5 km
距离珠海马拉松：50 天
计划模板：稳态进阶周
强度课：11/04 短间歇、11/06 节奏跑
目标比赛配速：4:59/km
不可训练日：周一、周日
```

## 四、新增备赛周期 Skill

### 新增 Skill

新增 `race-block-coach`，以大迫杰和 Yomif Kejelcha 的公开训练原则为基础，提供面向大众跑者的比赛日倒计时结构。

内容覆盖：

- 赛前约 8–5 周专项导入。
- 赛前约 4–3 周专项高峰。
- 赛前约 2 周减量 Taper。
- 赛周安排。
- 大众跑者换算、安全边界和公开证据来源。

### 接入

- `src/skill-loader.js` 改为 Skill 清单式加载。
- 保留 `loadEliteCoachSkill()` 向后兼容。
- `buildSystemPrompt()` 同时加载 `elite-marathon-coach` 和 `race-block-coach`。
- `src/knowledge-rag.js` 将新 Skill 纳入本地训练知识检索。

## 五、对话与计划弹窗修复

### 模型行为轨迹

- 成功完成后 900 ms 自动隐藏。
- 新增“隐藏”按钮。
- 失败和取消时保留轨迹，方便错误定位。
- 切换、新建、清空会话时隐藏旧轨迹。
- 成功运行不再显示“重试”按钮。

### 计划确认

- 计划预览生成后，在 Agent 完成时直接打开确认弹窗。
- 写入成功后立即关闭确认弹窗。
- 清理确认勾选、写入按钮和待处理计划状态。
- 轨迹中的确认按钮变为“已写入训练日历”。
- 冲突数量直接使用服务端 `preview.conflicts`。

## 六、Agent 运行与模型约束修复

### 外部模型不能绕过计划工具

- 计划类问题结束后检查是否调用 `previewPlanWrite`。
- 未调用时，服务端自动执行确定性计划预览。
- 最终回答以服务端计划结果为准。
- 增加模拟外部模型跳过工具链的回归测试。

### 运行中会话隔离

- Agent 运行期间禁止切换、新建、删除会话或清空对话。
- 避免旧运行的流式回答追加到新会话。

### 记忆抽取超时

- 用户回答完成后立即清理 Run 超时并保存完成状态。
- 记忆抽取在回答完成之后执行。
- 抽取失败只记录审计步骤，不回滚回答。

### 记忆重要性

- 修复 `importance: 0` 被错误转成 `0.5`。
- 缺失或非法值才使用默认重要性。

### Run 重试

- 只有 `failed` 或 `cancelled` 状态允许重试。
- 已完成和运行中的 Run 返回 `RUN_NOT_RETRYABLE`。

### SSE 续传

- 每条 SSE 事件增加递增 ID。
- 服务端读取 `Last-Event-ID`，只重放缺失事件。
- 前端按事件 ID 去重。
- 连接永久中断时轮询 Run 详情恢复结果。
- SSE 实测：完整事件 ID `1–36`，请求 `Last-Event-ID: 35` 时只返回 `36`。

### 上下文预算

- 超过 Token 预算时，依次裁剪低优先级记忆、知识、日历、近期跑步和历史周期数据。
- 返回 `budgetSatisfied` 标记最终上下文是否满足预算。

### Readiness 代理负荷

- COROS 缺少 `trainingLoad` 时，使用 75% 跑量变化 + 25% 训练频率变化计算代理负荷。
- readiness 返回 `loadSource`，区分真实训练负荷与历史代理负荷。
- 当前实际数据已从“节奏平稳”修正为“负荷偏高”，比例约 `4.73`。

## 七、属性级长期记忆覆盖

### 原问题

旧实现只要两条记忆都包含“周一”，就视为冲突。例如：

```text
周一喜欢冲刺跑
周一喜欢户外跑
```

第二条会错误替代第一条。

### 新结构

`memory_items` 新增：

- `facets_json`
- `superseded_at`
- `superseded_by`

每个 facet 使用：

```json
{
  "scope": "weekday:1",
  "attribute": "session-type",
  "value": "冲刺"
}
```

只有 `scope + attribute` 相同才冲突：

- `weekday:1/availability`
- `weekday:1/session-type`
- `weekday:1/surface`
- `weekday:1/long-distance-day`
- `global/goal-race`

因此：

- “周一冲刺”和“周一户外”同时保留。
- “周一户外”和“周一室内”互相替代。
- “周一冲刺”和“周一节奏跑”互相替代。
- “周一不可跑”和“周一可以跑”互相替代。
- “周一可以跑”和“周一喜欢跑长一些”同时保留。

### 旧误覆盖修复

启动时会重新计算全部记忆 facet，并修复旧版误覆盖：

- 若旧记忆与其替代者没有相同 `scope + attribute`，自动恢复旧记忆。
- 若确实存在相同属性冲突，继续保持替代状态。

实际数据库修复结果：

```text
恢复：周一我可以跑了
保留：周一喜欢跑长一些
保留：我喜欢最后冲刺一下
保留：周日我要去越野跑不了
仍被替代：周一我跑不了 → 周一我可以跑了
```

## 八、数据库迁移

自动迁移，无数据删除：

- `memory_items.facets_json`
- `memory_items.superseded_at`
- `memory_items.superseded_by`

迁移后会：

1. 重新计算全部旧记忆的属性 facet。
2. 修复旧规则下不成立的替代关系。
3. 只把状态为 active 且合法冲突的记忆标记为已替代。
4. 从 FTS 中移除已替代记忆，避免继续参与检索。

## 九、测试与评估

### 自动化测试

```text
tests 87
pass 87
fail 0
```

覆盖新增内容：

- 长期记忆无关键词注入。
- 上下文预算裁剪。
- 属性级记忆覆盖。
- 旧误覆盖修复。
- 零重要性。
- 非法记忆类型 API。
- 已完成 Run 禁止重试。
- 外部模型强制计划预览。
- SSE Last-Event-ID。
- 心率档案同日合并和时长加权。
- readiness 代理负荷。
- 10K 目标配速。
- 计划意图边界。

### Agent 固定评估

```text
total: 30
passed: 30
failed: 0
score: 100
modelMode: local
```

### 语法与运行检查

- `node --check` 通过。
- 页面和 `/api/health` 返回 `200`。
- DeepSeek 无效 Key 探测返回 `MODEL_AUTH_FAILED`，说明网络路径正常。

## 十、修改文件汇总

### 后端

- `src/agent-api.js`
- `src/agent-runtime.js`
- `src/agent-store.js`
- `src/agent-tools.js`
- `src/coach.js`
- `src/context-engine.js`
- `src/knowledge-rag.js`
- `src/local-planner.js`
- `src/memory-extraction.js`
- `src/metrics.js`
- `src/plan-constraints.js`
- `src/skill-loader.js`
- `src/training-plan.js`

### 前端

- `public/index.html`
- `public/app.js`
- `public/styles.css`

### Skill

- `skills/elite-marathon-coach/SKILL.md`
- `skills/elite-marathon-coach/references/planning-rules.md`
- `skills/race-block-coach/SKILL.md`
- `skills/race-block-coach/references/race-periodization.md`
- `skills/race-block-coach/references/evidence.md`

### 测试与评估

- `test/agent-api.test.js`
- `test/agent.test.js`
- `test/metrics.test.js`
- `test/training-plan.test.js`
- `scripts/evaluate-agent.mjs`
- `data/agent-eval-cases.json`

### 文档

- `README.md`
- `docs/AGENT_FRAMEWORK_AUDIT.md`
- `docs/CHANGELOG-2026-11-03-memory-context-fix.md`
- `docs/CHANGELOG-2026-11-03-training-decision-policy.md`
- `docs/CHANGELOG-2026-11-03-chat-dialog-fixes.md`
- `docs/CHANGELOG-2026-11-03-agent-review-fixes.md`
- `docs/CHANGELOG-2026-11-03-afternoon.md`

## 十一、运行说明

- 当前服务地址：`http://localhost:4173`
- 浏览器需要刷新以加载最新前端脚本。
- 新数据库字段已经自动迁移。
- 已替代记忆保留审计字段，不进入模型上下文。
- 训练日历中的旧计划不会自动覆盖，需要重新生成并确认。
