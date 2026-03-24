# Claude2API — 部署指南

Claude.ai Web → Anthropic API 兼容代理。将 Claude.ai 账号会话转换为标准 `/v1/messages` 接口，支持流式输出、工具调用、图片上传、扩展思考等功能。

---

## 快速部署

### 1. 克隆此仓库

```bash
git clone https://github.com/realnoob007/claude2api-deploy.git
cd claude2api-deploy
```

### 2. 配置环境变量

```bash
cp .env.example .env
nano .env   # 填入你的配置
```

### 3. 启动服务

```bash
docker compose up -d
```

### 4. 验证

```bash
curl http://localhost:8080/health
# {"status":"ok","accounts":1}
```

---

## 两种运行模式

### 模式一：Simple 模式（推荐新手，无需数据库）

在 `.env` 中设置：

```env
CLAUDE_SESSION_KEYS=sk-ant-sid01-xxx,sk-ant-sid01-yyy
```

多个 Session Key 用逗号分隔，服务启动后自动轮询使用，无需 PostgreSQL。

### 模式二：PostgreSQL 模式（推荐生产）

`docker-compose.yml` 已内置 PostgreSQL 和 Redis，**开箱即用，无需额外安装**。

在 `.env` 中配置（`DB_HOST` 固定填 `postgres`，对应 compose 内的服务名）：

```env
DB_HOST=postgres
DB_PORT=5432
DB_USER=claude2api
DB_PASS=your_strong_password    # 必填，自定义强密码
DB_NAME=claude2api

REDIS_PASS=your_redis_password  # 必填，需与 REDIS_URL 保持一致
REDIS_URL=redis://:your_redis_password@redis:6379/0
```

> 如需连接**外部**数据库，将 `DB_HOST` 改为远程 IP/域名，并在 `docker-compose.yml` 中删除 `postgres` 和 `redis` 服务及对应的 `depends_on`。

通过管理面板动态添加/删除账号，支持每账号独立限额、封禁自动切换等高级功能。

---

## 获取 Session Key

1. 打开 [claude.ai](https://claude.ai) 并登录
2. 打开浏览器开发者工具（F12）→ Application → Cookies
3. 找到 `sessionKey`，复制其值（格式：`sk-ant-sid01-...`）

---

## 配置说明

| 变量 | 默认值 | 说明 |
|---|---|---|
| `LISTEN_ADDR` | `:8080` | 服务监听地址，格式为 `:端口号` |
| `CLAUDE_SESSION_KEYS` | — | Simple 模式：逗号分隔的 Session Keys |
| `DB_HOST` | `localhost` | PostgreSQL 主机 IP 或域名 |
| `DB_PORT` | `5432` | PostgreSQL 端口（默认 5432，远程可能不同）|
| `DB_USER` | `claude2api` | PostgreSQL 用户名 |
| `DB_PASS` | — | PostgreSQL 密码 |
| `DB_NAME` | `claude2api` | PostgreSQL 数据库名 |
| `DB_SSLMODE` | `disable` | PostgreSQL SSL 模式：`disable`（本地/VPS）\| `require`（Neon/Supabase 等托管数据库）|
| `DATABASE_URL` | — | 完整 PostgreSQL DSN，优先级高于所有 `DB_*` 变量（推荐 Neon/Supabase 使用）|
| `PROXY_HOST` | — | 住宅代理主机（留空则直连 claude.ai）|
| `PROXY_PORT` | `3010` | 代理端口 |
| `PROXY_USER` | — | 代理认证用户名 |
| `PROXY_PASS` | — | 代理认证密码 |
| `PROXY_REGION` | `US` | 代理出口区域（如 `US`、`JP`、`GB`）|
| `ADMIN_USER` | `admin` | 管理面板用户名 |
| `ADMIN_PASS` | `admin` | 管理面板密码（**务必修改**）|
| `REDIS_URL` | — | Redis 连接串，用于持久化 metrics（可选）|
| `CLAUDE_DAILY_LIMIT` | `0` | 每账号每日请求上限（`0` = 不限）|
| `MAX_RETRIES` | `3` | 账号切换重试次数 |
| `COOLDOWN_MINUTES` | `5` | 触发 429 后冷却时间（分钟）|
| `CLAUDE_API_KEY` | — | 保护 `/v1/messages` 的 API Key（可选）|

---

## 数据库连接说明

默认使用 `docker-compose.yml` 内置的 PostgreSQL 容器，`DB_HOST` 固定填服务名 `postgres`，无需额外安装。

```env
# 默认：使用内置 postgres 容器
DB_HOST=postgres
DB_PORT=5432
DB_USER=claude2api
DB_PASS=your_strong_password
DB_NAME=claude2api
# DB_SSLMODE 默认 disable，本地容器无需修改
```

### 连接外部 PostgreSQL（VPS / 自建）

将 `DB_HOST` 改为对应 IP 或域名，并删除 `docker-compose.yml` 中的 `postgres` 服务和 `depends_on` 相关配置：

```env
DB_HOST=db.example.com
DB_PORT=5432
DB_USER=claude2api
DB_PASS=your_strong_password
DB_NAME=claude2api
DB_SSLMODE=disable  # 本地网络/VPS 通常不需要 SSL
```

### 连接托管 PostgreSQL（Neon / Supabase / Railway 等）

这类平台强制要求 TLS 连接。推荐直接使用平台提供的完整连接串（`DATABASE_URL`）：

```env
# Neon
DATABASE_URL=postgres://user:pass@ep-xxx-yyy.us-east-2.aws.neon.tech/dbname?sslmode=require

# Supabase
DATABASE_URL=postgres://postgres:pass@db.xxxxxxxx.supabase.co:5432/postgres?sslmode=require
```

或者分字段配置（效果相同）：

```env
DB_HOST=ep-xxx-yyy.us-east-2.aws.neon.tech
DB_PORT=5432
DB_USER=your_neon_user
DB_PASS=your_neon_password
DB_NAME=your_db_name
DB_SSLMODE=require   # ← 关键：托管数据库必须设置
```

> **注意**：使用托管数据库时，需删除 `docker-compose.yml` 中的 `postgres` 服务及 `claude2api` 下的 `depends_on.postgres` 配置。

---

## Redis 连接说明

默认使用 `docker-compose.yml` 内置的 Redis 容器。`REDIS_PASS` 和 `REDIS_URL` 中的密码必须保持一致。

```env
# 默认：使用内置 redis 容器
REDIS_PASS=your_redis_password
REDIS_URL=redis://:your_redis_password@redis:6379/0
```

如需连接外部 Redis，将 `REDIS_URL` 中的主机名改为实际地址，并删除 `docker-compose.yml` 中的 `redis` 服务和 `depends_on` 相关配置：

```env
# 示例：外部 Redis（带密码，非标准端口）
REDIS_URL=redis://:your_redis_password@redis.example.com:6388/0
```

连接串通用格式：
```
redis://:密码@主机:端口/库编号
```

---

## 代理配置说明

`PROXY_HOST`、`PROXY_PORT`、`PROXY_USER`、`PROXY_PASS`、`PROXY_REGION` 共同组成代理连接信息。留空则直连 claude.ai。

代理地址由服务内部按以下规则自动拼接：

| 场景 | 实际效果 |
|---|---|
| 仅填 `PROXY_HOST` / `PROXY_PORT` | 直连代理，无认证 |
| 填写 `PROXY_USER` / `PROXY_PASS` | 固定用户名密码认证 |
| 用户名含 `{sid}` 占位符 | 按账号固定出口 IP（旋转住宅代理）|
| 用户名含 `region-XX` | 指定出口区域（arxlabs 等服务商格式）|

管理面板中也可为每个账号单独配置代理，支持以下格式：

```
# 直连（不走代理）
留空

# 固定代理（无认证）
http://proxy.example.com:3128

# 固定代理（用户名密码认证）
http://user:pass@proxy.example.com:3010

# 旋转住宅代理（按账号固定 IP，{sid} 会被替换为账号唯一标识）
http://user-session-{sid}:pass@us.arxlabs.io:3010

# 旋转住宅代理 + 指定区域（arxlabs / brightdata 等常见格式）
http://user-region-US-session-{sid}:pass@us.arxlabs.io:3010
http://user-country-JP-session-{sid}:pass@proxy.provider.io:3010
```

---

## API 使用

服务兼容 Anthropic Messages API：

```bash
curl http://localhost:8080/v1/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $CLAUDE_API_KEY" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "你好！"}]
  }'
```

未设置 `CLAUDE_API_KEY` 时，`Authorization` 头可省略。

### 支持的模型名

| 请求模型名（示例） | 实际路由 |
|---|---|
| `claude-sonnet-4-6` | claude-sonnet-4-6 |
| `claude-sonnet-4-5` | claude-sonnet-4-5-20250929 |
| `claude-haiku-4-5` | claude-haiku-4-5-20251001 |
| `claude-opus-4-6` | claude-opus-4-6（需要 Pro）|
| `*-thinking` 后缀 | 同模型 + 开启扩展思考 |
| 自定义映射 | 在管理面板配置 |

---

## 管理面板

访问 `http://localhost:8080/admin`，使用 `ADMIN_USER` / `ADMIN_PASS` 登录。

功能：
- 账号管理（添加 / 批量导入 / 删除 / 状态监控）
- 代理配置（支持 `{sid}` 占位符实现按账号固定 IP）
- 模型名映射（自定义入参模型名 → Claude.ai 模型名）
- 实时 metrics（请求数、延迟、成功率）
- 全局限速配置
- API Key 管理
- 支持中文 / English 切换，深色 / 浅色主题

---

## Docker 镜像说明

| 镜像标签 | 架构 | 适用场景 |
|---|---|---|
| `pushzx/claude2api:latest` | amd64 (x86_64) | 普通 PC / 云服务器 |
| `pushzx/claude2api:latest-arm64` | arm64 (aarch64) | 树莓派、Apple Silicon、ARM 云服务器 |

ARM 设备部署时，在 `docker-compose.yml` 中将镜像名改为：

```yaml
image: pushzx/claude2api:latest-arm64
```

---

## 更新镜像

```bash
docker compose pull
docker compose up -d
```

---

## 更新日志

### v0.9.0 — 2026-03-24
**账号健康检测**

管理面板：
- 新增单账号测活接口 `POST /api/accounts/test`：对指定账号发起真实 Claude.ai 探测请求，自动标记失效账号为封禁状态
- 新增批量测活接口 `POST /api/accounts/test-all`：一键检测所有非封禁账号的存活状态
- UI 账号列表每行新增 **Test** 按钮（橙色）和顶部 **一键测活** 批量按钮（蓝色）
- 失效账号测试后自动置为 `banned`，无需手动操作

后端：
- `handler/admin.go`：新增测活路由及并发检测逻辑
- `database/db.go` / `database/mem.go`：新增测活状态写入支持
- `config/config.go`：相关配置字段补全

---

### v0.8.0 — 2026-03-15
**支持托管 PostgreSQL（Neon / Supabase / Railway 等强制 TLS 连接）**

后端：
- `database/db.go`：`New()` 函数不再硬编码 `sslmode=disable`，改为接收 `sslmode` 参数；新增 `databaseURL` 参数，支持直接传入完整 DSN，优先级高于所有分字段配置
- `config/config.go`：新增 `DBSSLMode`（读取 `DB_SSLMODE` 环境变量，默认 `disable`）和 `DatabaseURL`（读取 `DATABASE_URL` 环境变量）两个配置字段
- `main.go`：新增 `--db-sslmode` 和 `--database-url` CLI 参数

新增环境变量：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `DB_SSLMODE` | `disable` | PostgreSQL SSL 模式，本地无需修改；Neon/Supabase 填 `require` |
| `DATABASE_URL` | — | 完整 DSN，优先于所有 `DB_*` 字段（推荐 Neon/Supabase 使用）|

**`docker-compose.yml` 无需修改**（新变量默认值向下兼容），仅需在 `.env` 中按需添加。

---

### v0.7.0 — 2026-03-11
**新增 SOCKS5/SOCKS4 代理支持 + SOCKS 错误自动重试**

后端：
- `claude/client.go`：`isNetworkError()` 新增 `socks5`、`socks4`、`socks connect`、`proxy` 关键字检测，SOCKS 代理连接失败时自动触发重试

管理面板：
- Fixed 代理卡片新增 `socks5://user:pass@host:port`、`socks5h://user:pass@host:port` 示例格式
- Rotating 代理卡片新增 SOCKS5 旋转代理示例（`socks5://user-session-{sid}:pass@host:port`）
- 中英文 i18n 描述同步更新，注明同时支持 `socks4` / `socks4a` 格式
- tls-client 基于 `golang.org/x/net/proxy` 原生支持全部 SOCKS 协议，无需额外依赖

代理格式（管理面板中可直接填写）：

```
# SOCKS5（标准，远端 DNS 解析）
socks5://user:pass@host:port

# SOCKS5h（DNS 交由代理服务器解析，适合穿透场景）
socks5h://user:pass@host:port

# SOCKS4 / SOCKS4a
socks4://host:port
socks4a://host:port

# 旋转 SOCKS5（{sid} 替换为账号唯一标识）
socks5://user-session-{sid}:pass@host:port
```

---

### v0.6.0 — 2026-03-10
**修复 403 封禁判断、账号过滤器、输入样式、上下文错误处理**

后端：
- `IsBanError`：移除对 `permission_error` 的豁免 — 403 + `account_session_invalid` 现在正确标记账号为封禁（之前被静默忽略）
- 新增 Claude stream error 事件处理（上下文过长等）：流式路径发送 SSE error 事件；非流式返回 HTTP 400 + 标准 JSON 错误体

前端（管理面板）：
- 账号状态过滤 Pills（全部 / 活跃 / 冷却 / 封禁），实时显示各状态数量
- 邮箱 + Session Key 搜索框（纯客户端过滤，无额外请求）
- `renderAccounts()` 与 `refreshAccounts()` 解耦，过滤操作即时响应
- 模型映射新增行：虚线卡片展示 INPUT → TARGET 标签和箭头
- CSS：`input:not([type])` 选择器修复裸 `<input>` 标签（mmNewKey、mmCustomVal、accountSearch 在深色模式下白色背景问题）
- `select` 元素与 `input` 样式统一（边框、圆角、焦点环）
- i18n 新增：filterAll / filterActive / filterLimited / filterBanned / searchPlaceholder

---

### v0.5.0 — 2026-03-10
**新增精确 Token 计数（tiktoken-go / cl100k_base）+ ARM64 镜像支持**

- `claude/tokens.go`：用 `github.com/tiktoken-go/tokenizer` 替换原有字符估算启发式算法，采用 cl100k_base 编码（与 Claude/GPT-4 一致）；codec 通过 `sync.Once` 初始化，不可用时自动回退到字符估算
- `EstimateRequestTokens`：精确统计 system、messages（text / tool_use / tool_result / image）及工具定义的 token 数
- `StreamTranslator`：在每个 `text_delta` / `thinking_delta` 事件中累计 `outputTokens`
- 所有 SSE `message_delta` 及非流式响应现在携带真实的 `input_tokens` / `output_tokens`
- 新增 `pushzx/claude2api:latest-arm64` Docker 镜像，支持树莓派等 ARM 设备部署

---

### v0.4.0 — 2026-03-10
**Fix 502 EOF：TLS 客户端缓存 + 全链路网络重试**

根本原因：原版每次 API 调用都创建全新的 TLS 客户端，导致每个子请求（GetOrgID、CreateConversation、SendMessage、UploadFile 等）都需要重新建立 TCP+TLS 握手。在住宅代理环境下，频繁触发 EOF / 连接重置错误。

修复内容：
- `proxy/session.go`：按 `(sessionKey + proxyURL)` 缓存 TLS 客户端，同一账号复用 keep-alive 连接；代理地址变更时自动创建新连接（旧缓存 GC 回收）；新增 `InvalidateAccount()` 支持封号/错误时主动驱逐缓存
- `claude/client.go`：新增 `isNetworkError()` + `doWithRetry()` 辅助函数，所有 HTTP 调用在 EOF、连接重置、broken pipe、超时、TLS 握手错误等瞬态网络异常时自动重试最多 3 次，每次重试前主动废弃缓存连接以确保建立新连接
- `handler/messages.go`：消息处理改进

---

### v0.3.0 — 2026-03-09
**修复与优化：流式输出、重试、多轮工具调用历史**

- **工具调用多轮历史**：`buildAssistantTurn` 正确合并 text + tool_use；`tool_result` 作为顶层段落输出，不再错误地包裹 `**User**:` 前缀
- **ReAct 解析**：`extractJSONObject` 改为大括号深度解析器，替换原有正则，正确处理嵌套 `{}` 的 Python 代码块（字典、类定义、f-string）
- **图片上传**：改为并发 goroutine 上传，N 张图片耗时从 `N×时间` 降为 `max(上传时间)`；每个文件自动重试 3 次
- **其他**：启动时自动加载 `.env`；Paprika 模式缓存（`sync.Map`）避免每次请求都触发冗余 PATCH

---

### v0.2.0 — 2026-03-08
**完整 tool_use 实现（ReAct 格式，参考 web2api 方案）**

- 复刻 web2api 经过验证的 ReAct 工具调用策略，适配 Anthropic API 格式
- **Prompt 侧**：携带 `tools` 数组时自动注入 ReAct 格式规范 + 工具定义；多轮历史中 `tool_use` 块转为 Action/Action Input，`tool_result` 块转为 Observation，支持连续工具调用推理
- **流式解析**：`reActActive` 模式下文本块静默缓存；`message_delta` 时 `ParseReActOutput` 检测工具调用/最终回答，触发正确的 Anthropic SSE 事件序列
- **非流式**：收集 `content_block_start(tool_use)` + `input_json_delta`，构建完整 `tool_use` 响应块
- `stop_reason` 在检测到工具调用时正确设置为 `"tool_use"`
