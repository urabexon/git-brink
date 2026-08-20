# git-brink

**English** · [日本語](./README.ja.md) · [Français](./README.fr.md)

Show what a stray `git add -A` would leak from your working tree — before it happens.

```
$ git brink ~

git-brink  /Users/you
origin: git@github.com:you/some-public-repo.git

CRITICAL  repository root is your home directory
          everything under $HOME is inside this repository's working tree
CRITICAL  58 sensitive path(s) would be staged by 'git add -A'
          .aws/                              AWS credentials
          .ssh/                              SSH keys
          .config/gh/hosts.yml               GitHub CLI token
          .s3cfg                             S3 credentials
          .zsh_history                       shell/tool history
          projects/deploy-key.pem            key material
          ... and 38 more
WARN      159 nested git repositories inside the working tree

2 critical, 1 warning(s).
```

## Why

Secret scanners look at what you have *already* committed or staged.
This looks at what is about to *become* committable.

The failure mode it was written for is embarrassingly simple: a `git init` or
`git clone` run one directory too high. Suddenly `$HOME` is a repository,
`.ssh/` and `.aws/` are sitting in its working tree, `origin` points somewhere
public — and nothing is wrong yet. Nothing is staged. Nothing is committed.
A scanner finds nothing, because there is nothing to find. You are one
`git add -A` away from a very bad afternoon.

git-brink answers the only question that matters at that moment:
**if I ran `git add -A && git commit && git push` right now, what would go out?**

## Install

```sh
curl -o /usr/local/bin/git-brink \
  https://raw.githubusercontent.com/urabexon/git-brink/main/git-brink
chmod +x /usr/local/bin/git-brink
```

Anything named `git-*` on your `PATH` becomes a git subcommand, so you can
then run it as `git brink`.

Requires bash 3.2+ (the macOS system bash is fine) and git.
`gh` is used opportunistically to check whether the remote is public, and is
never required.

## Usage

```sh
git brink                 # check the repository containing the current directory
git brink ~/projects      # check the repository containing a given path
git brink --no-network    # skip the remote-visibility lookup
```

### Exit codes

| Code | Meaning |
|------|---------|
| `0`  | nothing found |
| `1`  | warnings only |
| `2`  | at least one critical finding |

Useful in a pre-commit hook:

```sh
#!/bin/sh
git brink --no-network || exit 1
```

## What it checks

| Severity | Check |
|----------|-------|
| CRITICAL | Repository root is `$HOME`, an ancestor of `$HOME`, or `/` |
| CRITICAL | Untracked, unignored paths that look like keys, credentials, tokens, environment files, or shell history |
| CRITICAL | The above, when the remote is public |
| WARN | No `.gitignore` while untracked entries exist |
| WARN | Nested git repositories inside the working tree |
| WARN | Scan budget exhausted, so the result is inconclusive |

Sample and template files (`.env.example`, `*.sample`, `*.template`) and the
public halves of key pairs (`*.pub`) are deliberately not reported.

## How it stays fast

The naive approach — enumerating every untracked file — takes minutes on a
repository rooted at `$HOME`, because git walks `Library/`, every cache, and
every `node_modules` on the disk. git-brink asks git to collapse wholly
untracked directories instead, then descends into them itself, shallowly and
under a fixed budget. The home-directory case above completes in about three
seconds.

If the budget runs out, git-brink says so rather than reporting a clean
result it cannot vouch for.

## What it is not

It does not scan file contents, and it does not look at history. If you need
pattern-matching on what is already committed, use
[gitleaks](https://github.com/gitleaks/gitleaks) or
[git-secrets](https://github.com/awslabs/git-secrets) — they solve the other
half of the problem, and the two compose well.

## License

[MIT](./LICENSE)
