# db postgres backup and restore

Provider commands (contributed by `aux4/db-postgres-backup`) that wrap
`pg_dump`/`pg_restore`/`psql`. Connection is supplied via `--config` (a
`config.yaml` profile) and the destination via `--path` (or `--dir` + `--file`).
`backup` prints a result manifest to stdout; `restore` loads the dump back into
the database. The `pg_dump`/`pg_restore`/`psql` binaries are resolved
automatically (PATH or the Homebrew keg-only location) — no manual `PATH`
export is required.

```file:config.yaml
config:
  test:
    host: 127.0.0.1
    port: 5432
    database: bkptest
    user: postgres
    password: mysecretpassword
  plainformat:
    host: 127.0.0.1
    port: 5432
    database: bkptest
    user: postgres
    password: mysecretpassword
    format: plain
```

```beforeAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --user postgres --password mysecretpassword --query "DROP DATABASE IF EXISTS bkptest"
```

```beforeAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --user postgres --password mysecretpassword --query "CREATE DATABASE bkptest"
```

```beforeAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --database bkptest --user postgres --password mysecretpassword --query "CREATE TABLE items (id INT PRIMARY KEY, name TEXT, qty INTEGER)"
```

```beforeAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --database bkptest --user postgres --password mysecretpassword --query "INSERT INTO items VALUES (1, 'apple', 10), (2, 'pear', 20), (3, 'plum', 30)"
```


```afterAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --user postgres --password mysecretpassword --query "DROP DATABASE IF EXISTS bkptest"
```

```afterAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --user postgres --password mysecretpassword --query "DROP DATABASE IF EXISTS bkptest_restore"
```

```afterAll
rm -f /tmp/aux4-pgbkp-test.dump /tmp/aux4-pgbkp-dirfile.dump /tmp/aux4-pgbkp-plain.sql /tmp/aux4-pgbkp-clioverride.dump /tmp/aux4-pgbkp-schemaonly.dump /tmp/aux4-pgbkp-failed.dump /tmp/aux4-pgbkp-autoext.dump
```

## backup with --config and --path (custom format)

### should write the dump and print a manifest

```execute
aux4 db postgres backup --configFile config.yaml --config test --path /tmp/aux4-pgbkp-test.dump
```

```expect:regex
\{"path":"/tmp/aux4-pgbkp-test\.dump","bytes":\d+,"checksum":"[a-f0-9]{64}","status":"success","format":"pg-dump-custom"\}
```

### should create a non-empty dump file

```execute
test -s /tmp/aux4-pgbkp-test.dump && echo present
```

```expect
present
```

## backup with --dir and --file

### should resolve the path from dir + file

```execute
aux4 db postgres backup --configFile config.yaml --config test --dir /tmp --file aux4-pgbkp-dirfile.dump
```

```expect:regex
\{"path":"/tmp/aux4-pgbkp-dirfile\.dump","bytes":\d+,"checksum":"[a-f0-9]{64}","status":"success","format":"pg-dump-custom"\}
```

## backup with plain format

### should produce a plain SQL dump

```execute
aux4 db postgres backup --configFile config.yaml --config plainformat --path /tmp/aux4-pgbkp-plain.sql
```

```expect:regex
\{"path":"/tmp/aux4-pgbkp-plain\.sql","bytes":\d+,"checksum":"[a-f0-9]{64}","status":"success","format":"pg-dump-plain"\}
```

### should contain SQL statements

```execute
grep -c 'CREATE TABLE' /tmp/aux4-pgbkp-plain.sql
```

```expect
1
```

## dump options

### should let a CLI flag override the config profile

```execute
aux4 db postgres backup --configFile config.yaml --config test --noOwner false --path /tmp/aux4-pgbkp-clioverride.dump >/dev/null
test -s /tmp/aux4-pgbkp-clioverride.dump && echo present
```

```expect
present
```

### should dump schema only when requested

```execute
aux4 db postgres backup --configFile config.yaml --config test --schemaOnly true --path /tmp/aux4-pgbkp-schemaonly.dump
```

```expect:regex
\{"path":"/tmp/aux4-pgbkp-schemaonly\.dump","bytes":\d+,"checksum":"[a-f0-9]{64}","status":"success","format":"pg-dump-custom"\}
```

## auto-append extension

### should append .dump when path has no extension

```execute
aux4 db postgres backup --configFile config.yaml --config test --path /tmp/aux4-pgbkp-autoext
```

```expect:regex
\{"path":"/tmp/aux4-pgbkp-autoext\.dump","bytes":\d+,"checksum":"[a-f0-9]{64}","status":"success","format":"pg-dump-custom"\}
```

## backup failure

### should not leave a partial dump file behind

```execute
aux4 db postgres backup --configFile config.yaml --config test --password WRONGPASSWORD --path /tmp/aux4-pgbkp-failed.dump 2>/dev/null
test -e /tmp/aux4-pgbkp-failed.dump && echo "leftover" || echo "cleaned up"
```

```expect
cleaned up
```

## backup with no path

### should fail fast when neither path nor dir/file is given

```execute
aux4 db postgres backup --configFile config.yaml --config test
```

```error:partial
Error: provide --path, or --dir and --file
```

## restore from custom format

```beforeAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --database bkptest --user postgres --password mysecretpassword --query "DROP TABLE items"
```

### should restore the dump and print an outcome

```execute
aux4 db postgres restore --configFile config.yaml --config test --path /tmp/aux4-pgbkp-test.dump
```

```expect
{"path":"/tmp/aux4-pgbkp-test.dump","status":"success","action":"restore"}
```

### should bring the rows back

```execute
aux4 db postgres execute --host 127.0.0.1 --port 5432 --database bkptest --user postgres --password mysecretpassword --query "SELECT * FROM items ORDER BY id" | jq -c .
```

```expect
[{"id":1,"name":"apple","qty":10},{"id":2,"name":"pear","qty":20},{"id":3,"name":"plum","qty":30}]
```

## restore from plain SQL format

```beforeAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --database bkptest --user postgres --password mysecretpassword --query "DROP TABLE items"
```

### should restore the plain SQL dump

```execute
aux4 db postgres restore --configFile config.yaml --config test --path /tmp/aux4-pgbkp-plain.sql
```

```expect
{"path":"/tmp/aux4-pgbkp-plain.sql","status":"success","action":"restore"}
```

### should bring the rows back from the plain SQL restore

```execute
aux4 db postgres execute --host 127.0.0.1 --port 5432 --database bkptest --user postgres --password mysecretpassword --query "SELECT * FROM items ORDER BY id" | jq -c .
```

```expect
[{"id":1,"name":"apple","qty":10},{"id":2,"name":"pear","qty":20},{"id":3,"name":"plum","qty":30}]
```

## restore into a fresh database

```beforeAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --user postgres --password mysecretpassword --query "DROP DATABASE IF EXISTS bkptest_restore"
```

```beforeAll
aux4 db postgres execute --host 127.0.0.1 --port 5432 --user postgres --password mysecretpassword --query "CREATE DATABASE bkptest_restore"
```

### should restore the dump into a different database

```execute
aux4 db postgres restore --configFile config.yaml --config test --database bkptest_restore --path /tmp/aux4-pgbkp-test.dump
```

```expect
{"path":"/tmp/aux4-pgbkp-test.dump","status":"success","action":"restore"}
```

### should have the rows in the fresh database

```execute
aux4 db postgres execute --host 127.0.0.1 --port 5432 --database bkptest_restore --user postgres --password mysecretpassword --query "SELECT * FROM items ORDER BY id" | jq -c .
```

```expect
[{"id":1,"name":"apple","qty":10},{"id":2,"name":"pear","qty":20},{"id":3,"name":"plum","qty":30}]
```

## restore with a missing file

### should fail fast when the dump file does not exist

```execute
aux4 db postgres restore --configFile config.yaml --config test --path /tmp/does-not-exist.dump
```

```error:partial
Error: backup file not found: /tmp/does-not-exist.dump
```
