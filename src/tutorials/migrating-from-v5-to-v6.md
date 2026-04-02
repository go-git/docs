---
layout: default
title: Migrating from v5 to v6
---
# Migrating from v5 to v6

> **Note:** This documentation targets the upcoming v6 release of go-git and is subject to change until v6 is officially released. Items are labelled with their current status:
>
> - ✅ **Merged** — already in the `main` branch
> - 🔲 **Planned** — accepted but not yet merged
> - 💬 **In Discussion** — under active discussion; details may change

The v6 release of go-git is a major release that introduces significant API
and behaviour changes. This guide covers everything you need to update when
upgrading from v5.

The canonical tracking issue for all breaking and behaviour changes is
[go-git/go-git#910](https://github.com/go-git/go-git/issues/910).

---

## Quick migration checklist

Use this checklist to audit your codebase before upgrading:

- [ ] Update the module import path from `github.com/go-git/go-git/v5` to `github.com/go-git/go-git/v6`
- [ ] Call `defer r.Close()` on every `*git.Repository` obtained from filesystem-backed operations
- [ ] Remove the `isBare bool` positional argument from `git.PlainClone` calls; set `CloneOptions.Bare` instead
- [ ] Update any explicit `git.AllTags` / tag-fetch logic — tags are **no longer fetched by default** on `Fetch`
- [ ] Replace `plumbing.TagMode` references with `git.TagMode`
- [ ] Rename `config.Version_0` / `config.Version_1` constants to `config.Version0` / `config.Version1`
- [ ] Update code that implements or embeds `commitgraph.Index` — the interface gained `io.Closer` and new methods
- [ ] Review any code that constructs `osfs` instances for `Plain*` operations — `BoundOS` is now the default
- [ ] Remove any validation that requires `refs/` prefix on `branch.*.merge` config values

---

## Breaking changes

### 1. `Repository.Close()` must be called ✅ Merged

**What changed:** A `Close() error` method has been added to `*git.Repository`. It
releases file handles held by the underlying filesystem storage (e.g. open
pack index `.idx` files). For in-memory repositories `Close` is a no-op.

**Why:** On Windows, open file handles prevented `os.RemoveAll` from
cleaning up temporary directories, causing intermittent CI failures. The
broader fix is to give callers a standard way to release these resources.

**Impact:** Any code that opens a filesystem-backed repository and does not
call `Close` will leak file descriptors. This is especially noticeable on
Windows but is a correctness issue on all platforms.

**How to migrate:** Add `defer r.Close()` wherever you obtain a `*git.Repository`:

```go
// v5
r, err := git.PlainOpen("/path/to/repo")
if err != nil { ... }
// no cleanup

// v6
r, err := git.PlainOpen("/path/to/repo")
if err != nil { ... }
defer r.Close()
```

The same applies to repositories returned by `Submodule.Repository()`:

```go
sub, err := w.Submodule("mymodule")
if err != nil { ... }
subrepo, err := sub.Repository()
if err != nil { ... }
defer subrepo.Close()
```

**References:** [PR #1906](https://github.com/go-git/go-git/pull/1906)

---

### 2. `git.PlainClone` — `isBare` parameter removed ✅ Merged

**What changed:** The positional `isBare bool` parameter has been removed
from `git.PlainClone` and `git.PlainCloneContext`. A `Bare bool` field has
been added to `git.CloneOptions` instead.

**Why:** Consolidates all clone configuration into a single options struct,
making the API more extensible and consistent with `git.Clone`.

**Impact:** Any call to `git.PlainClone` or `git.PlainCloneContext` that
passes `isBare` as a positional argument will not compile.

**How to migrate:**

```go
// v5
r, err := git.PlainClone("/path/to/repo", true, &git.CloneOptions{
    URL: "https://github.com/go-git/go-git",
})

// v6
r, err := git.PlainClone("/path/to/repo", &git.CloneOptions{
    URL:  "https://github.com/go-git/go-git",
    Bare: true,
})
```

**References:** [go-git/go-git#910](https://github.com/go-git/go-git/issues/910)

---

### 3. Tags are no longer fetched by default on `Fetch` ✅ Merged

**What changed:** `git.FetchOptions` previously defaulted to fetching all
tags (`AllTags`). In v6 the default is **no tags**. This aligns go-git's
behaviour with the `git fetch` CLI, which only auto-follows tags
reachable from fetched commits when no explicit tag refspec is given.

**Why:** When comparing actual `git fetch` behaviour with the test
expectations in go-git, it became clear that the tests expected all tags to
be pulled without the user explicitly requesting them.

**Impact:** Code that relied on tags being silently included in every
`Fetch` call will no longer receive them.

**How to migrate:** If you need all remote tags, set `Tags: git.AllTags`
explicitly:

```go
// v5 — tags were returned by default
err := r.Fetch(&git.FetchOptions{
    RemoteName: "origin",
})

// v6 — must opt in to fetching all tags
err := r.Fetch(&git.FetchOptions{
    RemoteName: "origin",
    Tags:       git.AllTags,
})
```

**References:** [PR #1459](https://github.com/go-git/go-git/pull/1459)

---

### 4. `plumbing.TagMode` moved to `git.TagMode` ✅ Merged

**What changed:** The `TagMode` type and its constants (`AllTags`,
`NoTags`, `InvalidTagMode`) have been moved from the `plumbing` package
to the top-level `git` package.

**Why:** `TagMode` is a concept that belongs to high-level clone/fetch
options rather than the low-level plumbing layer.

**Impact:** Any explicit reference to `plumbing.TagMode`, `plumbing.AllTags`,
or `plumbing.NoTags` will not compile.

**How to migrate:**

```go
// v5
import "github.com/go-git/go-git/v5/plumbing"

opts := &git.CloneOptions{
    Tags: plumbing.AllTags,
}

// v6
opts := &git.CloneOptions{
    Tags: git.AllTags,
}
```

**References:** [go-git/go-git#910](https://github.com/go-git/go-git/issues/910)

---

### 5. `config.Version_X` constants renamed to `config.VersionX` ✅ Merged

**What changed:** The exported constants in `plumbing/format/config` were
renamed to comply with Go naming conventions enforced by the `revive`
linter:

| v5 name | v6 name |
|---------|---------|
| `config.Version_0` | `config.Version0` |
| `config.Version_1` | `config.Version1` |

**Why:** Go convention is to avoid underscores in exported names. The
rename was part of a broader linter-driven cleanup
([PR #1771](https://github.com/go-git/go-git/pull/1771)).

**Impact:** Any code that references the old underscore-separated constant
names will not compile.

**How to migrate:** Replace all occurrences:

```go
// v5
cfg.Core.RepositoryFormatVersion = config.Version_0

// v6
cfg.Core.RepositoryFormatVersion = config.Version0
```

**References:** [PR #1771](https://github.com/go-git/go-git/pull/1771)

---

### 6. `commitgraph.Index` interface extended ✅ Merged

**What changed:** The `Index` interface in `plumbing/format/commitgraph`
was extended as part of commit-graph chain support:

- `io.Closer` was added — callers must now call `Close()` when done with an index
- `GetCommitDataByIndex(i uint32) (*CommitData, error)` replaced the old node-based method
- `HasGenerationV2() bool` and `MaximumNumberOfHashes() uint32` were added

**Why:** Adding commit-graph chain (multi-file) support required the
interface to carry more capability and to properly manage open file handles
([PR #854](https://github.com/go-git/go-git/pull/854)).

**Impact:** Any type that implements `commitgraph.Index` must add the new
methods. Code that uses the interface needs to handle the `Close()` call.

**How to migrate:** Add the missing method implementations to any custom
`Index` type and add `defer index.Close()` where you obtain an index:

```go
// v6 — close the index when done
idx, err := commitgraph.OpenFileIndex(f)
if err != nil { ... }
defer idx.Close()
```

**References:** [PR #854](https://github.com/go-git/go-git/pull/854)

---

### 7. `osfs.BoundOS` is now the default for `Plain*` operations ✅ Merged

**What changed:** All `PlainInit`, `PlainClone`, `PlainOpen`, and related
functions now create their `osfs` instances using `osfs.WithBoundOS()`,
which restricts filesystem access to within the repository root and prevents
path-traversal outside it.

**Why:** The unbounded OS filesystem could be exploited via crafted
repository paths to read or write files outside the intended directory.
`BoundOS` provides a security boundary.

**Impact:** Code that relied on symlinks or relative paths that escape the
repository root will now get an error. Custom `osfs` instances passed to
lower-level APIs are not affected.

**How to migrate:** Ensure paths used within repository operations stay
within the repository root. If you need unrestricted filesystem access for
a specific use-case, construct the filesystem explicitly:

```go
// v6 — unrestricted access (use with caution)
import "github.com/go-git/go-billy/v6/osfs"

fs := osfs.New("/path/to/repo") // no BoundOS
```

**References:** [go-git/go-git#910](https://github.com/go-git/go-git/issues/910)

---

### 8. Config validation for `branch.*.merge` relaxed ✅ Merged

**What changed:** The git config parser previously rejected
`branch.<name>.merge` values that did not start with `refs/`. This
validation has been removed to match the behaviour of the `git` CLI, which
stores the value as-is without a prefix check.

**Why:** The `git` documentation describes `branch.<name>.merge` as
being "handled like the remote part of a refspec", and the reference
implementation performs no prefix validation
([PR #1923](https://github.com/go-git/go-git/pull/1923)).

**Impact:** Repositories whose config previously triggered a parse error
due to a missing `refs/` prefix will now load successfully. Code that
depended on this validation error to detect misconfigured branches will
need to add its own check.

**How to migrate:** No action required for most users — this is a
relaxation of an overly strict rule. If you previously worked around
parse failures by stripping or adding the prefix, you can remove that
workaround.

**References:** [PR #1923](https://github.com/go-git/go-git/pull/1923)

---

## Behaviour changes

### 9. `DiffTreeWithOptions` deprecated; `DiffTree` will detect renames by default 🔲 Planned

**What changed:** `object.DiffTreeWithOptions` is deprecated and will be
removed in v6. The plan is for `object.DiffTree` to detect renames by
default (equivalent to the current `DefaultDiffTreeOptions`).

**Why:** Rename detection should be the default rather than an opt-in, to
match user expectations and the behaviour of the `git diff` CLI.

**Impact:** Code that calls `DiffTreeWithOptions` with a non-nil options
struct will need to migrate to `DiffTree` or a new options-based API.
Code that calls `DiffTree` directly and does *not* want rename detection
will need to opt out explicitly.

**How to migrate (current recommendation):**

```go
// v5 — opt in to rename detection
changes, err := object.DiffTreeWithOptions(ctx, treeA, treeB,
    object.DefaultDiffTreeOptions)

// v6 — rename detection will be the default
changes, err := object.DiffTree(treeA, treeB)
```

**References:** [`difftree.go`](https://github.com/go-git/go-git/blob/cf196ea5ba6c8b16712a9868cc2d4cca4f287ae4/plumbing/object/difftree.go#L63), [go-git/go-git#910](https://github.com/go-git/go-git/issues/910)

---

### 10. `Worktree.Add` and `Worktree.AddGlob` deprecated 🔲 Planned

**What changed:** `Worktree.Add(path string)` and
`Worktree.AddGlob(pattern string)` are marked for deprecation in v6 in
favour of `Worktree.AddWithOptions(*AddOptions)`.

**Why:** `AddWithOptions` provides a unified, extensible interface for
staging files. The standalone methods duplicate functionality and limit
future extension.

**Impact:** Existing calls to `Add` and `AddGlob` will continue to compile
in v6 but may be removed in a subsequent release.

**How to migrate:**

```go
// v5
hash, err := w.Add("file.txt")

// v6
err := w.AddWithOptions(&git.AddOptions{Path: "file.txt"})
```

```go
// v5
err := w.AddGlob("*.go")

// v6
err := w.AddWithOptions(&git.AddOptions{Glob: "*.go"})
```

**References:** [`worktree_status.go`](https://github.com/go-git/go-git/blob/cf196ea5ba6c8b16712a9868cc2d4cca4f287ae4/worktree_status.go#L273), [go-git/go-git#910](https://github.com/go-git/go-git/issues/910)

---

### 11. `AddOptions.Validate` signature change 🔲 Planned

**What changed:** The `Validate` method on options structs (e.g.
`AddOptions.Validate`) is planned to change its signature from
`Validate(storer.Storer)` to `Validate(r *Repository)`.

**Why:** Passing the full `*Repository` gives validators access to more
context (e.g. config, worktree) without requiring a separate lookup.

**Impact:** Any code that calls `opts.Validate(...)` directly will need
to pass a `*Repository` instead of a `storer.Storer`.

**How to migrate:** Pass the repository instance directly:

```go
// v5
err := opts.Validate(repo.Storer)

// v6
err := opts.Validate(repo)
```

**References:** [`options.go`](https://github.com/go-git/go-git/blob/cf196ea5ba6c8b16712a9868cc2d4cca4f287ae4/options.go#L711), [go-git/go-git#910](https://github.com/go-git/go-git/issues/910)

---

## Planned changes (not yet merged)

The following items are tracked in
[go-git/go-git#910](https://github.com/go-git/go-git/issues/910) but have
not yet been finalised or merged. Details may change before the v6.0.0
release.

### 12. SHA-256 repository support 🔲 Planned

**What changed:** go-git is adding support for SHA-256 object format
repositories alongside existing SHA-1 repositories. Supporting both hash
algorithms simultaneously requires changes to `plumbing.Hash` and related
APIs.

**Why:** The git project has standardised SHA-256 as the next-generation
object hash format, and many modern repositories will migrate to it.

**Current state:** Core infrastructure work is under way — SHA-256 pack
files, loose objects, rev files, and basic init/clone/open operations are
implemented. Outstanding items include signed commits/tags and submodule
support.

**Impact (expected):** Code that stores or compares `plumbing.Hash` values
as fixed-size byte arrays may need to be updated. The exact API surface
will be documented once stabilised.

**References:** [Issue #706](https://github.com/go-git/go-git/issues/706), [Issue #899](https://github.com/go-git/go-git/issues/899)

---

### 13. Global and system git config support 💬 In Discussion

**What changed:** go-git v5 only reads per-repository configuration
(`.git/config`). v6 plans to add support for the global
(`$HOME/.gitconfig`) and system (`/etc/gitconfig`) configuration files,
matching the behaviour of the `git` CLI.

**Why:** Missing global config causes subtle differences for users who have
settings like `user.name`, `user.email`, `core.autocrlf`, or custom URL
rewriting configured globally.

**Current state:** Under active development; the exact API for selecting
config scopes is still being discussed.

**Impact (expected):** The default behaviour of operations that consult
config (e.g. commit author, autocrlf, credential helpers) will change to
also consider global and system config. Users who want repository-only
config can opt out via a new scoped-config API.

**References:** [Issue #395](https://github.com/go-git/go-git/issues/395)

---

### 14. New transport API 🔲 Planned

**What changed:** The internal transport layer is being redesigned. The
`plumbing/transport/server` package's `Server` type and related helpers
will change.

**Why:** The current transport API has grown organically and has some rough
edges around protocol version negotiation, custom loaders, and composability.

**Current state:** The new `plumbing/transport` package structure is taking
shape (see the `serve.go`, `upload_pack.go` and `receive_pack.go`
top-level functions), but a stable public API has not yet been finalised.

**Impact (expected):** Code that registers custom transport clients
(`client.InstallProtocol`) or implements server-side transport handlers
will need to be updated.

**References:** [`server.go`](https://github.com/go-git/go-git/blob/cf196ea5ba6c8b16712a9868cc2d4cca4f287ae4/plumbing/transport/server/server.go#L92), [go-git/go-git#910](https://github.com/go-git/go-git/issues/910)

---

### 15. Additional `repository.go` API changes 🔲 Planned

Several methods in `repository.go` are flagged for revision:

- The internal resolution path used by `Repository.ResolveRevision` and
  related methods is under review.
- Some repository-initialisation helpers may change their signatures.

These changes are still being finalised. Follow
[go-git/go-git#910](https://github.com/go-git/go-git/issues/910) for
updates.

**References:** [`repository.go` L426–440](https://github.com/go-git/go-git/blob/0a1c5ab118998a22ef88d8e6c42970197d77413c/repository.go#L426), [`repository.go` L561](https://github.com/go-git/go-git/blob/cf196ea5ba6c8b16712a9868cc2d4cca4f287ae4/repository.go#L561), [go-git/go-git#910](https://github.com/go-git/go-git/issues/910)

---

## Module path

The Go module import path changes to reflect the new major version:

```go
// v5
import "github.com/go-git/go-git/v5"

// v6
import "github.com/go-git/go-git/v6"
```

Update all subpackage imports accordingly, for example:

```
github.com/go-git/go-git/v5/plumbing  →  github.com/go-git/go-git/v6/plumbing
github.com/go-git/go-git/v5/config    →  github.com/go-git/go-git/v6/config
```

---

## References

- [API and Behaviour changes for v6 (go-git/go-git#910)](https://github.com/go-git/go-git/issues/910)
- [SHA-256 support (go-git/go-git#706)](https://github.com/go-git/go-git/issues/706)
- [SHA-1 and SHA-256 simultaneously (go-git/go-git#899)](https://github.com/go-git/go-git/issues/899)
- [Global git config support (go-git/go-git#395)](https://github.com/go-git/go-git/issues/395)
- [commitgraph chain support (PR #854)](https://github.com/go-git/go-git/pull/854)
- [Tags not returned by default on Fetch (PR #1459)](https://github.com/go-git/go-git/pull/1459)
- [Revive linter / constant renames (PR #1771)](https://github.com/go-git/go-git/pull/1771)
- [branch.*.merge validation fix (PR #1923)](https://github.com/go-git/go-git/pull/1923)
- [Repository.Close() (PR #1906)](https://github.com/go-git/go-git/pull/1906)
