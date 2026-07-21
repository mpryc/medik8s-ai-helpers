---
description: Prepare midstream repo for release (update bundle-hack, verify Tekton files, commit and push)
argument-hint: <operator> --version <version> [--branch <midstream-branch>] [--dry-run]
---

## Name
medik8s-release:downstream

## Synopsis
```
/medik8s-release:downstream <operator> --version <version> [--branch <midstream-branch>] [--dry-run]
```

## Description
The `medik8s-release:downstream` command prepares a midstream (GitLab/dragonfly) operator repository for a downstream release. This includes updating `bundle-hack/update_bundle.sh` with correct version metadata, verifying that Tekton pipeline files have the right `semver-version`, and ensuring all downstream-specific files are consistent.

**Requires VPN access** to push to GitLab.

### What Gets Updated
- `bundle-hack/update_bundle.sh` — `PREVIOUS_VERSION`, `SKIP_RANGE_LOWER`, `DOCS_RHWA_VERSION`
- Verification that `.tekton/*.yaml` files have correct `semver-version` (set by `update-version.sh`)
- Verification that `Containerfile.*-bundle` has correct `CI_VERSION`

### Y-Stream vs Z-Stream
- **Y-stream**: May need a new midstream branch (e.g., `snr-0-13`), new Tekton pipeline files
- **Z-stream**: Reuses existing branch and pipeline files; only version and bundle-hack updates

## Implementation

### Phase 1: Resolve Paths and Branch
```bash
declare -A OP_REPO=(
    [SNR]=self-node-remediation [NHC]=node-healthcheck-operator
    [FAR]=fence-agents-remediation [MDR]=machine-deletion-remediation
    [NMO]=node-maintenance-operator [SBR]=storage-based-remediation
)
declare -A OP_BRANCH_PREFIX=(
    [SNR]=snr [NHC]=nhc [FAR]=far [MDR]=mdr [NMO]=nmo [SBR]=sbr
)

REPO="${OP_REPO[$operator]}"

# Discover midstream clone — do NOT assume any fixed directory layout
MIDSTREAM_DIR=$(find "$HOME" -maxdepth 6 -type d -name "${REPO}" 2>/dev/null | while read dir; do
  git -C "$dir" remote -v 2>/dev/null | grep -qi "gitlab" && echo "$dir" && break
done)
# If not found, ask the user for the path

# Derive branch if not provided
MAJOR=$(echo $version | cut -d. -f1)
MINOR=$(echo $version | cut -d. -f2)
BRANCH="${OP_BRANCH_PREFIX[$operator]}-${MAJOR}-${MINOR}"
```

### Phase 2: Verify Current State
1. Check we're on the correct branch:
   ```bash
   git -C "$MIDSTREAM_DIR" branch --show-current
   ```
2. Check that `scripts/update-version.sh` has the target version:
   ```bash
   grep '^VERSION=' "$MIDSTREAM_DIR/scripts/update-version.sh"
   ```
3. Verify Tekton `semver-version` matches:
   ```bash
   grep -A1 'name: semver-version' "$MIDSTREAM_DIR/.tekton/"*push*.yaml
   ```

### Phase 3: Update bundle-hack/update_bundle.sh
Read the current file and update:
- `PREVIOUS_VERSION` — set to the previous release version
- `SKIP_RANGE_LOWER` — set for Y-stream, comment out for first Z-stream
- `DOCS_RHWA_VERSION` — set to match RHWA release (e.g., `4.22-0`)
- `OPERATOR_COMPONENT` — verify it points to correct registry image

### Phase 4: Verify Consistency
1. Tekton `semver-version` matches target version (`v<version>`)
2. `Containerfile.*-bundle` has matching `CI_VERSION`
3. `bundle-hack/test_vars.sh` or `test_update_bundle.sh` has matching `CI_VERSION`
4. If `rpms.in.yaml` exists, verify it's up to date

### Phase 5: Report
Print a diff summary of changes and current state of all version-related fields.

If `--dry-run`, stop here. Otherwise, offer to commit and push.

## Return Value
Summary of changes made or verified, with any inconsistencies flagged.

## Examples

1. **Prepare SNR 0.13.1 Z-stream**:
   ```
   /medik8s-release:downstream SNR --version 0.13.1
   ```

2. **Prepare NHC 0.12.0 Y-stream on specific branch**:
   ```
   /medik8s-release:downstream NHC --version 0.12.0 --branch nhc-0-12
   ```

## Arguments
- `$1`: Operator short name (SNR, NHC, FAR, MDR, NMO, SBR)
- `--version <version>`: Target version (without `v` prefix)
- `--branch <branch>`: Midstream branch name (auto-derived if omitted)
- `--dry-run`: Show changes without committing

## See Also
- `/medik8s-release:version-bump` — Bump version across Tekton/Containerfile (run first)
- `/medik8s-release:fbc` — Update FBC after midstream preparation
- `/medik8s-release:verify-bundle` — Validate the downstream bundle
