---
layout: default
title: Migrating from v5 to v6
---
# Migrating from v5 to v6

**Warning:** This documentation targets the v6 version of go-git, and it is subject to change until v6 is officially released.

The v6 release of go-git brings significant changes to the API. This guide will help you migrate your code from v5 to v6.

## Changes to `git` package:

- The `bare` argument has been removed from `git.PlainClone` and `git.PlainInit`. A new field `Bare` has been added to `git.CloneOptions` and `git.InitOptions` to indicate whether the repository should be cloned as a bare repository. The same applies to the `*Context()` versions of these functions.


## Changes to `plumbing` package:
- `plumbing.TagMode` has been moved to `git.TagMode`.

## References

- [API and Behaviour changes for v6](https://github.com/go-git/go-git/issues/910)
