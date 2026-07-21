# Release Workflow — End-to-End Guide

Complete step-by-step guide for releasing medik8s operators, covering both Y-stream (minor) and Z-stream (patch) releases across upstream, midstream, and downstream.

## Overview

A medik8s release flows through three environments:

```
Upstream (GitHub)  →  Midstream (GitLab)  →  Downstream (Konflux + registry.redhat.io)
     tag, build          version bump           hermetic build, FBC, Errata
     community PRs       bundle-hack            Release CRs, stage/prod
```

## Y-Stream (Minor) Release

### Prerequisites
- All features/fixes merged to `main`
- CI green on all operators
- SAST review completed (3 weeks before Feature Freeze)
- Release versions planned (version, previous, skipRange, OCP target)

### Step 1: Create Upstream Release Branch (at Code Freeze)
```bash
git checkout main && git pull
git checkout -b release-0.13
git push origin release-0.13
```

### Step 2: Upstream Release
Use `/medik8s-release:upstream` or `community-release.sh`:
- Create tags on upstream repos
- Build & push images to quay.io/medik8s
- Create community catalog PRs (K8s + OKD)

### Step 3: Create Midstream Branch
```bash
# On GitLab, create new branch from previous release branch
git checkout snr-0-12
git checkout -b snr-0-13
git push origin snr-0-13
# Set as default branch on GitLab
```

### Step 4: Version Bump in Midstream
Use `/medik8s-release:version-bump`:
- Edit `scripts/update-version.sh` with new version
- Run script to update Tekton files, Containerfiles, test scripts

### Step 5: Update bundle-hack
Use `/medik8s-release:downstream`:
- Set `PREVIOUS_VERSION`, `SKIP_RANGE_LOWER`, `DOCS_RHWA_VERSION`
- Commit and push to midstream branch

### Step 6: Create Tekton Pipelines (if new minor)
Create new `.tekton/` pipeline files for the new version.

### Step 7: Konflux Builds
Pushing to midstream triggers Tekton PipelineRuns automatically.
Verify builds at `quay.io/redhat-user-workloads/rhwa-tenant/`.

### Step 8: Update FBC
Use `/medik8s-release:fbc`:
- Add version to `operator-versions.yaml` (with `update-bundle: true`)
- Add version to `releases.yaml`
- Run `gen_template.sh` and `gen_fbc.sh`
- Verify with `verify-bundle-images.sh`

### Step 9: Create Konflux Release CRs
Run `create_release.sh` from `dragonfly/tools/releases/`.

### Step 10: Errata & GA
- Create/update Errata advisory
- Move through: NEW FILES → QA → REL_PREP → SHIPPED LIVE
- After GA: update FBC to use `registry.redhat.io`, remove `update-bundle: true`

### Step 11: Post-Release
- Tag downstream repos (`tag_downstream.sh`)
- Publish GitHub Releases
- Announce release

## Z-Stream (Patch) Release

Same flow but skip: branch creation, new Tekton files, new FBC Containerfiles.
Key difference: comment out `SKIP_RANGE_LOWER` for first Z-stream after Y-stream.

See `/medik8s-release:audit` to track progress.

## Catalog Artifacts

**Important distinction:**
- **FBC fragment** (Konflux component) = intermediate, RHWA-only catalog
- **Stage index** (release artifact) = full `redhat-operator-index` after IIB merge

QE must test against the stage index, not the intermediate fragment, to catch merge issues.

## Audit Your Progress

Run `/medik8s-release:audit --version X.Y.Z --ocp 4.22` at any time to see a live checklist of what's done and what remains.
