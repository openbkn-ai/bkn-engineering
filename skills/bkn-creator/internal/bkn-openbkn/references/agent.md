# Agent 命令参考

Decision Agent CRUD、发布管理与对话。

与 CLI 一致：运行 `openbkn agent` 或 `openbkn agent chat --help` 等可查看与当前版本同步的用法。

## CRUD 命令

```bash
# 已发布的 Agent
openbkn agent list [--name <kw>] [--limit 50] [--verbose]
openbkn agent get <agent_id> [--verbose] [--save-config <path>]
openbkn agent get-by-key <key>

# 私人空间的 Agent
openbkn agent personal-list [--name <kw>] [--size 48] [--verbose]

# Agent 模板
openbkn agent template-list [--name <kw>] [--size 48] [--verbose]
openbkn agent template-get <template_id> [--save-config <path>] [--verbose]

# Agent 分类
openbkn agent category-list [--verbose]

# 创建 Agent
openbkn agent create --name <name> --profile <profile> --llm-id <model_id> [--key <key>] [--product-key DIP|AnyShare|ChatBI] [--system-prompt <sp>] [--llm-max-tokens 4096] [--config <json|path>]

# 更新/删除
openbkn agent update <agent_id> [--name <n>] [--profile <p>] [--system-prompt <sp>] [--knowledge-network-id <id> [--config-path <path>]]
openbkn agent delete <agent_id> [-y]
```

## 发布管理

```bash
openbkn agent publish <agent_id> [--category-id <category_id>]
openbkn agent unpublish <agent_id>
```

**发布说明**：
- `--category-id`：指定 Agent 分类（可选）
- 默认发布到广场（square）
- 发布时会使用默认配置：
  ```json
  {
    "business_domain_id": "bd_public",
    "category_ids": ["<category_id>"] | [],
    "description": "",
    "publish_to_where": ["square"],
    "pms_control": null
  }
  ```

## 对话

```bash
openbkn agent chat <agent_id> -m '<message>' [--conversation-id <id>] [--stream/--no-stream]
openbkn agent chat <agent_id>                    # 交互式模式
openbkn agent sessions <agent_id> [--limit <n>]
openbkn agent history <agent_id> <conversation_id>
openbkn agent trace <conversation_id> [--pretty|--compact]
```

## Trace 数据

```bash
openbkn agent trace <conversation_id>
```

获取指定会话的 trace 数据，用于追踪数据流、调试问题、构建证据链。

选项：
- `--pretty`：格式化 JSON 输出（默认）
- `--compact`：紧凑 JSON 输出

## 说明

- `create` 需要 `--llm-id`，可通过模型工厂 API 查询可用 LLM：`GET /api/mf-model-manager/v1/llm/list?page=1&size=100`
- `get` 的 `--save-config` 自动添加时间戳防止文件被覆盖，输出文件路径格式：`<basename>-<timestamp>.<ext>`
- `update` 的 `--config-path` 从指定路径读取配置文件（由 `get --save-config` 生成），`--knowledge-network-id` 配置业务知识网络
- `create` 的 `--config` 支持两种方式：
  - **文件路径**：`--config /path/to/config.json`（推荐，避免长度限制）
  - **JSON 字符串**：`--config '{"input":...,"llms":...}'`
- `template-get` 的 `--save-config` 自动添加时间戳防止文件被覆盖
- `update` 采用 read-modify-write 模式：先 GET 当前配置，修改字段后 PUT 回去
- `list` 只返回已发布的 agent；`get` 可以获取未发布的（需要是 owner）
- `publish` 后 agent 才会出现在 `list` 里

## 更新 Agent 知识网络配置

通过 `get --save-config` 保存配置，然后使用 `update --config-path --knowledge-network-id` 更新。

```bash
# 1. 获取并保存 Agent 配置（自动添加时间戳）
openbkn agent get <agent_id> --save-config /tmp/agent_config.json
# 输出: /tmp/agent_config-2026-04-02T14-50-55.json

# 2. 更新知识网络配置
openbkn agent update <agent_id> --config-path /tmp/agent_config-2026-04-02T14-50-55.json --knowledge-network-id d5ordervm0qr3o2trdn0

# 3. 重新发布使配置生效
openbkn agent publish <agent_id>
```

**选项说明**：
- `--save-config <path>`：保存配置到文件，自动添加时间戳防止覆盖
  - 支持目录路径（以 `/` 结尾），自动生成文件名
  - 自动创建不存在的目录
- `--config-path <path>`：从文件读取配置（配合 `--save-config` 使用）
- `--knowledge-network-id <id>`：配置业务知识网络ID

**简写方式**（不保存文件）：
```bash
# 直接更新知识网络（自动从API获取当前配置）
openbkn agent update <agent_id> --knowledge-network-id <kn_id>
```

## 基于模板创建 Agent

通过模板快速创建 Agent，避免手动配置复杂的 config 对象。

### 方式一：使用 --save-config（推荐）

直接保存模板配置到文件，避免长 JSON 字符串被截断。

```bash
# 1. 列举所有模板
openbkn agent template-list

# 2. 保存模板配置到文件（自动添加时间戳，防止覆盖）
openbkn agent template-get <template_id> --save-config /tmp/config.json
# 输出: /tmp/config-2026-04-02T14-30-45.json

# 3. 使用配置文件创建 Agent
openbkn agent create --name "我的Agent" --profile "描述" --config /tmp/config-2026-04-02T14-30-45.json
```

**--save-config 说明**：
- 输出文件路径自动添加时间戳，格式：`<basename>-<timestamp>.<ext>`
- 支持目录路径（以 `/` 结尾），自动生成文件名：`/tmp/dir/` → `/tmp/dir/agent-config-2026-04-02T14-30-45.json`
- 自动创建不存在的目录

### 方式二：手动提取配置

```bash
# 1. 获取模板详情
openbkn agent template-get <template_id> --verbose

# 2. 从返回的 JSON 中提取 config 对象，手动创建 Agent
openbkn agent create --name "我的Agent" --profile "描述" --config '{"input":{...},"llms":...}'
```

### 完整示例

```bash
# 1. 列举所有分类（可选）
openbkn agent category-list

# 2. 列举所有模板
openbkn agent template-list

# 返回示例：
# [
#   {"id": "88", "name": "合同审核助手演示版_模板", "description": "..."},
#   {"id": "92", "name": "业务知识网络召回_模板", "description": "..."}
# ]

# 3. 保存模板配置
CONFIG=$(openbkn agent template-get 88 --save-config /tmp/contract-audit.json)
echo "配置已保存到: $CONFIG"

# 4. 创建 Agent
AGENT_ID=$(openbkn agent create --name "合同审核助手" --profile "基于模板创建" --config "$CONFIG" | jq -r '.id')

# 5. 发布 Agent
openbkn agent publish $AGENT_ID
```

## 端到端示例

```bash
# 方式一：从零创建 → 配置知识网络 → 发布 → 对话 → 清理
openbkn agent create --name "测试助手" --profile "SDK 测试用" --llm-id <model_id> --system-prompt "你是一个测试助手"
openbkn agent update <agent_id> --knowledge-network-id <kn_id>
openbkn agent publish <agent_id> --category-id 01JRYRKP0M8VYHQSX4FXR5CKG1
openbkn agent chat <agent_id> -m "你好"
openbkn agent unpublish <agent_id>
openbkn agent delete <agent_id> -y

# 方式二：基于模板创建（推荐）
CONFIG=$(openbkn agent template-get 88 --save-config /tmp/config.json)
AGENT_ID=$(openbkn agent create --name "合同审核助手" --profile "描述" --config "$CONFIG" | jq -r '.id')
openbkn agent publish $AGENT_ID
openbkn agent chat $AGENT_ID -m "帮忙审核合同：JJFAGHBJF25090012"

# 多轮对话
openbkn agent chat <agent_id> -m "分析库存数据" --no-stream
openbkn agent chat <agent_id> -m "给出改进建议" --conversation-id <conv_id>
openbkn agent history <agent_id> <conv_id>
```
## Trace 数据分析

当用户需要追踪数据流、调试问题、理解结果如何从 trace 数据中得出时，使用 `openbkn agent trace` 命令获取 trace 数据并构建证据链。

### 使用场景

- 用户想了解某个结果是如何得出的
- 用户需要追踪数据在系统中的流转
- 用户想通过 trace 数据调试问题
- 用户询问"证据链"或"因果关系"

### 操作步骤

1. **获取 Conversation ID**：从用户处获取或通过 `openbkn agent sessions <agent_id>` 查询

2. **获取 Trace 数据**：
   ```bash
   openbkn agent trace <conversation_id>
   ```
   
   选项：
   - `--pretty`：格式化输出（默认）
   - `--compact`：紧凑输出

3. **解析并分析 Trace 数据**：
   - 解析 JSON 响应
   - 识别关键 spans 及其关系
   - 查找与用户问题匹配的事件
   - 构建操作时间线

4. **构建证据链**：
   ```
   [步骤 1] → [步骤 2] → [步骤 3] → [结果]
      ↓           ↓           ↓
   [输入]     [处理]      [输出]
   ```

5. **呈现分析结果**：
   - 清晰的步骤说明
   - 每步的关键数据点
   - 步骤间的因果关系
   - 回答用户问题的结论

### 示例

**用户问题**："为什么订单失败了？"

**证据链**：
```
[HTTP 请求] → [校验] → [支付检查] → [失败]
      ↓           ↓           ↓          ↓
   订单数据    校验通过     余额不足    订单被拒绝
   已接收      但有警告     已检测到
```

**解释**：
1. 14:30:00 收到订单请求
2. 校验通过但标记了警告
3. 支付检查发现余额不足
4. 订单因支付失败被拒绝

### 分析技巧

- 查找 trace 中的错误事件或异常
- 关注时间戳以理解执行顺序
- 识别 spans 之间的父子关系
- 突出流程中的关键决策点
- 向用户解释时使用清晰、非技术性的语言