# Nova Chat Serverless (Cloudflare Worker)

本仓库承载 GraphQL API，运行在 **Cloudflare Workers** 并通过 OpenAI Chat Completions 生成回复。前端通过 GraphQL 与之通信，最终部署到 **workers.dev** 或绑定自定义域名。

## 技术选型

- **Cloudflare Workers (Service Module)**：原生运行环境，零冷启动，方便与 Pages 搭配。
- **Wrangler 3+**：本地调试与部署 CLI，内置 dev server、KV/Secrets 管理。
- **GraphQL Yoga (edge 版本)**：兼容 Fetch API，自动处理 CORS，并提供 Schema/Resolver 结构化开发体验。
- **OpenAI REST API**：使用 `fetch` 直接调用 `chat.completions` 或 `responses` 接口；可轻松替换为其他 LLM。
- **TypeScript + Miniflare 类型**：获得 Worker 平台 API 的类型提示。

## GraphQL Schema（建议）

```graphql
type Message {
  id: ID!
  role: String!
  text: String!
  createdAt: String!
}

type Query {
  health: String!
  history(limit: Int = 30): [Message!]!
}

input SendMessageInput {
  text: String!
}

type SendMessagePayload {
  user: Message!
  bot: Message!
}

type Mutation {
  sendMessage(input: SendMessageInput!): SendMessagePayload!
}
```

- `history` 可接入 KV/D1/Vector Store 存储，默认为内存模式（单 Worker 可使用 Durable Object 扩展会话）。
- `sendMessage` 中调用 OpenAI，返回最新的用户/机器人消息，用于前端缓存追加。

## 环境变量与密钥

在 `wrangler.toml` 或 `wrangler secret put` 中配置：

| 变量 | 说明 |
| --- | --- |
| `OPENAI_API_KEY` | 必填，OpenAI 访问密钥 |
| `OPENAI_MODEL` | 可选，默认 `gpt-4o-mini` |
| `SYSTEM_PROMPT` | 可选，定义机器人语气/角色 |
| `ALLOWED_ORIGINS` | 逗号分隔的前端域名，控制 CORS |

本地开发可在 `.dev.vars` 写入：

```
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
ALLOWED_ORIGINS=http://localhost:5173
```

## 开发与部署流程

```bash
npm install            # 安装依赖
wrangler dev           # 本地启动，默认 http://127.0.0.1:8787/graphql
wrangler deploy        # 部署到 workers.dev（需登录）
wrangler tail          # 查看生产日志
```

### 推荐脚本

在 `package.json` 中添加：

```json
{
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "schema": "graphql-codegen --config codegen.ts" // 如需生成类型
  }
}
```

## 部署注意事项

1. **CORS**：在 Yoga `createYoga` 初始化中设置 `origin: ALLOWED_ORIGINS ?? '*'`，确保前端可调用。
2. **超时与流式**：Workers 默认 30s 执行时间，可使用 OpenAI 流式响应（SSE）为前端提供实时反馈。
3. **多环境配置**：通过 `wrangler.toml` 的 `env.production` / `env.preview` 区块区分不同 OpenAI Key 或速率限制。
4. **GitHub 仓库**：推送后可在 Actions 中配置 CI（`npm ci && npm run deploy -- --dry-run`）或使用 Wrangler 的 GitHub 集成自动部署。

## 与前端联调

- 在前端 `.env` 中指向 `https://<worker-subdomain>.workers.dev/graphql`。
- 建议先在本地同时运行 `wrangler dev` 与 `npm run dev`，通过 `http://127.0.0.1:8787/graphql` 验证。
- 生产环境启用 HTTPS，自定义域名需在 Cloudflare Dashboard 绑定。
