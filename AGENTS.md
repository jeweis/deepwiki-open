# AGENTS.md

## Docker 部署：DeepWiki + LiteLLM

推荐使用 `docker-compose-litellm.yml` 部署。该部署方式会启动三个容器：

- `deepwiki`：DeepWiki API 和前端服务
- `litellm`：OpenAI-compatible 模型网关
- `db`：LiteLLM 使用的 PostgreSQL

LiteLLM 应该作为独立容器运行，不要合并进 DeepWiki 容器。DeepWiki 只连接 LiteLLM 暴露的 OpenAI-compatible API：

```env
OPENAI_BASE_URL=http://litellm:4000/v1
OPENAI_API_KEY=<LiteLLM master key>
```

## 配置模型

真实部署配置放在本地文件：

```bash
cp docker-compose-litellm.env.example docker-compose-litellm.env
```

然后编辑 `docker-compose-litellm.env`：

```env
LITELLM_MASTER_KEY=sk-1234

OPENAI_BASE_URL=http://litellm:4000/v1
OPENAI_API_KEY=sk-1234
DEEPWIKI_EMBEDDER_TYPE=openai

DEEPWIKI_TEXT_UPSTREAM_MODEL=openai/your-chat-model-id
DEEPWIKI_TEXT_UPSTREAM_BASE_URL=https://your-text-upstream.example.com/v1
DEEPWIKI_TEXT_UPSTREAM_API_KEY=your-text-upstream-api-key

DEEPWIKI_EMBEDDING_UPSTREAM_MODEL=openai/your-embedding-model-id
DEEPWIKI_EMBEDDING_UPSTREAM_BASE_URL=https://your-embedding-upstream.example.com/v1
DEEPWIKI_EMBEDDING_UPSTREAM_API_KEY=your-embedding-upstream-api-key
```

说明：

- `OPENAI_BASE_URL` 和 `OPENAI_API_KEY` 是 DeepWiki 连接 LiteLLM 的配置。
- `DEEPWIKI_TEXT_UPSTREAM_*` 是 LiteLLM 转发文本模型请求时使用的上游配置。
- `DEEPWIKI_EMBEDDING_UPSTREAM_*` 是 LiteLLM 转发 embedding 请求时使用的上游配置。
- `OPENAI_API_KEY` 应该等于 LiteLLM 的访问 key，例如默认的 `sk-1234`，不要填上游模型服务的 key。
- 上游模型服务的真实 key 只放在 `DEEPWIKI_TEXT_UPSTREAM_API_KEY` 和 `DEEPWIKI_EMBEDDING_UPSTREAM_API_KEY`。

`docker-compose-litellm.env` 包含真实密钥，已被 `.gitignore` 忽略；不要提交。可提交的模板文件是 `docker-compose-litellm.env.example`。

## 模型别名

DeepWiki 配置中使用 LiteLLM 暴露的两个模型别名：

- 文本模型：`deepwiki-chat`
- embedding 模型：`deepwiki-embedding`

`litellm-config.yml` 负责把这两个别名映射到真实上游模型：

```yaml
model_list:
  - model_name: deepwiki-chat
    litellm_params:
      model: os.environ/DEEPWIKI_TEXT_UPSTREAM_MODEL
      api_base: os.environ/DEEPWIKI_TEXT_UPSTREAM_BASE_URL
      api_key: os.environ/DEEPWIKI_TEXT_UPSTREAM_API_KEY

  - model_name: deepwiki-embedding
    litellm_params:
      model: os.environ/DEEPWIKI_EMBEDDING_UPSTREAM_MODEL
      api_base: os.environ/DEEPWIKI_EMBEDDING_UPSTREAM_BASE_URL
      api_key: os.environ/DEEPWIKI_EMBEDDING_UPSTREAM_API_KEY
```

## 启动

从项目根目录运行：

```bash
docker compose --env-file docker-compose-litellm.env -f docker-compose-litellm.yml up -d --build
```

查看日志：

```bash
docker compose --env-file docker-compose-litellm.env -f docker-compose-litellm.yml logs -f litellm deepwiki
```

停止服务：

```bash
docker compose --env-file docker-compose-litellm.env -f docker-compose-litellm.yml down
```

## 验证

确认 LiteLLM 暴露模型：

```bash
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer sk-1234"
```

返回结果应包含：

- `deepwiki-chat`
- `deepwiki-embedding`

测试文本模型：

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepwiki-chat",
    "messages": [{"role": "user", "content": "Reply with exactly: chat-ok"}]
  }'
```

测试 embedding 模型：

```bash
curl http://localhost:4000/v1/embeddings \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepwiki-embedding",
    "input": ["embedding smoke test"]
  }'
```

确认 DeepWiki 健康状态：

```bash
curl -f http://localhost:8001/health
```

确认 DeepWiki 模型配置：

```bash
curl http://localhost:8001/models/config
```

`defaultProvider` 应为 `openai`，OpenAI provider 中应包含 `deepwiki-chat`。

## 持久化数据

Docker 部署使用 named volumes：

- `deepwiki-open_deepwiki_data`：挂载到容器内 `/root/.adalflow`，保存仓库缓存、embedding 和索引数据。
- `deepwiki-open_deepwiki_logs`：挂载到容器内 `/app/api/logs`，保存 API 日志。

如果更换 embedding 模型或 embedding 维度，已有索引可能不兼容，需要清理或重建 `deepwiki-open_deepwiki_data` 中的数据。
