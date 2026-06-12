# `gorm/sqlite`

Pure-Go SQLite driver for the Sova [`gorm`](../core/README.md) port.
Wraps [`github.com/glebarez/sqlite`](https://github.com/glebarez/sqlite),
which uses `modernc.org/sqlite` internally — no CGO toolchain or
external `libsqlite3` is required, the database engine compiles into
your binary as pure Go.

## Install

```toml
[dependencies]
gorm          = { git = "https://github.com/sova-lang/ported-libraries", subdir = "gorm/core",   tag = "gorm-v0.1.0" }
"gorm/sqlite" = { git = "https://github.com/sova-lang/ported-libraries", subdir = "gorm/sqlite", tag = "gorm-sqlite-v0.1.0" }
```

```bash
sova install
```

## Usage

```sova
package myapp on backend

import "gorm" using *
import "gorm/sqlite"

func main() {
    let db = open(sqlite.open("app.db"))
    if db == none {
        println("could not open app.db")
        return
    }
    let _ = exec(db, "CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, body TEXT)", [])
    let _ = exec(db, "INSERT INTO notes (body) VALUES (?)", ["first note"])
    println("notes: " + (count(table(db, "notes")) as string))
    let _ = close(db)
}
```

The package exposes a single function:

```sova
func open(dsn: string): any
```

`dsn` is either a filesystem path or one of SQLite's URI forms:

| DSN                                            | Meaning                                       |
| ---------------------------------------------- | --------------------------------------------- |
| `"app.db"`                                     | File on disk; created if missing.             |
| `":memory:"`                                   | Anonymous in-memory database.                 |
| `"file:cache?mode=memory&cache=shared"`        | Named in-memory database (sharable per-process). |
| `"file:app.db?_pragma=foreign_keys(1)"`        | URI form with pragmas applied at open.        |

The returned dialector is passed straight to `gorm.open(...)`:

```sova
let dialector = sqlite.open(":memory:")
let db = open(dialector)
```

## Why pure Go

- **No C toolchain** — `sova build` cross-compiles without needing
  `gcc`, MSVC, or a C SQLite installation.
- **CGO disabled by default** — the Sova build sets `CGO_ENABLED=0`,
  which a CGO-based SQLite driver would refuse to build under.
- **Single binary, anywhere** — the generated executable embeds the
  SQLite engine; no shared-library hunt at runtime.

The trade-off is throughput: `modernc.org/sqlite` is a translated
pure-Go SQLite that is roughly 30–50 % slower than the C original on
tight write-heavy benchmarks. For application workloads this is
typically unnoticeable.

## Version

The Sova port pins `github.com/glebarez/sqlite@v1.11.0`. Bump by
editing `src/sqlite.sova` and re-tagging the repository.
