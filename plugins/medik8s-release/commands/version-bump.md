---
description: Bump version across midstream files (Tekton semver-version, Containerfile CI_VERSION, bundle-hack test scripts)
argument-hint: <operator> --version <version> [--branch <midstream-branch>]
---

## Name
medik8s-release:version-bump

## Synopsis
```
/medik8s-release:version-bump <operator> --version <version> [--branch <midstream-branch>]
```

## Description
The `medik8s-release:version-bump` command updates the version across all version-sensitive files in a midstream operator repository. It wraps the `scripts/update-version.sh` script that each midstream repo provides.

### Files Updated
- `.tekton/*.yaml` — `semver-version` parameter (e.g., `v0.13.0` → `v0.13.1`)
- `.tekton/*.yaml` — `upstream-minor-version` parameter (e.g., `13`)
- `Containerfile.*-bundle` — `ENV CI_VERSION` line
- `bundle-hack/test_vars.sh` or `bundle-hack/test_update_bundle.sh` — `export CI_VERSION=`

### When to Use
- **Before every release** (both Y-stream and Z-stream)
- Must be run **before** `/medik8s-release:downstream`
- The `.tekton/` files hardcode the full version (e.g., `v0.13.1`), so this must be bumped for every patch

## Implementation

### Phase 1: Locate Midstream Repo
```bash
declare -A OP_REPO=(
    [SNR]=self-node-remediation [NHC]=node-healthcheck-operator
    [FAR]=fence-agents-remediation [MDR]=machine-deletion-remediation
    [NMO]=node-maintenance-operator [SBR]=storage-based-remediation
)
REPO="${OP_REPO[$operator]}"

# Discover midstream clone — do NOT assume any fixed directory layout
MIDSTREAM_DIR=$(find "$HOME" -maxdepth 6 -type d -name "${REPO}" 2>/dev/null | while read dir; do
  git -C "$dir" remote -v 2>/dev/null | grep -qi "gitlab" && echo "$dir" && break
done)
# If not found, ask the user for the path
```

### Phase 2: Update VERSION in Script
Edit `scripts/update-version.sh` to set the new version:
```bash
# Change: VERSION="0.13.0" → VERSION="0.13.1"
sed -i "s/^VERSION=.*/VERSION=\"${version}\"/" "$MIDSTREAM_DIR/scripts/update-version.sh"
```

### Phase 3: Run the Script
```bash
cd "$MIDSTREAM_DIR"
./scripts/update-version.sh
```

### Phase 4: Verify
```bash
# Check all Tekton files got updated
grep -A1 'name: semver-version' .tekton/*push*.yaml
grep -A1 'name: semver-version' .tekton/*pull-request*.yaml

# Check Containerfiles
grep 'CI_VERSION' Containerfile.*-bundle

# Check test scripts
grep 'CI_VERSION' bundle-hack/test_*.sh 2>/dev/null
```

### Phase 5: Report
Print summary of files changed and current version values.

## Return Value
List of files updated with before/after version values.

## Examples

1. **Bump SNR to 0.13.1**:
   ```
   /medik8s-release:version-bump SNR --version 0.13.1
   ```

2. **Bump FAR to 0.8.0 on specific branch**:
   ```
   /medik8s-release:version-bump FAR --version 0.8.0 --branch far-0-8
   ```

## Arguments
- `$1`: Operator short name (SNR, NHC, FAR, MDR, NMO, SBR)
- `--version <version>`: Target version (without `v` prefix)
- `--branch <branch>`: Midstream branch (auto-derived if omitted)

## See Also
- `/medik8s-release:downstream` — Full midstream preparation (run after version bump)
- `/medik8s-release:verify-bundle` — Validate the resulting bundle
