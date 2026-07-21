---
name: bkn-openbkn
description: >-
  OpenBKN SDK / CLI 操作参考层。
  覆盖认证、平台业务域（config）、知识网络管理与查询、Agent CRUD 与对话、
  数据源管理、数据视图、Skill 注册、Vega 可观测、context-loader 语义搜索、通用 API 调用。
  仅由 bkn-creator pipeline 内部读取，不独立注册到宿主 skill 系统。
---

# OpenBKN CLI

OpenBKN 平台的命令行工具，覆盖认证、平台业务域（`config`）、知识网络管理与查询、Agent CRUD 与对话、数据源管理。

## 安装

```bash
npm install -g @openbkn/bkn-sdk
```

需 Node.js 22+。也可用 `npx @openbkn/bkn-sdk` 临时运行。

## 使用方式

```bash
openbkn [--user <userId|username>] <command> [subcommand] [options]
```

**完整子命令与参数以当前安装的 CLI 为准**：运行 `openbkn --help`（或 `-h`）查看与代码同步的用法列表；查版本用 `openbkn --version` / `-V` / `openbkn version`。子命令细节用 `openbkn <group> <subcommand> --help`（例如 `openbkn auth --help`、`openbkn bkn push --help`）。

本 skill 下的 `references/*.md` 与 CLI 行为对齐；**表格与 reference 为速查**，新增标志（如 `auth` 的 `--alias`、BKN `validate`/`push` 的编码选项）在 reference 中有说明。

**别名**：`openbkn curl` 等同于 `openbkn call`；`openbkn context` 等同于 `openbkn context-loader`。

**业务域（business domain）**：多数请求依赖 `x-business-domain`。`openbkn auth login` 后应优先执行 `openbkn config show`；列表为空时用 `config list-bd` 查看平台可选域，再 `config set-bd <uuid>`。详见 [`references/config.md`](references/config.md)。

## 使用前提

**认证凭据通过 `~/.openbkn/` 管理。默认操作：在存在 `refresh_token` 时，用 OAuth2 `refresh_token` 授权换发新的 `access_token`（过期或临近过期时自动执行，无需额外参数）。禁止提前检查环境变量，禁止询问用户提供密码或 Token。**

### 认证优先级

1. `OPENBKN_TOKEN` + `OPENBKN_BASE_URL` 环境变量 → 静态 Token（如存在则优先使用，**不会**用 refresh 换发）
2. `~/.openbkn/` 凭据（`openbkn auth login` 写入）→ **默认**用 refresh_token 换发 access_token（推荐）
3. `OPENBKN_USER` 环境变量（或全局 `--user` 参数）→ 使用指定用户的凭证，不切换活跃用户

### 业务域优先级（与认证独立）

1. `OPENBKN_BUSINESS_DOMAIN` 环境变量  
2. 当前平台 `config.json` 中的 `businessDomain`（`openbkn config set-bd`）  
3. 默认 `bd_public`（首次登录后 CLI 可能已自动写入更合适的值）

## 命令组总览

| 命令组 | 说明 | 常用命令 | 详细参考 |
|--------|------|---------|---------|
| `auth` | 认证管理（支持多账号） | `auth login <url> [--alias name]`（简写：`auth <url> [--alias …]`）；可选 `--no-browser`、`-u/-p` HTTP `/oauth2/signin`；**初始密码**（401001017）下 TTY 可交互改密，脚本用 `--new-password`；`auth change-password [<url>] [-u …]`（EACP 改密；URL 与 `-u` 都可省略，分别回退到当前平台与当前激活账号；无需 token）；`auth list` / `auth users` / `auth switch`；全局 `--user` / `OPENBKN_USER`；**无当前平台时** `auth status` / `whoami` 可用 env 兜底（见 `references/auth.md`） | `references/auth.md` |
| `token` | 打印当前 access token（自动刷新） | `token` | — |
| `config` | **平台业务域（优先于多数 bkn/agent/ds 操作）** | `config show`, `config list-bd`, `config set-bd <uuid>` | `references/config.md` |
| `bkn` | BKN 知识网络管理、Schema、查询、Action | `bkn validate`/`push` 默认检测 `.bkn` 编码并规范为 UTF-8，可用 `--no-detect-encoding` 或 `--source-encoding gb18030`；另有 `pull`、`object-type`、`search`、`create-from-ds`/`create-from-csv` 等，见 `references/bkn.md` | `references/bkn.md` |
| `agent` | Agent CRUD、发布、对话、Trace、模板、分类 | `agent list`, `agent get <id>`, `agent create --name <n> --profile <p> --config <json>`, `agent publish <id> --category-id <cid>`, `agent chat <id> -m "..."`、`agent category-list`, `agent template-list`, `agent template-get <tpl_id>`、`agent sessions <agent_id>`、`agent history <conversation_id>`、`agent trace <conversation_id>` | `references/agent.md` |
| `ds` | 数据源管理 | `ds list`, `ds get <id>`, `ds import-csv <ds_id> --files <glob> [--recreate]` | `references/ds.md` |
| `dataview` | 数据视图（mdl-data-model / vega-backend） | `dataview list`、`find --name`、`get`、`query`、`delete`；BKN 绑定也可用 `vega resource` ID（type=resource） | `references/dataview.md` |
| `dataflow` | Dataflow 文档流程 | `dataflow list`, `dataflow run <dagId> --file <path>`, `dataflow run <dagId> --url <remote-url> --name <filename>`, `dataflow runs <dagId> [--since <date-like>]`, `dataflow logs <dagId> <instanceId> [--detail]` | `references/dataflow.md` |
| `skill` | Skill 注册、市场查找、渐进式读取、下载与安装 | `skill list`、`market`、`register --zip-file`、`content`、`read-file`、`install` | `references/skill.md` |
| `toolbox` | 平台工具箱（toolbox）管理 | `toolbox create --name <n> --service-url <url>`、`toolbox list`、`toolbox publish/unpublish <id>`、`toolbox delete <id> [-y]` | `references/toolbox.md` |
| `tool` | 工具箱内 tool 注册与启停（OpenAPI） | `tool upload --toolbox <id> <openapi-spec>`、`tool list --toolbox <id>`、`tool enable/disable --toolbox <id> <tool-id>...` | `references/tool.md` |
| `vega` | Vega 可观测平台 | `vega health`, `vega catalog list`, `vega resource list`, `vega query execute -d <json>`, `vega sql --resource-type <t> --query "<sql>"` / `vega sql -d <json>` | `references/vega.md` |
| `context-loader` | MCP 分层检索 | `context-loader config show`, `context-loader kn-search <query>` | `references/context-loader.md` |
| `call` | 通用 API 调用 | `call <url> [-X POST] [-d '...']`（可用 `curl` 别名；支持 `--url`、`--data-raw` 等，见 `openbkn --help`） | `references/call.md` |

## 操作指南

| 场景 | 说明 | 详细参考 |
|------|------|---------|
| 登录后确认业务域 | `config show`；若异常或列表为空 → `config list-bd` → `config set-bd <uuid>` | [references/config.md](references/config.md) |
| 从数据库/CSV 构建 KN | 连接数据源 → CSV 导入 → 创建 KN → 构建索引 → 查询验证 → 绑定 Agent | [references/build-kn-from-db.md](references/build-kn-from-db.md) |
| CLI 排障速查 | 权限、pull、build、import、dataview SQL 等 | [references/troubleshooting.md](references/troubleshooting.md) |
| 列/查数据视图 | `list` 浏览；`find --name` 按名搜索（`--exact`/`--wait`）；`query` 对视图跑 SQL | [references/dataview.md](references/dataview.md) |
| 管理 Dataflow 文档流程 | `list` 看 DAG；`run` 触发本地文件或远程 URL；`runs --since` 看自然日运行记录；`logs --detail` 查步骤载荷 | [references/dataflow.md](references/dataflow.md) |
| Trace 数据分析 | `agent trace <conversation_id>` 获取 trace 数据，构建证据链 | — |
| 管理 Skill | `list` / `market` 查找 Skill；`content` / `read-file` 渐进式读取；`install` 下载并解压本地使用 | [references/skill.md](references/skill.md) |
| 注册外部工具 | `toolbox create` 建箱 → `tool upload` 上传 OpenAPI → `tool list` 拿 `tool_id` → `tool enable` 启用 → `toolbox publish` 切到 published | [references/toolbox.md](references/toolbox.md) · [references/tool.md](references/tool.md) |

**按需阅读**：需要子命令完整参数或编排示例时，读取对应的 reference 文件。

## 调用示例

```
/openbkn 列出所有知识网络
/openbkn 查看 Vega 健康状况
/openbkn 有哪些 Agent
/openbkn 跟 Agent xxx 对话，问他"今天库存情况"
/openbkn 搜索知识网络 xxx 中关于"供应链"的内容
/openbkn 用 dataview find 模糊搜索名字含 BOM 的数据视图
/openbkn 列出所有 dataflow
/openbkn 触发 dataflow 123，上传本地文件 ./demo.pdf
/openbkn 查看 dataflow 123 在 2026-04-01 的运行记录
/openbkn 查看 dataflow 123 的实例 456 日志，并展开 input output
/openbkn 列出所有 Agent 模板
/openbkn 基于 "数据分析助手" 模板创建一个新的 Agent
/openbkn 在 skill market 里查找名字包含 openbkn 的 skill
/openbkn 读取 skill xxx 的 SKILL.md 并保存到本地目录
/openbkn 创建一个名为 weather-svc 的 toolbox，对接 https://weather.example.com
/openbkn 把 ./openapi.json 上传到 toolbox 1234567890 并启用所有工具，最后发布
```

## 注意事项

- **不要自行猜测 business_domain 值**。首次使用时运行 `openbkn config show` 或 `openbkn config list-bd` 确认当前 business domain。如果返回 `bd_public (default)` 但命令结果为空，可能需要用 `openbkn config set-bd <uuid>` 设置正确的值（也可用 `config list-bd` 从平台列出后再 `set-bd`，或从平台 UI 请求头中获取 `X-Business-Domain`）
- Action 执行有副作用，执行前向用户确认
- **禁止运行 `openbkn auth status` 做预检**。直接执行目标命令，CLI 会自动处理认证和 token 刷新
- Token 1 小时过期。当 `~/.openbkn/` 中存在 `refresh_token`（通过 OAuth2 登录获得）时，CLI 会**自动刷新**；遇到 401 错误时 CLI 会自动尝试刷新，刷新失败才提示用户重新运行 `openbkn auth login <url>`

## 查询策略（object-type query）

调用 `object-type query` 时必须限制 `limit`、用 `search_after` 分页、用 `condition` 过滤，避免宽表 JSON 截断。完整规则与示例见 [`references/bkn.md`](references/bkn.md#object-type-query-strategy-for-llm-and-agent)。
