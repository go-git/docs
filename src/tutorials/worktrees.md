---
layout: default
title: Working with Linked Worktrees
---
# Working with Linked Worktrees

Git worktrees allow you to check out multiple branches simultaneously, each
in its own working directory. go-git supports creating and opening linked
worktrees through the experimental `x/plumbing/worktree` package.

## Overview

A Git repository has one **main worktree** (created by `git init` or
`git clone`) and zero or more **linked worktrees**. Each linked worktree
has its own checked-out branch and working directory, while sharing the
same object database and refs with the main repository.

Common use cases:

- Running tests on one branch while developing on another
- Comparing behavior across branches side by side
- Building a release while continuing feature work

## Creating a Linked Worktree

To create a linked worktree, you need:

1. A `storage.Storer` that implements `WorktreeStorer` (e.g. `filesystem.Storage`)
2. A `billy.Filesystem` for the new worktree's working directory
3. The commit hash to check out

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"

	"github.com/go-git/go-billy/v6/osfs"
	"github.com/go-git/go-git/v6"
	"github.com/go-git/go-git/v6/plumbing"
	"github.com/go-git/go-git/v6/storage/filesystem"

	xworktree "github.com/go-git/go-git/v6/x/plumbing/worktree"
)

func main() {
	repoPath := "/path/to/repo"
	worktreePath := "/path/to/new-worktree"
	commitHash := "abc123..."

	// Open the repository's .git storage
	dotgitFs := osfs.New(filepath.Join(repoPath, ".git"), osfs.WithChrootOS())
	store := filesystem.NewStorageWithOptions(dotgitFs, nil, filesystem.Options{})

	// Create a worktree manager
	wt, err := xworktree.New(store)
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}

	// Create the linked worktree filesystem
	worktreeFs := osfs.New(worktreePath)
	name := filepath.Base(worktreePath)

	// Add the worktree at the given commit
	err = wt.Add(worktreeFs, name,
		xworktree.WithCommit(plumbing.NewHash(commitHash)))
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Worktree %q created at %s\n", name, worktreePath)
}
```

## Opening an Existing Linked Worktree

Once a linked worktree exists on disk, open it like any other repository
using `git.Open` with the shared storage and the worktree filesystem:

```go
r, err := git.Open(store, worktreeFs)
if err != nil {
	log.Fatal(err)
}

ref, err := r.Head()
if err != nil {
	log.Fatal(err)
}

commit, err := r.CommitObject(ref.Hash())
if err != nil {
	log.Fatal(err)
}

fmt.Println(commit)
```

## Worktree Configuration Extension

When `extensions.worktreeConfig` is enabled in the repository config,
each linked worktree can have its own `config.worktree` file at
`.git/worktrees/<name>/config.worktree`. go-git's filesystem storage
reads these files and overlays them on the shared repository config,
matching the behavior of upstream Git.

This is useful for per-worktree settings such as `core.sparseCheckout`
or custom configuration that should differ between worktrees.

Reference: [git-worktree configuration file](https://git-scm.com/docs/git-worktree#_configuration_file)

## Worktree Naming

Worktree names must match the pattern `[a-zA-Z0-9\-]+`. Typically the
name is derived from the base directory name of the worktree path (e.g.
`/tmp/hotfix` produces the name `hotfix`).

## Limitations

- Only `add` is fully supported for creating worktrees. Not all flags
  or subcommands from `git worktree` are implemented.
- The worktree package lives in `x/plumbing/worktree`, meaning its API
  is **experimental** and may change without notice.
- The storer must implement `WorktreeStorer` — currently only
  `storage/filesystem` satisfies this.
- Worktree lock, move, and prune operations are not yet supported.

See the [full example](_examples/worktrees/main.go) in the go-git repository.
