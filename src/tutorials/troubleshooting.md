---
layout: default
title: Troubleshooting
---
# Troubleshooting

This guide covers common issues encountered when using go-git.

## Debugging and Tracing

### Available Trace Options

go-git supports different tracing levels. Enable them by calling `trace.SetTarget`:

```golang
import `github.com/go-git/go-git/v6/utils/trace`

...

trace.SetTarget(trace.SSH | trace.Performance)
```

Here are the current trace options:

- General: General traces general operations.
- Packet: Packet traces git packets.
- SSH:	SSH handshake operations. This does not have a direct translation to an upstream trace option.
- Performance: performance information for specific go-git operations.
- HTTP: HTTP operations and requests.
```

### SSH Connection Issues

To enable trace information for SSH handshakes, use the `GIT_TRACE_SSH` environment variable:

```bash
$ GIT_TRACE_SSH=true go run _examples/clone/main.go git@github.com:go-git/go-git /tmp/go-git
19:09:14.134147 common.go:43: ssh: Using default auth builder (user: git)
19:09:14.134193 sshagent.go:42: ssh: net.Dial unix sock /tmp/ssh-XXXXXXULYg1c/agent.6427
19:09:14.134224 auth_method.go:224: ssh: ssh-public-key-callback user=git
19:09:14.134317 auth_method.go:345: ssh: found key: sk-ssh-ed25519@openssh.com SHA256:otsLd7qnCx9TrhUfIPCiP2cbk3Rkb5gtxyUJy2Zeheo
19:09:14.134323 auth_method.go:255: ssh: known_hosts sources [/home/levi/.ssh/known_hosts /etc/ssh/ssh_known_hosts]
19:09:14.134330 auth_method.go:260: ssh: filtered known_hosts sources [/home/levi/.ssh/known_hosts]
19:09:14.134533 auth_method.go:255: ssh: known_hosts sources [/home/levi/.ssh/known_hosts /etc/ssh/ssh_known_hosts]
19:09:14.134540 auth_method.go:260: ssh: filtered known_hosts sources [/home/levi/.ssh/known_hosts]
19:09:14.134577 common.go:155: ssh: host key algorithms [ssh-ed25519]
19:09:14.458332 auth_method.go:330: ssh: hostkey callback hostname=github.com:22 remote=20.26.156.215:22 pubkey="ssh-ed25519 SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU"
```

## Common Errors and Solutions

### Performance Issues

#### Large Repository Cloning
**Problem:** Slow clone times for large repositories

**Solutions:**
1. Use shallow clone:
   ```go
   _, err := git.PlainClone("/path", &git.CloneOptions{
       URL:   "https://github.com/user/repo",
       Depth: 1, // Only fetch latest commit
   })
   ```

2. Clone single branch:
   ```go
   _, err := git.PlainClone("/path", &git.CloneOptions{
       URL:          "https://github.com/user/repo", 
       SingleBranch: true,
       ReferenceName: plumbing.NewBranchReferenceName("main"),
   })
   ```

3. Skip tags when not needed:
   ```go
   _, err := git.PlainClone("/path", &git.CloneOptions{
       URL:     "https://github.com/user/repo",
       NoTags:  true,
   })
   ```

## Getting Help

If you encounter issues not covered here:

1. Check the [examples directory](https://github.com/go-git/go-git/tree/main/_examples) for usage patterns
2. Enable tracing to get detailed debug information
3. Review the [GitHub issues](https://github.com/go-git/go-git/issues) for similar problems
4. Create a minimal reproduction case when reporting bugs
