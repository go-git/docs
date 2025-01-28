---
layout: default
title: go-git Documentation
---

# go-git Documentation

go-git is a highly extensible Git implementation in pure Go. This documentation will help you understand and use go-git effectively in your projects.

**Warning:** This documentation targets the v6 version of go-git. APIs are subject to change until v6 is officially released.

## Quick Start

```go
import (
    "github.com/go-git/go-git/v5"
)

func main() {
    // Clone repository
    repo, err := git.PlainClone("/path/to/repo", false, &git.CloneOptions{
        URL: "https://github.com/go-git/go-git",
    })
}
```

## Basic Concepts

For a detailed explanation of go-git's architecture, please refer to our [Architecture document](docs/concepts/architecture.md).
