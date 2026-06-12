# ported-libraries

Official Sova ports of third-party Go libraries that practical web
development reaches for. Each port lives in its own subdirectory with
its own `sova.toml`, so consumers can pull one library at a time
through the package manager's `subdir` selector.

## Packages

| Package          | Wraps                                                                             | Description                                            |
| ---------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `gorm`           | [`gorm.io/gorm`](https://gorm.io)                                                 | Core ORM: connections, CRUD, queries, transactions.    |
| `gorm/sqlite`    | [`github.com/glebarez/sqlite`](https://github.com/glebarez/sqlite)                | Pure-Go SQLite driver — no CGO toolchain required.     |
| `gorm/postgres`  | [`gorm.io/driver/postgres`](https://github.com/go-gorm/postgres)                  | Postgres driver built on `pgx`.                        |
| `redis`          | [`github.com/redis/go-redis/v9`](https://github.com/redis/go-redis)               | Redis client with key-value commands and Pub/Sub.      |

## Pulling a single library

Each subdirectory is independently versionable through the
`subdir` field on a git dependency:

```toml
[dependencies]
gorm           = { git = "https://github.com/sova-lang/ported-libraries", subdir = "gorm/core",     tag = "gorm-v0.1.0" }
"gorm/sqlite"  = { git = "https://github.com/sova-lang/ported-libraries", subdir = "gorm/sqlite",   tag = "gorm-sqlite-v0.1.0" }
redis          = { git = "https://github.com/sova-lang/ported-libraries", subdir = "redis",         tag = "redis-v0.1.0" }
```

For local development against the monorepo:

```toml
[dependencies]
gorm           = { path = "../ported-libraries/gorm/core" }
"gorm/sqlite"  = { path = "../ported-libraries/gorm/sqlite" }
redis          = { path = "../ported-libraries/redis" }
```

## How a port works

Each port is a thin Sova wrapper that declares the underlying Go
package + version through Sova's `extern` mechanism. The Sova
package manager passes the `@version` selector to `go mod tidy`
at compile time, so the right release lands in the build's
`go.mod` automatically — there is no manual Go-module bookkeeping
on the consumer side.

```sova
extern {
    func openDB(dsn: string): any = {
        backend("gorm.io/gorm@v1.25.12"): "..."
    }
}
```

Bumping the version in this repository is the only step needed to
update every consumer that pulls the port through git.

## Side

Every port in this repository targets `on backend`. The Go-ecosystem
packages we wrap have no JavaScript counterpart, so frontend / shared
forms are intentionally absent.
