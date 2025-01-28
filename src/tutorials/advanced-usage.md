---
layout: default
title: Advanced Usage of go-git
---
# Advanced Usage of go-git

This guide covers advanced usage patterns and performance considerations
when working with go-git.

## Performance Optimizations

When working with large repositories or performing intensive operations,
consider the following performance tips:

### Reduce the amount of data being cloned
- Shallow clones with setting depth to 1.
- Use Sparse Checkout for only representing part of the repository tree
in the Worktree.
- When no tags are required, use `git.NoTags`. This will skip the cloning
of tags, and all the objects they point to - which may result in a
significant increase in cloned data.

### Storage Optimization
- The fastest storage is the memory storage. For very large repositories
make sure there is enough memory available. Used in conjunction with the
filter option can lead to super efficient operations.

### SHA1
- By default, go-git uses `github.com/pjbgf/sha1cd` as the SHA1 algorithm.
It introduces collision detection, which results in slower performance.
When only trustworthy servers are used, consider using the native SHA1
implementation with `hash.RegisterHash(sha1.New())`.

## In-memory repositories

### Clone in between repositories held in memory

Here's an example of using a memfs filesystem, which can be used for multiple
repositories. In this case, it can also be used to clone in-between sub-memfs
filesystems which in fact reside on a higher level memfs.

```go
var wg sync.WaitGroup

// /
// /target
// /src
//	/go-git
//	/go-billy
func main() {
	rootfs := memfs.New()
	target, _ := rootfs.Chroot("/target")
	src, _ := rootfs.Chroot("/src")
	master, _ := src.Chroot("/go-git-master")
	concurrency, _ := src.Chroot("/go-git-concurrency")

	wg.Add(2)
	// providing a thread-safe memfs enables running the below as go routines
	clone(master, "https://github.com/go-git/go-git", "master")
	clone(concurrency, "https://github.com/go-git/go-git", "read-concurrency")
	wg.Wait()

	loader := server.NewFilesystemLoader(rootfs)
	client.InstallProtocol("file", server.NewClient(loader))

	url := "file:///src/go-git-master"

	dot, err := target.Chroot(".git")
	CheckIfError(err)

	Info("git clone %s", url)
	storer := filesystem.NewStorage(dot, cache.NewObjectLRUDefault())
	r, err := git.Clone(storer, target, &git.CloneOptions{
		URL: url,
	})
	CheckIfError(err)

	ref, err := r.Head()
	CheckIfError(err)

	commit, err := r.CommitObject(ref.Hash())
	CheckIfError(err)

	fmt.Println(commit)

	url2 := "file:///src/go-git-concurrency"
	err = r.Fetch(&git.FetchOptions{
		RemoteURL: url2,
		RefSpecs: []config.RefSpec{
			...
		},
	})
	CheckIfError(err)
}

func clone(fs billy.Filesystem, url, branch string) {
	storer := filesystem.NewStorage(fs, cache.NewObjectLRUDefault())

	_, err := git.Clone(storer, nil, &git.CloneOptions{
		URL:           url,
		SingleBranch:  true,
		ReferenceName: plumbing.NewBranchReferenceName(branch),
	})
	wg.Done()

	CheckIfError(err)
	fmt.Printf("Clone %q from %q: DONE\n", branch, url)
}
```
