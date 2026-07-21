# Toolbox 命令参考（toolbox）

管理 OpenBKN 平台 **工具箱（toolbox）**。一个 toolbox 关联一个外部服务（`service-url`），承载一组可被 Agent 调用的工具（tool）。
工具的注册（OpenAPI 上传）与启停由配套的 [`tool` 命令组](tool.md) 负责。

后端：`/api/agent-operator-integration/v1/tool-box`。

## 命令

```bash
openbkn toolbox create --name <n> --service-url <url> [--description <d>] [-bd value] [--pretty|--compact]
openbkn toolbox list   [--keyword <s>] [--limit <n>] [--offset <n>] [-bd value] [--pretty|--compact]
openbkn toolbox publish   <box-id> [-bd value]
openbkn toolbox unpublish <box-id> [-bd value]
openbkn toolbox delete    <box-id> [-y|--yes] [-bd value]
openbkn toolbox export    <box-id> [-o <file>|-] [--type toolbox|mcp|operator] [-bd value]
openbkn toolbox import    <file>   [--type toolbox|mcp|operator] [-bd value] [--pretty|--compact]
```

> `export` / `import` 走的是另一组后端入口 `/api/agent-operator-integration/v1/impex/{export|import}/{type}/...`，
> 与上面 `tool-box` 系列接口共用同一个服务，但不在同一个 path 下。

## 子命令说明

### `create`

- 创建一个新 toolbox。
- 必填：`--name`、`--service-url`。`--description` 可选。
- 请求体固定字段：`metadata_type=openapi`、`source=custom`（CLI 不暴露）。
- 输出：后端原始响应（典型形如 `{"box_id": "..."}`），按 `--pretty`（默认）格式化。

```bash
openbkn toolbox create \
  --name "weather-svc" \
  --service-url "https://weather.example.com" \
  --description "外部天气查询服务"
```

### `list`

- 列出当前 business domain 下的 toolbox。
- 可选过滤：`--keyword`（透传给后端）、`--limit`、`--offset`。CLI 不设默认值，未传时由后端决定分页行为。
- `--limit` / `--offset` 必须是数字，否则报 `--limit must be a number` / `--offset must be a number` 并退出非零。
- 输出：后端原始 JSON（典型形如 `{"entries": [...]}`），按 `--pretty`/`--compact` 格式化。

```bash
openbkn toolbox list
openbkn toolbox list --keyword weather --limit 10
```

### `publish` / `unpublish`

- `publish`：把 toolbox 状态改为 `published`。
- `unpublish`：状态改回 `draft`。
- 成功只在 stderr 打印一行 `Published toolbox <id>` / `Unpublished toolbox <id>`，stdout 无内容（适合脚本）。

```bash
openbkn toolbox publish 1234567890123456789
openbkn toolbox unpublish 1234567890123456789
```

### `delete`

- 删除指定 toolbox。
- 默认交互式确认（输入 `y` / `yes` 才执行，否则 stderr 打印 `Aborted.` 并退出非零）；脚本中用 `-y` / `--yes` 跳过确认。
- 成功在 stderr 打印 `Deleted toolbox <id>`。

```bash
openbkn toolbox delete 1234567890123456789          # 交互式
openbkn toolbox delete 1234567890123456789 -y       # 自动确认
```

### `export`

- 拉取一个 toolbox / mcp / operator 的完整配置（`.adp` JSON 文件），用于跨环境迁移或备份。
- `--type` 默认 `toolbox`，可改为 `mcp` 或 `operator`。
- `-o <file>` 指定输出文件名；`-o -` 写到 stdout；省略则落到当前目录的 `<type>_<id>.adp`。
- 成功在 stderr 打印 `Exported <type> <id> → <file> (<n> bytes)`，stdout 留给文件内容（仅当 `-o -` 时）。

```bash
# 导出 toolbox 到默认文件名
openbkn toolbox export 1234567890123456789
# → toolbox_1234567890123456789.adp

# 自定义文件名
openbkn toolbox export 1234567890123456789 -o my-backup.adp

# 直接 piped 到 jq
openbkn toolbox export 1234567890123456789 -o - | jq '.box_name'
```

### `import`

- 把 `export` 拉下来的 `.adp` 文件回灌到目标环境。
- multipart 上传，字段名 `data`（与后端 `ImportConfig` 约定一致）。
- 成功输出后端响应（典型形如 `{"box_id": "..."}` 或 `{"imported": true}`），按 `--pretty`/`--compact` 格式化。

```bash
openbkn toolbox import my-backup.adp
openbkn toolbox import my-backup.adp --type mcp --compact
```

> ⚠️ 导入会在目标环境**新建**对应实体（box/mcp/operator），不会就地更新。若已存在同名/同 ID 实体，按后端策略可能报冲突，请先 `delete` 或换 ID。

## 通用选项

| 选项 | 说明 | 适用子命令 |
|------|------|-----------|
| `-bd, --biz-domain <s>` | 覆盖业务域。默认走 `resolveBusinessDomain()`（`OPENBKN_BUSINESS_DOMAIN` env → 当前平台 `config.json` → `bd_public`） | 全部 |
| `--pretty` | 把响应体当作 JSON 解析后以 2 空格缩进重排（解析失败则按原文输出，默认） | `create`、`list` |
| `--compact` | 原样输出后端响应文本，不做美化（便于管道处理） | `create`、`list` |

## 典型工作流

```bash
# 1. 建 toolbox（拿到 box_id）
BOX_ID=$(openbkn toolbox create --name my-svc --service-url https://my.svc --compact \
         | jq -r '.box_id')

# 2. 上传 OpenAPI 规范，得到该次注册的 tool id 列表
TOOL_IDS=$(openbkn tool upload --toolbox $BOX_ID ./openapi.json --compact \
           | jq -r '.success_ids[]')

# 3. 启用刚注册的 tool
openbkn tool enable --toolbox $BOX_ID $TOOL_IDS

# 4. 发布 toolbox（状态切到 published）
openbkn toolbox publish $BOX_ID
```

> 上面的 `box_id` / `success_ids` 字段以 `packages/typescript/test/e2e/toolbox-tool.test.ts` 验证过的真实响应为准；如果后端版本变更，回退到 `openbkn tool list --toolbox $BOX_ID` 拿 id 即可。

## 关联

- 工具上传/启停：[tool.md](tool.md)
