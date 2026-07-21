# Z-Stream (Patch) Release Process

Detailed guide for Z-stream (patch) releases. Same pipeline as Y-stream but with fewer steps.

## What's Different from Y-Stream

| Step | Y-stream | Z-stream |
|------|----------|----------|
| Upstream release branch | Create new | Reuse existing |
| Upstream tag | New tag | New tag (required!) |
| Community release | Same | Same |
| Midstream branch | Create new | Reuse existing |
| `scripts/update-version.sh` | Set new minor | Bump patch |
| `.tekton/*.yaml` | Create new files | Same files, version bumped by script |
| `bundle-hack/update_bundle.sh` | Set PREVIOUS + SKIP_RANGE_LOWER | Update PREVIOUS, comment out SKIP_RANGE_LOWER (first Z) |
| FBC `releases.yaml` | Add new release block | No change |
| FBC `operator-versions.yaml` | Add new version entry | Add new patch entry |
| FBC Containerfile | May need new | No change |
| FBC Tekton pipelines | May need new pair | No change |
| Errata | New advisory (manual) | May be auto-created |

## Key Points

### New Upstream Tag is Required
Every Z-stream needs a new git tag (e.g., `v0.13.1`). The downstream build references this exact tag.

### Tekton semver-version Must Be Bumped
The `.tekton/*.yaml` files hardcode `semver-version: v0.13.0`. For a Z-stream to `0.13.1`, this must be updated to `v0.13.1` using `scripts/update-version.sh`.

### skipRange Handling
For the first Z-stream after a Y-stream, **comment out** `SKIP_RANGE_LOWER` in `bundle-hack/update_bundle.sh`. This ensures OLM can upgrade from the Y-stream version.

## Quick Steps

1. Cherry-pick fixes to release branch upstream
2. Create new upstream tag: `git tag -s v0.13.1`
3. Run community release (same as Y-stream)
4. Bump version in midstream: `scripts/update-version.sh`
5. Update `bundle-hack/update_bundle.sh` (PREVIOUS_VERSION, skipRange)
6. Push to existing midstream branch → Konflux builds automatically
7. Update FBC `operator-versions.yaml` with new patch entry
8. Create Konflux Release CRs
9. Tag downstream repos

## Use the Audit Command

Run `/medik8s-release:audit --version 0.13.1 --ocp 4.22` to track progress through all steps.
