# `redis`

Sova port of [`github.com/redis/go-redis/v9`](https://github.com/redis/go-redis) —
connection, key-value commands, atomic counters, and Pub/Sub.
Backend-only.

## Install

```toml
[dependencies]
redis = { git = "https://github.com/sova-lang/ported-libraries", subdir = "redis", tag = "redis-v0.1.0" }
```

```bash
sova install
```

## Quick start

```sova
package myapp on backend

import "redis" using *

func main() {
    let client = connect("redis://localhost:6379/0")
    if client == none {
        println("could not parse url")
        return
    }
    if !ping(client) {
        println("server unreachable")
        let _ = close(client)
        return
    }

    let _ = set(client, "hello", "world", 60)
    println("hello = " + get(client, "hello"))

    let hits = incr(client, "page-views")
    println("page-views = " + (hits as string))

    let _ = close(client)
}
```

## Connection

| Function                                  | Returns | Description                                                                |
| ----------------------------------------- | ------- | -------------------------------------------------------------------------- |
| `connect(url)`                            | `any`   | Open a client from a `redis://[:password@]host:port/db` URL. `none` on parse error. |
| `connectAddr(addr, password, db)`         | `any`   | Open with explicit components when a URL is awkward.                        |
| `close(client)`                           | `bool`  | Release the connection pool.                                                |
| `ping(client)`                            | `bool`  | Round-trip a `PING`; `true` when the server answered `PONG`.                |

The client opens lazily — `connect` only validates the URL; the first
real network call (typically `ping`) is what actually dials.

## Key-value commands

| Function                                       | Returns  | Description                                                                |
| ---------------------------------------------- | -------- | -------------------------------------------------------------------------- |
| `set(client, key, value, ttlSeconds)`          | `bool`   | Set a string value; `ttlSeconds = 0` means "never expires".                |
| `setNX(client, key, value, ttlSeconds)`        | `bool`   | Set only if the key does not yet exist (atomic).                            |
| `get(client, key)`                             | `string` | Read a key; `""` when missing or on error (use `exists` to disambiguate).  |
| `del(client, keys)`                            | `int`    | Delete one or more keys; returns the count removed.                        |
| `exists(client, key)`                          | `bool`   | Report whether a key is present.                                            |
| `expire(client, key, ttlSeconds)`              | `bool`   | Apply a TTL to an existing key.                                             |
| `ttl(client, key)`                             | `int`    | Remaining TTL in seconds; `-1` = no expiry, `-2` = missing.                |
| `keys(client, pattern)`                        | `[]any`  | List matching keys (glob-style). Walks the keyspace synchronously — avoid on large dbs. |
| `flushDB(client)`                              | `bool`   | Wipe the current logical database. Tests only.                              |

## Atomic counters

| Function                                  | Returns | Description                                                |
| ----------------------------------------- | ------- | ---------------------------------------------------------- |
| `incr(client, key)`                       | `int`   | Increment by 1; creates the key set to `0` if missing.     |
| `incrBy(client, key, delta)`              | `int`   | Increment by `delta`.                                      |
| `decr(client, key)`                       | `int`   | Decrement by 1.                                            |

All three return the new value after the operation, or `-1` on error.

## Cache pattern

A typical `get-or-load` helper composes on top of the primitives:

```sova
func cacheGet(client: any, key: string, ttlSeconds: int, loader: func(): string): string {
    let cached = get(client, key)
    if cached != "" {
        return cached
    }
    let fresh = loader()
    let _ = set(client, key, fresh, ttlSeconds)
    return fresh
}
```

`get` returns the empty string both when the key is missing and on
errors — for sensitive cases (where `""` is a legitimate cached
value) use `exists` first.

## Pub/Sub

```sova
let sub = subscribe(client, ["news.crypto", "news.equities"], func(channel: string, payload: string) {
    println("[" + channel + "] " + payload)
})

// later:
let _ = publish(client, "news.crypto", "BTC up 3%")

// at teardown:
let _ = unsubscribe(sub)
```

| Function                                                        | Returns | Description                                                                   |
| --------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------- |
| `publish(client, channel, message)`                             | `int`   | Publish a message; returns the number of receiving subscribers.               |
| `subscribe(client, channels, handler)`                          | `any`   | Subscribe to exact channel names. Handler runs on a dedicated goroutine.       |
| `psubscribe(client, patterns, handler)`                         | `any`   | Subscribe by glob pattern (e.g. `"news.*"`); handler receives the matched channel name. |
| `unsubscribe(subscription)`                                     | `bool`  | Close a subscription handle and stop its consumer goroutine.                  |

The handler executes on a separate goroutine — it must not block on
state that the main routine holds (no recursive `sub.publish` from
inside the handler without a `go` launch). For ordered, low-volume
streams a small channel between handler and main goroutine is the
usual pattern.

## Version

The Sova port pins `github.com/redis/go-redis/v9@v9.7.0`. Bump by
editing `src/*.sova` and re-tagging the repository.
