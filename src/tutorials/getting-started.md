---
layout: default
title: Getting Started with go-git
---
# Getting Started with go-git

This guide demonstrates common Git operations using the go-git library.

The [go-git repository] contains many examples of how to do various
operations using the latest version of go-git.

[go-git repository]: https://github.com/go-git/go-git/tree/main/_examples

## Cloning a Repository

The following example shows how to clone a Git repository to your local filesystem:

```go
r, err := git.PlainClone("/path/to/repo", &git.CloneOptions{
    URL: "https://github.com/go-git/go-git",
})
```

## In-memory operations

The go-git library supports in-memory Git operations, which can be useful
for faster cloning operations without touching the filesystem.

Here's how to clone a repository into memory:

### "With worktree"

```go
wt := memfs.New()
storer := memory.NewStorage()
r, err := git.Clone(storer, wt, &git.CloneOptions{
    URL: "https://github.com/go-git/go-git",
})
```

### "Bare clone"

```go
storer := memory.NewStorage()
r, err := git.Clone(storer, nil, &git.CloneOptions{
    URL: "https://github.com/go-git/go-git",
})
```

For bare clones, the `Worktree` parameter should be set to `nil`.
