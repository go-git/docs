---
layout: default
title: Object Signing
---
# Object Signing

The `plugin.ObjectSigner` plugin lets applications provide a cryptographic signer for Git objects. When registered and the config settings `tag.gpgSign=true` or `commit.gpgSign=true` are set, `go-git` will automatically sign every new commit and tag using the configured signer.

A signer is any value that satisfies `plugin.Signer`:

```go
type Signer interface {
    Sign(message io.Reader) ([]byte, error)
}
```

The extended repository (`github.com/go-git/x`) ships two ready-to-use implementations:

| Package | Format | Protocol |
|---------|--------|----------|
| `github.com/go-git/x/plugin/objectsigner/gpg` | OpenPGP / GPG | ASCII-armored detached signature |
| `github.com/go-git/x/plugin/objectsigner/ssh` | SSH | [sshsig] armored signature |

## Application-level vs per-operation signing

There are two ways to attach a signer.

**Application-level** — Register once at startup. Every subsequent commit and tag is signed automatically without changing any call sites.

```go
func init() {
    plugin.Register(plugin.ObjectSigner(), func() plugin.Signer {
        return mySigner
    })
}
```

**Per-operation** — Pass a `Signer` directly in `CommitOptions` or `CreateTagOptions`. This takes precedence over the application-level plugin for that single call.

```go
w.Commit("message", &git.CommitOptions{
    Signer: mySigner,
    Author: &object.Signature{ /* ... */ },
})
```

## GPG / OpenPGP signing

Install the package:

```
go get github.com/go-git/x/plugin/objectsigner/gpg
```

`gpg.FromKey` wraps an `*openpgp.Entity` from [`github.com/ProtonMail/go-crypto`] and produces ASCII-armored detached signatures.

The following example registers a freshly generated RSA-2048 / SHA-256 key at application level, initialises an in-memory repository, commits a file, and prints both the commit object and its raw signature:

```go
package main

import (
	"crypto"
	"fmt"
	"os"
	"time"

	"github.com/ProtonMail/go-crypto/openpgp"
	"github.com/ProtonMail/go-crypto/openpgp/packet"
	"github.com/go-git/go-billy/v6/memfs"
	"github.com/go-git/x/plugin/objectsigner/gpg"

	"github.com/go-git/go-git/v6"
	. "github.com/go-git/go-git/v6/_examples"
	"github.com/go-git/go-git/v6/plumbing/object"
	"github.com/go-git/go-git/v6/storage/memory"
	"github.com/go-git/go-git/v6/x/plugin"
)

func main() {
	key, err := getKey()
	CheckIfError(err)

	// Register a static GPG signer at application level. All new Tags and Commits
	// will be signed by this key, unless overwritten on a "per-operation" basis.
	err = plugin.Register(plugin.ObjectSigner(), func() plugin.Signer { return gpg.FromKey(key) })
	CheckIfError(err)

	fs := memfs.New()
	st := memory.NewStorage()
	r, err := git.Init(st, git.WithWorkTree(fs))
	CheckIfError(err)

	w, err := r.Worktree()
	CheckIfError(err)

	// Create a new file inside the worktree.
	Info("echo \"hello world!\" > example-git-file")
	f, err := fs.OpenFile("example-git-file", os.O_CREATE|os.O_RDWR, 0o644)
	CheckIfError(err)

	_, err = f.Write([]byte("hello world!"))
	CheckIfError(err)
	CheckIfError(f.Close())

	// Add the new file to the staging area.
	Info("git add example-git-file")
	_, err = w.Add("example-git-file")
	CheckIfError(err)

	Info("git commit -m \"example go-git commit\"")
	commit, err := w.Commit("example go-git commit", &git.CommitOptions{
		// A signer can be defined on a per-operation basis, instead of the
		// application-level as per plugin.Register.
		// Signer: gpg.FromKey(key),

		Author: &object.Signature{
			Name:  "John Doe",
			Email: "john@doe.org",
			When:  time.Now(),
		},
	})
	CheckIfError(err)

	// Print the commit.
	Info("git show -s")
	obj, err := r.CommitObject(commit)
	CheckIfError(err)

	fmt.Println(obj)

	// Print raw signature.
	fmt.Println(obj.Signature)
}

func getKey() (*openpgp.Entity, error) {
	cfg := &packet.Config{
		RSABits:     2048,
		Time:        func() time.Time { return time.Now() },
		DefaultHash: crypto.SHA256,
	}

	return openpgp.NewEntity(
		"Test User",        // name
		"Test Key",         // comment
		"test@example.com", // email
		cfg,
	)
}
```

## SSH signing

Install the package:

```
go get github.com/go-git/x/plugin/objectsigner/ssh
```

`ssh.FromKey` wraps a `golang.org/x/crypto/ssh.Signer` and produces [sshsig]-format armored signatures. Two hash algorithms are supported: `ssh.SHA256` and `ssh.SHA512` (the default).

Keys can come from an SSH agent, a key file, or be generated in memory (useful for testing). The example below tries the SSH agent first and falls back to the other helpers:

```go
package main

import (
	"crypto/ecdsa"
	"crypto/elliptic"
	"crypto/rand"
	"fmt"
	"net"
	"os"
	"time"

	"github.com/go-git/go-billy/v6/memfs"
	"github.com/go-git/x/plugin/objectsigner/ssh"
	gossh "golang.org/x/crypto/ssh"
	"golang.org/x/crypto/ssh/agent"

	"github.com/go-git/go-git/v6"
	. "github.com/go-git/go-git/v6/_examples"
	"github.com/go-git/go-git/v6/plumbing/object"
	"github.com/go-git/go-git/v6/storage/memory"
	"github.com/go-git/go-git/v6/x/plugin"
)

func main() {
	// Generally this key would come from a file or SSH agent.
	key, err := getKeyFromAgent()
	CheckIfError(err)

	// Register a static SSH signer at application level. All new Tags and Commits
	// will be signed by this key, unless overwritten on a "per-operation" basis.
	err = plugin.Register(plugin.ObjectSigner(),
		func() plugin.Signer { return ssh.FromKey(key) })
	CheckIfError(err)

	fs := memfs.New()
	st := memory.NewStorage()
	r, err := git.Init(st, git.WithWorkTree(fs))
	CheckIfError(err)

	w, err := r.Worktree()
	CheckIfError(err)

	// Create a new file inside the worktree.
	Info("echo \"hello world!\" > example-git-file")
	f, err := fs.OpenFile("example-git-file", os.O_CREATE|os.O_RDWR, 0o644)
	CheckIfError(err)

	_, err = f.Write([]byte("hello world!"))
	CheckIfError(err)
	CheckIfError(f.Close())

	// Add the new file to the staging area.
	Info("git add example-git-file")
	_, err = w.Add("example-git-file")
	CheckIfError(err)

	Info("git commit -m \"example go-git commit\"")
	commit, err := w.Commit("example go-git commit", &git.CommitOptions{
		// A signer can be defined on a per-operation basis, instead of the
		// application-level as per plugin.Register.
		// Signer: ssh.FromKey(key, ssh.SHA512),

		Author: &object.Signature{
			Name:  "John Doe",
			Email: "john@doe.org",
			When:  time.Now(),
		},
	})
	CheckIfError(err)

	// Print the commit.
	Info("git show -s")
	obj, err := r.CommitObject(commit)
	CheckIfError(err)

	fmt.Println(obj)

	// Print raw signature.
	fmt.Println(obj.Signature)
}

// getKey generates a fresh ECDSA P-256 key (useful for testing).
func getKey() (gossh.Signer, error) {
	key, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	if err != nil {
		return nil, err
	}
	return gossh.NewSignerFromKey(key)
}

// getKeyFromFile loads an SSH private key from disk with an optional passphrase.
func getKeyFromFile(file, passphrase string) (gossh.Signer, error) {
	keyBytes, err := os.ReadFile(file)
	CheckIfError(err)

	if len(passphrase) > 0 {
		return gossh.ParsePrivateKeyWithPassphrase(keyBytes, []byte(passphrase))
	}

	return gossh.ParsePrivateKey(keyBytes)
}
```

[sshsig]: https://github.com/openssh/openssh-portable/blob/V_10_2/PROTOCOL.sshsig
[`github.com/ProtonMail/go-crypto`]: https://github.com/ProtonMail/go-crypto
