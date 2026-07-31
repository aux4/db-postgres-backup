#### Description

The `backup` command creates a dump of a PostgreSQL database by invoking the `pg_dump` system CLI. It is a backup **provider**: connection details come from a `config.yaml` profile (via `--config`/`--configFile`) so credentials stay out of any catalog, and the destination is resolved from a path.

This command is contributed by the `aux4/db-postgres-backup` package, which extends the `db:postgres` profile owned by `aux4/db-postgres`. It is packaged separately so the `pg_dump`/`pg_restore`/`psql` system dependency is only required when backup/restore is actually used — the core `aux4/db-postgres` query commands need no external CLIs.

The destination path is resolved in this order:

- **`--path <file>`** — the full path to write the dump to (takes precedence).
- **`--dir <directory>` + `--file <name>`** — combined as `<dir>/<file>`.

If neither `--path` nor a `--dir`/`--file` pair is provided, the command fails fast with a clear error and a non-zero exit code. When the resolved path has no extension, one is appended based on the output format: `.dump` for custom (default), `.sql` for plain, `.tar` for tar — this lets `aux4/backup` pass an extension-less base path and leave artifact naming to the provider, while an explicit `--path dump.dump` is still written exactly as given.

**Dump options.** What goes into the dump is controlled by variables, so they can be set once in the `config.yaml` profile (and overridden per run on the command line):

| Option | Default | pg_dump flag |
|--------|---------|--------------|
| `--format` | `custom` | `--format` |
| `--clean` | `true` | `--clean` |
| `--noOwner` | `true` | `--no-owner` |
| `--noPrivileges` | `true` | `--no-privileges` |
| `--schemaOnly` | — | `--schema-only` |
| `--dataOnly` | — | `--data-only` |
| `--table` | — | `-t` |
| `--excludeTable` | — | `--exclude-table` |
| `--options` | — | appended verbatim |

The defaults are chosen so a backup is portable out of the box. `--no-owner` and `--no-privileges` let you restore into a database owned by a different user without ownership or privilege errors. `--clean` ensures a restore replaces existing objects cleanly.

If `pg_dump` fails, the partially written file is removed rather than left behind as a misleading artifact.

After writing the dump, `backup` prints a **result manifest** as a single line of JSON to stdout:

```json
{
  "path": "<resolved path>",
  "bytes": "<file size>",
  "checksum": "<sha256>",
  "status": "success",
  "format": "pg-dump-custom"
}
```

The `bytes` value is computed with `wc -c` and the `checksum` with `shasum -a 256` (falling back to `sha256sum`). The `format` value reflects the pg_dump format used: `pg-dump-custom`, `pg-dump-plain`, or `pg-dump-tar`. The manifest is consumed by the `aux4/backup` orchestrator to catalog each run.

**Binary resolution:** the command locates `pg_dump` automatically. It first checks `PATH` (`command -v pg_dump`), and if not found falls back to the Homebrew keg-only location (`$(brew --prefix libpq)/bin/pg_dump`). This means it works on Linux (where `pg_dump` is on `PATH`) and on macOS (where `libpq` is keg-only and not on `PATH`) **without** any manual `PATH` export. If the binary cannot be found anywhere, the command fails with an install hint.

**System dependency:** requires `pg_dump`. Install with `brew install libpq` (macOS) or `apt-get install postgresql-client` (Debian/Ubuntu).

#### Usage

```bash
aux4 db postgres backup --configFile config.yaml --config <profile> --path <file>
aux4 db postgres backup --configFile config.yaml --config <profile> --dir <directory> --file <name>
```

--host      Database host (default: localhost)
--port      Database port (default: 5432)
--database  Database name (default: postgres)
--user      Database user (default: postgres)
--password  Database password
--path      Full path to write the dump file (takes precedence over --dir/--file)
--dir       Directory to write the dump file (combined with --file)
--file      File name for the dump (combined with --dir)

Dump options: --format, --clean, --noOwner, --noPrivileges
              --schemaOnly, --dataOnly, --table, --excludeTable, --options

Configuration file — dump options live alongside the connection settings:

```yaml
config:
  prod:
    host: 127.0.0.1
    port: 5432
    database: myapp
    user: postgres
    password: secret
    format: custom
    clean: true
    noOwner: true
    noPrivileges: true
```

#### Example

```bash
aux4 db postgres backup --configFile config.yaml --config prod --path /tmp/myapp.dump
```

```text
{"path":"/tmp/myapp.dump","bytes":4096,"checksum":"a1b2c3d4e5f6...","status":"success","format":"pg-dump-custom"}
```
