# 数据语义服务参考

本文件为 bkn-bind 提供数据语义服务的完整参考，包含配置、API 端点、调用流程和注意事项。

## 配置项

| 配置 | 来源 | 默认值 |
|------|------|------|
| `kn_id` | TOOLS.md → `data_semantic_kn_id` | `d6ptuq46vfkhfektuntg` |
| `ot_id` | TOOLS.md → `data_semantic_ot_id` | `d6rmtl46vfkhfektuoe0` |
| `base_url` | TOOLS.md → `data_semantic_base_url` | `https://10.4.134.26/api/data-semantic/v1` |
| `logic_view_base_url` | TOOLS.md → `data_semantic_logic_view_base_url` | `https://10.4.134.26/api/data-view/v1` |

**配置优先级**：TOOLS.md > 默认值

---

## API 端点

### 逻辑视图服务

**Base URL**: `{logic_view_base_url}`

| 端点 | 方法 | 用途 |
|------|------|------|
| `/form-view` | GET | 查询逻辑视图列表 |

**参数**：`keyword`（关键字）、`datasource_id`（数据源筛选）、`limit`/`offset`（分页）

**响应**：
```json
{
  "entries": [{ "id", "business_name", "technical_name", "subject_path" }],
  "total_count": 100
}
```

### 数据语义服务

**Base URL**: `{base_url}`

| 端点 | 方法 | 用途 |
|------|------|------|
| `/:id/status` | GET | 查询理解状态 |
| `/:id/generate` | POST | 触发语义理解 |
| `/:id/submit` | POST | 提交确认 |
| `/:id/fields` | GET | 查询字段语义 |
| `/:id/business-objects` | GET | 查询业务对象 |
| `/batch-object-match` | POST | 批量对象匹配 |

---

## 状态码

| 状态码 | 名称 | 说明 |
|--------|------|------|
| 0 | 未理解 | 尚未进行语义理解 |
| 1 | 理解中 | 语义理解进行中 |
| 2 | 待确认 | 理解完成待确认 |
| 3 | 已完成 | 已确认完成 |
| 4 | 待确认(已重新理解) | 重新理解后待确认 |
| 5 | 理解失败 | 语义理解失败 |

## 状态机流程

```
┌─────────┐ generate ┌─────────┐            ┌─────────┐
│ 0-未理解│ ─────────►│ 1-理解中│ ─────────► │ 2-待确认│
└─────────┘          └─────────┘            └────┬────┘
                                                 │
                               submit            │
                                                 ▼
┌─────────┐ generate ┌─────────┐           ┌──────────┐
│ 3-已完成│ ◄────────│4-待确认 │           │ 5-失败   │
└─────────┘          └─────────┘           └──────────┘
```

| 状态 | 处理动作 |
|------|----------|
| 0 | `POST /:id/generate` → 轮询 |
| 1 | 每 10 秒轮询 `GET /:id/status` → 状态=2 后 `POST /:id/submit` |
| 2 | `POST /:id/submit` → `POST /:id/generate` → 轮询 |
| 3/4 | `POST /:id/generate` → 轮询 |
| 5 | 终止，输出失败原因 |

---

## 字段角色映射

API 返回的 `field_role` 需映射为中文：

| 角色码 | 角色名称 |
|--------|----------|
| 1 | 业务主键 |
| 2 | 关联标识 |
| 3 | 业务状态 |
| 4 | 时间字段 |
| 5 | 业务指标 |
| 6 | 业务特征 |
| 7 | 审计字段 |
| 8 | 技术字段 |

---

## 对象匹配操作

用于 bkn-bind 的对象-视图匹配辅助。

### batchObjectMatch API

**Endpoint**: `POST /batch-object-match`

**请求体**：
```json
{
  "entries": [
    {
      "name": "客户信息",
      "data_source": {
        "id": "view_123",
        "name": "客户视图"
      }
    }
  ],
  "kn_id": "$KNID",
  "ot_id": "$OTID"
}
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `entries` | 是 | 业务对象列表，最多 100 条 |
| `entries[].name` | 是 | 业务对象名称（非空字符串） |
| `entries[].data_source.id` | 否 | 视图 ID |
| `entries[].data_source.name` | 否 | 视图名称 |
| `kn_id` | 是 | GKN 知识网络 ID |
| `ot_id` | 是 | GKN 业务对象类 ID |

**响应**：
```json
{
  "entries": [
    {
      "name": "客户信息",
      "data_source": [
        { "id": "mdl_customer", "name": "客户主档视图", "object_name": "客户信息" }
      ]
    }
  ],
  "need_understand": ["mdl_customer"]
}
```

| 字段 | 说明 |
|------|------|
| `entries[].name` | 原始输入名称 |
| `entries[].data_source[]` | 匹配的视图列表 |
| `need_understand[]` | 需要语义理解的视图 ID 列表 |

### 执行流程

1. 验证输入参数（entries 必填，最多 100 条）
2. 检查 kn_id 和 ot_id 配置（TOOLS.md > 默认值）
3. 获取 JWT Token
4. 调用 `POST /batch-object-match`
5. 处理响应数据

---

## 输出格式

**批量业务对象匹配结果**

| 项目 | 内容 |
|------|------|
| 输入业务对象 | `<count>` 个 |

**匹配统计**

| 项目 | 数量 |
|------|------|
| 成功匹配 | `<matched_count>` 个 |
| 需要理解 | `<need_understand_count>` 个 |

**匹配结果明细**

| 业务对象 | 视图 ID | 视图名称 |
|----------|---------|----------|
| 客户信息 | mdl_xxx | 客户主档视图 |

**需要理解的视图**

| 序号 | 视图 ID | 视图名称 |
|------|---------|----------|
| 1 | view_id_1 | 客户主档视图 |

---

## 注意事项

### 对象匹配

1. **kn_id 和 ot_id 必填**：未配置时提示联系技术工程师
2. **输入限制**：entries 最多 100 条
3. **空值过滤**：自动过滤空 name 的条目
4. **匹配优先级**：data_source.id > 业务对象表 > 视图表
5. **模糊匹配**：使用 LIKE '%name%' 方式

### 中文编码

**⚠️ 调用 API 时**：请求体包含中文必须使用管道方式：
```bash
echo '{"entries":[{"name":"订单"}]}' | curl -d @- -H "Authorization: Bearer $TOKEN"
```
- ❌ `curl -d '{"entries":[{"name":"订单"}]}'`（中文乱码）
- ✅ `echo '...' | curl -d @-`

**⚠️ 解析响应时**：API 返回可能为 GBK 编码，需用 latin1 解码后处理。

---

## 约束

1. 所有 API 调用必须携带有效 JWT Token
2. 字段角色码需映射为中文
3. 状态 5（理解失败）时终止流程
4. 字段语义和业务对象来自不同数据表，数值可能存在差异
5. 不在日志或输出中明文打印 Token