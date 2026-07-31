# aux4/db-postgres-backup

Backup and restore provider commands for PostgreSQL. This package extends the
`db:postgres` profile (owned by `aux4/db-postgres`) with `backup` and `restore`
commands that wrap the `pg_dump`, `pg_restore`, and `psql` system CLIs.

It is packaged **separately** from `aux4/db-postgres` on purpose: the core
`aux4/db-postgres` query commands (`execute`, `stream`) use the Node driver and
need no external binaries. Only backup/restore need the PostgreSQL client CLIs,
so that system dependency is isolated here — install this package only when you
need dump/restore.

Because it declares the same `db:postgres` profile, its commands merge into the
existing profile: after installing both packages, `aux4 db postgres --help`
shows the query commands **and** `backup`/`restore` together.

## Installation

```bash
aux4 aux4 pkger install aux4/db-postgres-backup
```

This pulls in `aux4/db-postgres` as a dependency, so the full `db:postgres`
profile (query commands + backup/restore) is available.

## Configuration

Connection details are supplied through a `config.yaml` profile so credentials
stay out of any backup catalog:

```yaml
config:
  prod:
    host: 127.0.0.1
    port: 5432
    database: myapp
    user: postgres
    password: secret
```

Use it with `--configFile config.yaml --config prod`. Individual values can also
be passed directly with `--host`, `--port`, `--database`, `--user`,
`--password`, which take precedence over config values.

### Dump options

What goes into the dump is controlled by the same mechanism, so options can be
set once per profile and overridden per run on the command line:

```yaml
config:
  prod:
    host: 127.0.0.1
    database: myapp
    user: postgres
    password: secret
    format: custom
    clean: true
    noOwner: true
    noPrivileges: true
```

| Option | Default | Effect |
|--------|---------|--------|
| `format` | `custom` | Output format passed to `pg_dump --format` (`custom`, `plain`, or `tar`) |
| `clean` | `true` | Include DROP statements before CREATE |
| `noOwner` | `true` | Omit ownership commands |
| `noPrivileges` | `true` | Omit privilege/grant commands |
| `schemaOnly` | — | Dump only schema, no data |
| `dataOnly` | — | Dump only data, no schema |
| `table` | — | Dump only this table (`pg_dump -t`) |
| `excludeTable` | — | Exclude a table from the dump |
| `options` | — | Additional raw `pg_dump` options, appended verbatim |

The defaults are chosen so a backup is portable out of the box. `--no-owner`
and `--no-privileges` let you restore into a database owned by a different user
without ownership or privilege errors. `--clean` ensures a restore replaces
existing objects cleanly.

## backup

Creates a dump of a PostgreSQL database with `pg_dump`.

The destination path is resolved as:

- **`--path <file>`** — full path to write the dump (takes precedence).
- **`--dir <directory>` + `--file <name>`** — combined as `<dir>/<file>`.

If neither is given, the command fails fast. When the resolved path has no
extension, one is appended based on the output format: `.dump` for custom
(default), `.sql` for plain, `.tar` for tar — so `aux4/backup` can pass an
extension-less base path and let this package name the artifact for its own
format; an explicit `--path dump.sql` is written exactly as given. If `pg_dump`
fails, the partial file is removed rather than left behind as a misleading
artifact.

On success it prints a result manifest as a single line of JSON to stdout:

```bash
aux4 db postgres backup --configFile config.yaml --config prod --path /tmp/myapp.dump
```

```text
{"path":"/tmp/myapp.dump","bytes":4096,"checksum":"a1b2c3...","status":"success","format":"pg-dump-custom"}
```

The manifest (`path`, `bytes`, `checksum`, `status`, `format`) is consumed by
the `aux4/backup` orchestrator to catalog each run. The `format` value reflects
the pg_dump format used: `pg-dump-custom`, `pg-dump-plain`, or `pg-dump-tar`.

## restore

Loads a dump produced by `backup` back into a PostgreSQL database. The command
auto-detects whether the file is plain SQL or a binary format (custom/tar):

- **Plain SQL** — restored with `psql -f`.
- **Custom or tar** — restored with `pg_restore --clean --if-exists --no-owner --no-privileges`.

```bash
aux4 db postgres restore --configFile config.yaml --config prod --path /tmp/myapp.dump
```

```text
{"path":"/tmp/myapp.dump","status":"success","action":"restore"}
```

Path resolution (`--path` or `--dir` + `--file`) is identical to `backup`. If
the resolved file does not exist, the command fails fast.

## Binary resolution

The commands locate the `pg_dump`/`pg_restore`/`psql` binaries automatically:

1. Check `PATH` first (`command -v pg_dump` / `command -v psql` / `command -v pg_restore`).
2. Fall back to the Homebrew keg-only location
   (`$(brew --prefix libpq)/bin/...`).

This works on Linux (binaries on `PATH`) **and** on macOS (where `libpq` is
keg-only and not on `PATH`) with **no manual `PATH` export**. If a binary
cannot be found anywhere, the command fails with an install hint.

## System dependency

Requires the PostgreSQL client tools (`pg_dump`, `pg_restore`, and `psql`):

- macOS: `brew install libpq`
- Debian/Ubuntu: `apt-get install postgresql-client`
