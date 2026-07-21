# Skill 命令参考

ADP/OpenBKN Skill 资源域：注册、市场查找、渐进式读取、下载与本地安装。

与 CLI 一致：运行 `openbkn skill --help` 或 `openbkn skill <subcommand> --help` 查看当前版本同步的参数。

## 常用命令

```bash
openbkn skill list [--name <kw>] [--status unpublish|published|offline] [--page-size 30]
openbkn skill market [--name <kw>] [--source <src>] [--page-size 30]
openbkn skill get <skill_id>
openbkn skill register --content-file <path> [--source <src>] [--extend-info '{"tag":"demo"}']
openbkn skill register --zip-file <skill.zip> [--source <src>] [--extend-info '{"tag":"demo"}']
openbkn skill status <skill_id> <unpublish|published|offline>
openbkn skill delete <skill_id> [-y]
```

## 渐进式读取

```bash
# Read the content index only (download URL + file manifest)
openbkn skill content <skill_id>

# Fetch and print SKILL.md body
openbkn skill content <skill_id> --raw

# Save SKILL.md locally
openbkn skill content <skill_id> --output ./SKILL.md

# Read one file entry
openbkn skill read-file <skill_id> references/guide.md

# Fetch file bytes and save locally
openbkn skill read-file <skill_id> references/guide.md --output ./guide.md
```

## 下载与安装

```bash
# Download the ZIP archive
openbkn skill download <skill_id> --output ./demo-skill.zip

# Install to a local folder (extract ZIP)
openbkn skill install <skill_id> ./skills/demo-skill

# Replace an existing non-empty directory
openbkn skill install <skill_id> ./skills/demo-skill --force
```

## 说明

- `list` 与 `market` 默认每页 30 条，`--page-size` / `--limit` 可覆盖
- `content` / `read-file` 默认返回索引元数据；加 `--raw` 或 `--output` 才会真正抓取远端内容
- `register --content-file` 支持上传单个 `SKILL.md`，也支持指定包含 `SKILL.md` 的目录（会自动打包为 ZIP）；`register --zip-file` 适合上传已构建好的 `.zip` 压缩包
- `install` 本质是 `download + unzip` 的本地便捷封装
