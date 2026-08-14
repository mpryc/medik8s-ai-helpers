# Z-Stream (Patch) Release Process

Detailed guide for Z-stream (patch) releases. Same 7-phase JIRA flow as Y-stream but with fewer steps.

## What's Different from Y-Stream

| JIRA Phase | Y-stream | Z-stream |
|------------|----------|----------|
| Phase 0: Upstream | Create release branch | Reuse existing |
| Phase 0: Tag | New tag | New tag (required!) |
| Phase 0: Community | Same | Same |
| JIRA 01: Konflux | Create new RP/RPA/KRD | Reuse existing, update version |
| JIRA 02: Midstream branch | Create new | Reuse existing |
| JIRA 02: Tekton files | Create new pipeline files | Same files, version bumped by script |
| JIRA 02: skipRange | Set SKIP_RANGE_LOWER | Comment out (first Z-stream) |
| JIRA 02: FBC releases.yaml | Add new release block | No change |
| JIRA 02: FBC operator-versions | Add new version entry | Add new patch entry |
| JIRA 02: FBC Containerfile | May need new | No change |
| JIRA 03-06 | Same | Same |
| JIRA 07: Errata | New advisory (manual) | May be auto-created |
| Post-GA: FBC bootstrap | Required for next OCP | Not needed |

## Key Points

### New Upstream Tag is Required
Every Z-stream needs a new git tag (e.g., `v0.13.1`). The downstream build references this exact tag.

### Tekton semver-version Must Be Bumped
The `.tekton/*.yaml` files hardcode `semver-version: v0.13.0`. For a Z-stream to `0.13.1`, this must be updated to `v0.13.1` using `scripts/update-version.sh`.

### skipRange Handling
For the first Z-stream after a Y-stream, **comment out** `SKIP_RANGE_LOWER` in `bundle-hack/update_bundle.sh`. This ensures OLM can upgrade from the Y-stream version.

### Konflux CRs are Reused
For Z-stream, the ReleasePlans, RPAs, and Application/Component CRs from the Y-stream minor are reused. Only the version reference in the RP synopsis needs updating.

## Quick Steps (mapped to JIRA phases)

**Phase 0 (Upstream)**:
1. Cherry-pick fixes to release branch upstream
2. Create new upstream tag: `git tag -s v0.13.1`
3. Run community release (same as Y-stream)

**JIRA 01 (Konflux Update)**:
4. Verify existing RP/RPA in `rhwa-tenant` — update version in RP synopsis

**JIRA 02 (Update Downstream Repo)**:
5. Bump version in midstream: `scripts/update-version.sh`
6. Update `bundle-hack/update_bundle.sh` (PREVIOUS_VERSION, comment out skipRange)
7. Push to existing midstream branch → Konflux builds automatically
8. Update FBC `operator-versions.yaml` with new patch entry

**JIRA 03-06 (Verify, Test, QE, Docs)**:
9. Same process as Y-stream

**JIRA 07 (GA Release)**:
10. Create Konflux Release CRs on release day
11. Tag downstream repos

## Use the Audit Command

Run `/medik8s-release:audit --version 0.13.1 --ocp 4.22` to track progress through all JIRA phases.
