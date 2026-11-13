# Personal Eigent个人助手部署指南

本项目由 FastAPI 同时提供 API 和静态页面，需要 Python 或容器运行环境，不能仅通过 GitHub Pages 运行完整服务。

## 本地 Docker

按 [快速开始](../README.md#快速开始) 创建并填写 `.env`，至少设置 `AUTH_SECRET`；需要真实模型回复时再配置模型 Key。

```bash
docker build -t personal-eigent .
docker run --rm -p 127.0.0.1:8787:8787 --env-file .env \
  -e DATA_DIR=/app/data -v personal-eigent-data:/app/data personal-eigent
```

打开 [http://127.0.0.1:8787/](http://127.0.0.1:8787/)。命名卷 `personal-eigent-data` 保存数据库等运行数据；容器删除后卷仍保留。

## Render Blueprint

登录 Render 并授权访问仓库后，打开 [部署 Personal Eigent个人助手](https://render.com/deploy?repo=https://github.com/leo-ai-pm/personal-eigent)，或在 Dashboard 选择 **New → Blueprint**，再选择 `leo-ai-pm/personal-eigent`。

Render 会读取仓库根目录的 [`render.yaml`](../render.yaml)，创建 Docker Web Service。当前模板使用免费实例，`autoDeploy: false` 表示推送代码不会自动重新部署。配置格式见 [Render Blueprint 文档](https://render.com/docs/blueprint-spec)。

### 环境变量

在服务的 **Environment** 中配置；模板标记为 `sync: false` 的值需要自行填写。

| 变量 | 说明 |
|------|------|
| `AUTH_SECRET` | 会话签名密钥，必填；Blueprint 自动生成，后续应保持稳定 |
| `HARNESS_ID_HMAC_KEY` | 轨迹中的用户假名密钥；Blueprint 自动生成 |
| `KIMI_API_KEY` | Kimi 模型 Key；不填且无其他有效模型 Key 时使用离线占位回复 |
| `LLM_FAST_*` / `LLM_STRONG_*` | 两档模型的接口地址、名称和 Key；使用其他服务商时按 [模型网关](llm-gateway.md) 配置 |
| `SECRETS_FERNET_KEY` | 安全输入区加密密钥；未配置时该能力不可用 |
| `HARNESS_ADMIN_TOKENS` | 可选管理令牌，格式 `name:token[:scope]`，令牌至少 16 位；未配置则管理入口关闭 |
| `PUBLIC_BASE_URL` | 启用渠道后使用的完整 HTTPS 服务地址，用于绑定与安全输入链接 |
| `FEISHU_ENABLED`、`FEISHU_APP_ID`、`FEISHU_APP_SECRET` | 可选飞书接入，默认关闭 |
| `OPENCLAW_WECHAT_ENABLED`、`OPENCLAW_*` | 可选微信桥接，默认关闭 |
| `PROACTIVE_ENABLED`、`PROACTIVE_TZ` | 可选调度开关和时区；需要提醒时设置 `PROACTIVE_ENABLED=1` |

模板预置两档 Kimi 模型和 `DATA_DIR=/app/data`。部署前应确认模型名称和 API Key 在你的服务商账户中可用。模板中的 `CORS_ALLOW_ORIGINS=*` 可按实际前端域名调整；本项目同域访问无需增加跨域前端。

生成 Fernet 密钥：

```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

不要把真实密钥提交到 Git。保持 `AUTH_SECRET`、`HARNESS_ID_HMAC_KEY` 和 `SECRETS_FERNET_KEY` 稳定，换密钥前应了解会话失效、轨迹关联和旧数据解密的影响。

### 数据持久化

Render 免费 Web Service 使用临时文件系统；重新部署、重启或休眠会丢失本地文件，包括 SQLite 数据。免费服务也不支持持久盘，适合演示和测试。[Render 免费服务限制](https://render.com/docs/free)

需要保留真实数据时，使用支持持久盘的实例，将 Disk 挂载到 `/app/data`，与 `DATA_DIR` 一致，并安排备份。`render.yaml` 含磁盘配置示例。

切换 PostgreSQL 时，按 [PostgreSQL / RLS](postgres-rls.md) 配置数据库角色、迁移和两个连接地址。审计与 LangGraph checkpoint 仍保存在本地 SQLite，仍需保留本地数据目录。

### 部署后检查

1. 打开 Render 分配的 HTTPS 地址，确认对话页可加载。
2. 访问 `/health`，检查模型处于 `live` 还是 `mock`，以及渠道状态。
3. 发起普通对话、保存一条测试记忆、创建并确认草稿，检查基本流程。
4. 如启用调度、飞书或微信，使用自己的测试渠道验证相应功能。

网页身份由浏览器会话 Cookie 决定。跨浏览器不会自动同步账号；清理 Cookie 或更换 `AUTH_SECRET` 后，原会话可能无法继续访问原档案。

## GitHub Actions 部署

首次 Blueprint 部署可以直接在 Render 完成，无需配置 GitHub Actions 密钥。

后续使用仓库的 [Deploy Render 工作流](../.github/workflows/deploy-render.yml) 时，在 GitHub 仓库 Secrets 中同时配置：

- `RENDER_API_KEY`
- `RENDER_SERVICE_ID`

在 **Actions → Deploy Render → Run workflow** 手动触发。工作流先对同一提交执行 Harness Golden 门禁，通过后才请求 Render 部署该提交；缺少任一部署 Secret 时，部署步骤跳过。

## 相关文件

- [`Dockerfile`](../Dockerfile)：容器镜像。
- [`render.yaml`](../render.yaml)：Render Blueprint。
- [`.dockerignore`](../.dockerignore)：构建上下文排除规则。
- [`.env.example`](../.env.example)：配置模板。
