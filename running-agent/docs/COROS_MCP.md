# COROS MCP 接入说明

本项目通过 COROS 官方 MCP 接入高驰训练数据，不使用账号密码或非公开接口。

## 接入流程

```text
点击连接 COROS
  ↓
发现区域授权服务器
  ↓
动态注册 OAuth 客户端
  ↓
Authorization Code + PKCE
  ↓
高驰账号登录与授权
  ↓
回调本地 Node 服务
  ↓
加密保存 access_token / refresh_token
  ↓
MCP initialize
  ↓
querySportRecords
  ↓
getActivityDetail
  ↓
统一活动格式并写入 SQLite
```

## 服务地址

| 地区 | MCP Endpoint |
|---|---|
| 自动 | `https://mcp.coros.com/mcp` |
| 中国大陆 | `https://mcpcn.coros.com/mcp` |
| 欧洲 | `https://mcpeu.coros.com/mcp` |
| 美国及其他地区 | `https://mcpus.coros.com/mcp` |

自动模式先读取 `/.well-known/openid-configuration`，再将后续授权、令牌刷新和 MCP 调用固定到高驰返回的具体区域。

## OAuth

- 授权方式：Authorization Code + PKCE S256
- Scope：`openid offline_access mcp.tools`
- 客户端注册：`POST /connect/register`
- 授权入口：`GET /oauth2/authorize`
- 令牌交换：`POST /oauth2/token`
- Access Token：30 天
- Refresh Token：180 天，每次刷新后轮换
- Token Endpoint 认证方式：`none`

服务端会提前 2 分钟刷新访问令牌。刷新失败时清除本地连接，并要求用户重新授权。

## MCP 调用

COROS MCP 使用无状态 Streamable HTTP：

1. `POST /mcp`，方法 `initialize`，协议版本 `2025-06-18`。
2. `POST /mcp`，方法 `tools/list`，用于检查连接和工具目录。
3. `POST /mcp`，方法 `tools/call`，调用 `querySportRecords`。
4. 对前 12 条包含 `labelId` 和 `sportType` 的活动调用 `getActivityDetail`。

请求头：

```text
Authorization: Bearer <access_token>
Accept: application/json, text/event-stream
Content-Type: application/json
```

不发送 `Mcp-Session-Id`，不发送 `notifications/initialized`。

## 活动映射

| COROS 字段 | 内部字段 |
|---|---|
| `labelId` | `externalId` |
| `startTime` / `startDate` | `startedAt` |
| `duration` / `durationSec` | `durationSec` |
| `distanceKm` / `distanceMeters` | `distanceM` |
| `avgPace` | `avgPaceSecPerKm` |
| `avgHr` / `averageHeartRate` | `avgHeartRate` |
| `maxHr` | `maxHeartRate` |
| `avgCadence` | `cadence` |
| `elevGain` | `elevationGainM` |

响应优先读取 `structuredContent`，同时兼容 JSON 文本和官方工具示例中的格式化文本。

## 当前限制

- 每次同步最多请求 50 条记录。
- 最多为前 12 条记录补充详情，以控制延迟和调用量。
- 不下载 FIT 文件，因此不保存 GPS 轨迹和逐秒数据。
- 不创建、修改或删除 COROS 训练计划。
- 高驰 MCP 没有 Webhook，本应用使用手动触发同步。

## 官方资料

- <https://support.coros.com/hc/en-us/articles/53181619102996-Build-on-COROS-MCP>
- <https://github.com/coroslab/COROS-MCP>
- <https://github.com/coroslab/COROS-MCP/blob/main/docs/zh/tools.md>
