#### Description

The `restore` command loads a dump produced by `aux4 db postgres backup` back into a PostgreSQL database. It is the counterpart of the `backup` provider command and uses the same connection and path resolution.

This command is contributed by the `aux4/db-postgres-backup` package, which extends the `db:postgres` profile owned by `aux4/db-postgres`. It is packaged separately so the `pg_dump`/`pg_restore`/`psql` system dependency is only required when backup/restore is actually used.

The command auto-detects the dump format:

- **Plain SQL** (produced by `--format plain`) — restored with `psql -f`.
- **Custom or tar** (produced by `--format custom` or `--format tar`) — restored with `pg_restore --clean --if-exists --no-owner --no-privileges`.

Connection details come from a `config.yaml` profile (via `--config`/`--configFile`). The dump path is resolved in this order:

- **`--path <file>`** — the full path to the dump to restore (takes precedence).
- **`--dir <directory>` + `--file <name>`** — combined as `<dir>/<file>`.

If neither `--path` nor a `--dir`/`--file` pair is provided, or the resolved file does not exist, the command fails fast with a clear error and a non-zero exit code.

On success `restore` prints a small outcome JSON to stdout:

```json
{
  "path": "<resolved path>",
  "status": "success",
  "action": "restore"
}
```

**Binary resolution:** the command locates `psql` and `pg_restore` automatically. It first checks `PATH` (`command -v psql` / `command -v pg_restore`), and if not found falls back to the Homebrew keg-only location (`$(brew --prefix libpq)/bin/...`). This means it works on Linux (where the binaries are on `PATH`) and on macOS (where `libpq` is keg-only and not on `PATH`) **without** any manual `PATH` export. If the required binary cannot be found, the command fails with an install hint.

**System dependency:** requires `psql` and `pg_restore`. Install with `brew install libpq` (macOS) or `apt-get install postgresql-client` (Debian/Ubuntu).

#### Usage

```bash
aux4 db postgres restore --configFile config.yaml --config <profile> --path <file>
aux4 db postgres restore --configFile config.yaml --config <profile> --dir <directory> --file <name>
```

--host      Database host (default: localhost)
--port      Database port (default: 5432)
--database  Database name (default: postgres)
--user      Database user (default: postgres)
--password  Database password
--path      Full path to the dump file to restore (takes precedence over --dir/--file)
--dir       Directory containing the dump file (combined with --file)
--file      File name of the dump (combined with --dir)

Configuration file:

```yaml
config:
  prod:
    host: 127.0.0.1
    port: 5432
    database: myapp
    user: postgres
    password: secret
```

#### Example

```bash
aux4 db postgres restore --configFile config.yaml --config prod --path /tmp/myapp.dump
```

```text
{"path":"/tmp/myapp.dump","status":"success","action":"restore"}
```
