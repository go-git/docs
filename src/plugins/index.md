---
layout: default
title: Plugins
---
# Plugins

`go-git` ships a lightweight, type-safe plugin registry (package `github.com/go-git/go-git/v6/x/plugin`) that lets callers replace internal implementations without modifying the core library.

Each plugin is identified by a typed **key** — a value that carries the Go type it manages at compile time. This means the compiler rejects any attempt to register an incompatible factory: a factory for type `A` cannot be stored under a key declared for type `B`.

## Lifecycle

The registry follows a strict, three-phase lifecycle:

1. **Register** — Provide a factory function early in program initialization, before any library code reads the plugin. The idiomatic place is an `init()` function or `main()` before calling any `go-git` APIs.
2. **Freeze** — The first call to `plugin.Get(key)` internally freezes that entry. Any subsequent `Register` call for the same key returns `plugin.ErrFrozen`.
3. **Get** — `plugin.Get(key)` invokes the registered factory and returns a fresh value of the plugin type.

```go
func init() {
    err := plugin.Register(plugin.ObjectSigner(), func() plugin.Signer {
        // return your signer implementation
        return mySigner
    })
    if err != nil {
        log.Fatal(err)
    }
}
```

## Available plugins

| Key | Plugin type | Description |
|-----|-------------|-------------|
| `plugin.ObjectSigner()` | `plugin.Signer` | Defines the default signer for Git objects (commits and tags). When set, go-git will use it to sign new tags or commits, based on the respective config entries `tag.gpgSign=true` and `commit.gpgSign=true`. |

## Errors

| Error | Returned by | Cause |
|-------|-------------|-------|
| `plugin.ErrFrozen` | `Register` | `Get` has already been called for that key |
| `plugin.ErrNotFound` | `Get` | No factory has been registered for the key |
| `plugin.ErrNilFactory` | `Register` | A `nil` factory was passed |

## Thread safety

`Register` and `Get` are safe for concurrent use. The freeze is an atomic transition: once `Get` marks a key as frozen, no concurrent `Register` can succeed for that key.
