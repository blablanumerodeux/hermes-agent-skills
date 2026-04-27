---
name: gh-git-auth-helper
description: Fix git push failing with credential errors despite gh auth being valid — use x-access-token in remote URL
---
# GitHub Push via `gh auth token` — Credential Fix

## Context

When `gh` is authenticated but `git push` fails with:
```
fatal: could not read Username for 'https://github.com': No such device or address
```
or:
```
remote: Invalid username or token. Password authentication is not supported for Git operations.
fatal: Authentication failed
```

The `gh auth token` output isn't routing to git's credential helper in this environment.

## Fix

```bash
cd /path/to/repo
TOKEN=$(gh auth token 2>/dev/null)
git remote set-url origin https://x-access-token:${TOKEN}@github.com/owner/repo.git
git push
```

Then restore the clean remote URL if needed:
```bash
git remote set-url origin https://github.com/owner/repo.git
```

## When to Use

- `git push` fails with credential errors after `gh auth status` shows you're logged in
- `git config --list | grep credential` returns nothing
- `GIT_ASKPASS=echo git push` still fails

## Verification

```bash
gh auth status   # confirms GitHub CLI is authenticated
git push         # should succeed without prompting
```

## Notes

- This bypasses the credential helper entirely by embedding the token in the remote URL
- Only use for push — don't leave the token in the remote URL permanently
- Restore clean URL after push: `git remote set-url origin https://github.com/owner/repo.git`
