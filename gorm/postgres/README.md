# `gorm/postgres`

PostgreSQL driver for the Sova [`gorm`](../core/README.md) port.
Wraps [`gorm.io/driver/postgres`](https://github.com/go-gorm/postgres),
which speaks the wire protocol through the `pgx` Go driver. No C
client library required.

## Install

```toml
[dependencies]
gorm            = { git = "https://github.com/sova-lang/ported-libraries", subdir = "gorm/core",     tag = "gorm-v0.1.0" }
"gorm/postgres" = { git = "https://github.com/sova-lang/ported-libraries", subdir = "gorm/postgres", tag = "gorm-postgres-v0.1.0" }
```

```bash
sova install
```

## Usage

```sova
package myapp on backend

import "gorm" using *
import "gorm/postgres"

func main() {
    let dsn = "postgres://app:secret@localhost:5432/app?sslmode=disable"
    let db = open(postgres.open(dsn))
    if db == none {
        println("could not connect to postgres")
        return
    }
    let _ = exec(db, "CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY, name TEXT NOT NULL)", [])
    let _ = exec(db, "INSERT INTO users (name) VALUES ($1)", ["Alice"])
    println("users: " + (count(table(db, "users")) as string))
    let _ = close(db)
}
```

## API

```sova
func open(dsn: string): any
func openWithConfig(config: any): any
```

`open(dsn)` accepts the two standard `pgx` connection-string shapes:

| DSN form                                                                        | Notes                                          |
| ------------------------------------------------------------------------------- | ---------------------------------------------- |
| `"host=localhost user=app password=secret dbname=app port=5432 sslmode=disable"` | Keyword/value form (libpq style).             |
| `"postgres://app:secret@localhost:5432/app?sslmode=disable"`                    | URI form.                                      |

`openWithConfig(config)` takes a map and is useful when you need to
toggle one of the postgres-specific knobs:

| Map key                  | Type   | What it controls                                                  |
| ------------------------ | ------ | ----------------------------------------------------------------- |
| `"dsn"`                  | string | The connection string (same content as for `open`).               |
| `"preferSimpleProtocol"` | bool   | Disable prepared-statement caching; needed for some pgbouncer modes. |
| `"withoutQuotingCheck"`  | bool   | Skip the quoting/identifier sanity pass at query-build time.     |

```sova
let cfg = {
    "dsn": "postgres://app:secret@localhost/app?sslmode=disable",
    "preferSimpleProtocol": true,
}
let db = open(postgres.openWithConfig(cfg))
```

## Placeholders

Postgres uses `$1`, `$2`, … rather than `?`. GORM's chain helpers
(`where`, `or`, `not`, ...) accept `?` placeholders and rewrite them
to `$N` for you, but **raw `exec`/`raw` strings must use `$N`
directly** because GORM doesn't rewrite the SQL you hand in.

```sova
let _ = exec(db, "INSERT INTO users (name, email) VALUES ($1, $2)", ["Alice", "a@example.com"])
```

## SSL

Production deployments should typically use `sslmode=verify-full`
plus an explicit `sslrootcert=/path/to/ca.crt`. The driver delegates
to `pgx` for the actual TLS negotiation, so any pgx-supported SSL
option works from the DSN.

## Version

The Sova port pins `gorm.io/driver/postgres@v1.5.11`. Bump by editing
`src/postgres.sova` and re-tagging the repository.
