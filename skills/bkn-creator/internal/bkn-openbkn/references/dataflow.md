# Dataflow 命令参考（dataflow）

用于操作 OpenBKN 的 Dataflow 文档流程，覆盖 DAG 列表、手动触发、运行记录和步骤日志。

## 命令

```bash
openbkn dataflow list [-bd value]
openbkn dataflow run <dagId> (--file <path> | --url <remote-url> --name <filename>) [-bd value]
openbkn dataflow runs <dagId> [--since <date-like>] [-bd value]
openbkn dataflow logs <dagId> <instanceId> [--detail] [-bd value]
```

## 子命令说明

### `list`

- 列出所有 dataflow DAG。
- CLI 以表格展示 `ID`、`Title`、`Status`、`Trigger`、`Creator`、`Updated At`、`Version ID`。
- 当前实现固定请求全部 DAG，不暴露分页参数。

```bash
openbkn dataflow list
```

### `run`

- 触发一次 dataflow 运行。
- 输入源二选一：
  - `--file <path>`：上传本地文件
  - `--url <remote-url> --name <filename>`：使用远程文件 URL
- `--file` 与 `--url` 互斥；`--url` 必须同时带 `--name`。
- 成功时只打印 `dag_instance_id`。

```bash
openbkn dataflow run 614185649708255523 --file ./demo.pdf
openbkn dataflow run 614185649708255523 --url https://example.com/demo.pdf --name demo.pdf
```

### `runs`

- 查看指定 DAG 的运行记录。
- 默认行为：
  - 请求最近 20 条
  - 排序参数固定为 `sortBy=started_at&order=desc`
- `--since <date-like>`：
  - 只要能被 `new Date(...)` 解析，就按**本地自然日**生成 `start_time` 和 `end_time`
  - 第一次先取 20 条
  - 若返回 `total > 20`，CLI 会自动补第二次请求取剩余结果
  - 若解析失败，视为未传，回退到最近 20 条
- CLI 以表格展示 `ID`、`Status`、`Started At`、`Ended At`、`Source Name`、`Content Type`、`Size`、`Reason`。

```bash
openbkn dataflow runs 614185649708255523
openbkn dataflow runs 614185649708255523 --since 2026-04-01
openbkn dataflow runs 614185649708255523 --since "2026-04-01T10:30:00+08:00"
```

### `logs`

- 查看一次运行的步骤日志。
- 默认输出摘要块，便于快速扫读执行过程。
- `--detail` 会额外打印缩进后的 `input` 和 `output` pretty JSON。
- CLI 内部按页循环拉取日志，直到取完全部结果；当前页大小固定为 `100`。

```bash
openbkn dataflow logs 614185649708255523 614191966095198499
openbkn dataflow logs 614185649708255523 614191966095198499 --detail
```

默认摘要输出示例：

```text
[0] 0 @trigger/dataflow-doc
Status: success
Started At: 1775616541
Updated At: 1775616541
Duration: 0
```

`--detail` 会在摘要后追加：

```text
    input:
        {
            "foo": "bar"
        }

    output:
        {
            "_type": "file",
            "name": "demo.pdf"
        }
```

## 参数说明

| 选项 | 含义 |
|------|------|
| `--file` | 仅 `run`：上传本地文件 |
| `--url` | 仅 `run`：远程文件地址 |
| `--name` | 仅 `run`：远程文件展示名；与 `--url` 配合必填 |
| `--since` | 仅 `runs`：按本地自然日过滤运行记录；支持任何 `new Date(...)` 可解析的格式 |
| `--detail` | 仅 `logs`：打印缩进后的 `input` / `output` JSON |
| `-bd` / `--biz-domain` | 业务域；默认来自 `openbkn config show` |

## 排障

- `run --file` 失败：先确认本地文件存在且可读。
- `runs --since` 结果不符合预期：确认传入值能被 `new Date(...)` 正确解析；否则 CLI 会退回最近 20 条。
- `logs` 看不到详细载荷：补 `--detail`。
- 结果为空：先用 `openbkn config show` 检查 business domain；必要时切到正确域后重试。
