# `gorm`

Sova port of [`gorm.io/gorm`](https://gorm.io) — connection
management, CRUD, query chains, raw SQL, and transactions.
Backend-only; pair with one of the driver ports
([`gorm/sqlite`](../sqlite/README.md), [`gorm/postgres`](../postgres/README.md))
to actually talk to a database.

## Install

```toml
[dependencies]
gorm          = { git = "https://github.com/sova-lang/ported-libraries", subdir = "gorm/core",   tag = "gorm-v0.1.0" }
"gorm/sqlite" = { git = "https://github.com/sova-lang/ported-libraries", subdir = "gorm/sqlite", tag = "gorm-sqlite-v0.1.0" }
```

```bash
sova install
```

## Quick start

```sova
package myapp on backend

import "gorm" using *
import "gorm/sqlite"

func main() {
    let db = open(sqlite.open("app.db"))
    if db == none {
        println("could not open database")
        return
    }

    let _ = exec(db, "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT, age INTEGER)", [])
    let _ = exec(db, "INSERT INTO users (name, age) VALUES (?, ?)", ["Alice", 30])
    let _ = exec(db, "INSERT INTO users (name, age) VALUES (?, ?)", ["Bob",   42])

    let adults = count(where(table(db, "users"), "age >= ?", [18]))
    println("adults: " + (adults as string))

    let _ = close(db)
}
```

## Connection

| Function                                  | Returns | Description                                                                 |
| ----------------------------------------- | ------- | --------------------------------------------------------------------------- |
| `open(dialector)`                         | `any`   | Open a database. Returns `none` on failure.                                 |
| `openWithConfig(dialector, config)`       | `any`   | Open with a custom config map (see `newConfig`).                            |
| `close(db)`                               | `bool`  | Close the underlying `*sql.DB`.                                             |
| `ping(db)`                                | `bool`  | Verify the connection with a round-trip query.                              |
| `lastError(db)`                           | `string` | Last error message attached to the GORM session; `""` on success.          |
| `rowsAffected(db)`                        | `int`   | Rows affected by the last terminal call.                                    |
| `newConfig()`                             | `any`   | Build an empty config map for `openWithConfig`.                             |

Config keys: `"skipDefaultTransaction"`, `"prepareStmt"`, `"dryRun"` (all bool).

## Migrations

| Function                       | Returns | Description                                          |
| ------------------------------ | ------- | ---------------------------------------------------- |
| `autoMigrate(db, models)`      | `bool`  | Create/update tables to match the model structs.     |
| `dropTable(db, models)`        | `bool`  | Drop tables for the listed models / table names.     |
| `hasTable(db, model)`          | `bool`  | Report whether a table exists.                       |

Models are Go-side structs registered via an `extern { type … }`
binding. The migrator reflects on them at runtime to derive column
shapes, primary keys, and indices.

## CRUD

| Function                                   | Returns | Description                                                         |
| ------------------------------------------ | ------- | ------------------------------------------------------------------- |
| `create(db, value)`                        | `any`   | Insert one record; primary-key field is populated on the value.     |
| `createBatch(db, values, batchSize)`       | `any`   | Insert many records in batched statements.                          |
| `save(db, value)`                          | `any`   | Upsert: INSERT when primary key is zero, otherwise UPDATE all fields. |
| `updates(db, values)`                      | `any`   | Update non-zero fields of a struct, or every key of a map.          |
| `update(db, column, value)`                | `any`   | Update a single column on the matched rows.                         |
| `deleteRecord(db, value)`                  | `any`   | Delete matched rows; pass the model so GORM knows the table.        |
| `first(db, out)`                           | `any`   | Fetch the first matching row into `out` (pointer).                  |
| `firstByID(db, out, id)`                   | `any`   | Fetch by primary-key value.                                         |
| `last(db, out)`                            | `any`   | Fetch the last matching row (PK desc).                              |
| `find(db, out)`                            | `any`   | Fetch all matching rows into a slice pointer.                       |
| `count(db)`                                | `int`   | Count matching rows.                                                |

Every function returns the chained `db` handle on the Sova side
(`any`-typed), so you can keep building the query or check
`lastError(db)` / `rowsAffected(db)` after.

## Query chains

```sova
let adults = find(
    limit(
        order(
            where(table(db, "users"), "age >= ?", [18]),
            "name asc",
        ),
        50,
    ),
    out,
)
```

| Function                                  | Returns | Description                                       |
| ----------------------------------------- | ------- | ------------------------------------------------- |
| `where(db, query, args)`                  | `any`   | Add a `WHERE` clause.                             |
| `not(db, query, args)`                    | `any`   | Add a `NOT` clause.                               |
| `or(db, query, args)`                     | `any`   | Add an `OR` clause.                               |
| `order(db, value)`                        | `any`   | Append an `ORDER BY` fragment.                    |
| `groupBy(db, name)`                       | `any`   | Append a `GROUP BY` (`group` is a Sova keyword).  |
| `having(db, query, args)`                 | `any`   | Append a `HAVING` clause.                         |
| `limit(db, n)`                            | `any`   | Cap result-set size.                              |
| `offset(db, n)`                           | `any`   | Skip the first `n` rows.                          |
| `selectColumns(db, columns)`              | `any`   | Restrict the columns returned (`select` reserved). |
| `preload(db, association)`                | `any`   | Eager-load an associated relation.                |
| `joins(db, query)`                        | `any`   | Add an explicit `JOIN` fragment.                  |
| `table(db, name)`                         | `any`   | Pin a table for the next terminal call.           |
| `distinct(db, column)`                    | `any`   | Mark a distinct select.                           |

Bind chain steps to local variables when you want to keep the original
`db` reusable; calling these functions never mutates the caller's
handle, they return a new one.

## Raw SQL

| Function                                  | Returns | Description                                                         |
| ----------------------------------------- | ------- | ------------------------------------------------------------------- |
| `exec(db, sql, args)`                     | `any`   | Run a non-query (INSERT/UPDATE/DELETE/DDL). Check `rowsAffected(db)`. |
| `raw(db, sql, args)`                      | `any`   | Stage a raw SELECT for the next `find` / `first` / `scan`.           |
| `scan(db, out)`                           | `any`   | Decode the staged row(s) into `out`.                                |

```sova
let q = raw(db, "SELECT name, age FROM users WHERE age >= ?", [18])
let _ = scan(q, results)
```

## Transactions

```sova
let ok = transaction(db, func(tx: any): bool {
    let a = exec(tx, "UPDATE accounts SET balance = balance - ? WHERE id = ?", [100, 1])
    let b = exec(tx, "UPDATE accounts SET balance = balance + ? WHERE id = ?", [100, 2])
    return lastError(a) == "" && lastError(b) == ""
})
```

Return `false` (or panic) from the callback to roll back, `true` to
commit. Nested calls participate in the outer transaction through
SAVEPOINTs.

| Function                       | Returns | Description                                          |
| ------------------------------ | ------- | ---------------------------------------------------- |
| `transaction(db, fn)`          | `bool`  | Run `fn` in a transaction; commit on `true`, roll back on `false`. |
| `begin(db)`                    | `any`   | Start a manual transaction.                          |
| `commit(tx)`                   | `bool`  | Commit a manual transaction.                         |
| `rollback(tx)`                 | `bool`  | Roll back a manual transaction.                      |
| `savepoint(tx, name)`          | `bool`  | Create a savepoint inside a transaction.             |
| `rollbackTo(tx, name)`         | `bool`  | Roll back to a previously-created savepoint.         |

## Errors

GORM attaches errors to the session handle rather than throwing.
After any terminal operation, check `lastError(db)`:

```sova
let _ = first(db, user)
let err = lastError(db)
if err != "" {
    if err == "record not found" {
        println("nothing matched")
    } else {
        println("query failed: " + err)
    }
}
```

## Models

Define models on the Go side and bind them in via an `extern` type
declaration. Example: in `models.go.tpl` you ship next to your Sova
sources, write a Go-format type with GORM struct tags; expose it in
Sova through an extern type binding. The migrator and the CRUD calls
all accept the resulting opaque `any`-typed value.

## Version

The Sova port pins `gorm.io/gorm@v1.25.12`. Bump by editing
`src/*.sova` and re-tagging the repository. The Sova package manager
passes the pin to `go mod tidy` automatically when this dependency
is installed.
