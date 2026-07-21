# OpenBKN CLI troubleshooting (quick reference)

English quick reference for common CLI issues. For step-by-step KN build flows, see [build-kn-from-db.md](build-kn-from-db.md).

| Symptom | Likely cause | What to do |
|--------|----------------|------------|
| `bkn pull` returns 403 / 500 | Permissions or transient API error | Retry; or use `openbkn bkn get <kn_id> --export` if supported for your workflow |
| Build reports primary key / field **unmapped** | `data_properties` missing `mapped_field` for required columns | Use `openbkn bkn object-type create` without `--property` (CLI loads dataview fields and fills mappings), or pass `--property` JSON with `mapped_field`, or recreate OT via `create-from-ds` |
| `create-from-ds` prints **No tables available** | Metadata not ready yet (discovery lag) | Run `openbkn ds tables <ds_id>` to confirm; CLI retries listing a few times—wait and retry the command |
| `ds import-csv` fails for an **existing table** after column changes | Schema mismatch vs previous import | Use `--recreate` on first batch (overwrite) or `--table-prefix` to target a new table |
| `dataview query` errors on **DDL** or odd SQL | Ad-hoc SQL should be `SELECT` / `WITH`; server applies pagination | Use a `SELECT` statement; use `--raw-sql` only if you must bypass the client check |
| `bkn build --wait` **times out** | Large data or slow job | Increase `--timeout`; after timeout, CLI prints last known status—run `openbkn bkn stats <kn_id>` for progress |
