---
description: Sync all local repos with remotes, report repos on non-default branches
argument-hint: [--upstream] [--downstream] [--forks] [--dry-run]
---

## Name
medik8s-release:sync

## Synopsis
```
/medik8s-release:sync [--upstream] [--downstream] [--forks] [--dry-run]
```

## Description
The `medik8s-release:sync` command fetches and fast-forwards all local medik8s repositories.

It updates the default branch (main/master or operator-specific like `snr-0-13`) even when checked out on a feature branch, and reports repos with active work on non-default branches.

### Repository Groups
- **`--upstream`**: All upstream GitHub repos (operators, shared, UI, docs, OLM, archived)
- **`--downstream`**: All midstream GitLab repos + RHWA repos (requires VPN)
- **`--forks`**: All personal forks
- Default (no flags): sync all groups

## Implementation

### Discover and Sync Repos
**Do NOT assume any fixed directory layout.** Discover medik8s repos dynamically:

```bash
# Find all medik8s-related git repos under $HOME
REPOS=$(find "$HOME" -maxdepth 6 -type d -name ".git" 2>/dev/null | while read gitdir; do
  dir=$(dirname "$gitdir")
  remote_url=$(git -C "$dir" remote get-url origin 2>/dev/null)
  case "$remote_url" in
    *github.com/medik8s/*|*github.com/*medik8s*|*gitlab*medik8s*|*gitlab*rhwa*)
      echo "$dir";;
  esac
done)
```

If `--upstream` is specified, filter to repos with `github.com/medik8s` remotes.
If `--downstream` is specified, filter to repos with `gitlab` remotes.
If `--forks` is specified, filter to repos with fork-pattern remotes.

For each discovered repo:
1. Run `git fetch --all`
2. If on default branch, run `git merge --ff-only`
3. If on a non-default branch, update the default branch ref without switching (`git fetch origin main:main`)
4. Collect results and print grouped summary
5. Report failures with VPN hint for downstream repos

## Return Value
Summary table showing synced/skipped/failed repos, plus list of repos on non-default branches.

## Examples

1. **Sync everything**:
   ```
   /medik8s-release:sync
   ```

2. **Sync only upstream**:
   ```
   /medik8s-release:sync --upstream
   ```

3. **Dry-run**:
   ```
   /medik8s-release:sync --dry-run
   ```

## Arguments
- `--upstream`: Sync only upstream repos
- `--downstream`: Sync only downstream/GitLab repos
- `--forks`: Sync only personal forks
- `--dry-run`: Show what would be synced without doing it

## See Also
- `/medik8s-release:status` — Check release readiness after syncing
