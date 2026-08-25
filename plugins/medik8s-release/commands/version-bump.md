---
description: Bump version across midstream files (Tekton semver-version, Containerfile CI_VERSION, bundle-hack test scripts)
argument-hint: <operator> --version <version> [--branch <midstream-branch>]
---

## Name
medik8s-release:version-bump

## Synopsis
```
/medik8s-release:version-bump <operator> --version <version> [--ocp-version <ocp-version>] [--branch <midstream-branch>]
```

## Description
The `medik8s-release:version-bump` command updates the version across all version-sensitive files in a midstream operator repository. It wraps the `scripts/update-version.sh` script that each midstream repo provides.

### Files Updated
- `.tekton/*.yaml` — `semver-version` parameter (e.g., `v0.13.0` → `v0.13.1`)
- `.tekton/*.yaml` — `upstream-minor-version` parameter (e.g., `13`)
- `.tekton/*.yaml` — `rhwa-url` parameter (points to OCP version docs, e.g., `5.0`)
- `Containerfile.*-bundle` — `ENV CI_VERSION` line
- `bundle-hack/test_vars.sh` or `bundle-hack/test_update_bundle.sh` — `export CI_VERSION=`

**For Y-stream releases only** (e.g., 0.13.x → 0.14.0):
- `.tekton/*.yaml` — Filenames renamed from `*-0-13-*.yaml` → `*-0-14-*.yaml`
- `.tekton/*.yaml` — All internal references updated (component names, pipeline names, branch triggers, etc.)

### When to Use
- **Before every release** (both Y-stream and Z-stream)
- Must be run **before** `/medik8s-release:downstream`
- The `.tekton/` files hardcode the full version (e.g., `v0.13.1`), so this must be bumped for every patch
- **Always prompt for OCP version** - the `--ocp-version` parameter is required to update the rhwa-url documentation links

### RHWA to OCP Version Mapping
The rhwa-url points to OpenShift Container Platform documentation. Common mappings:
- RHWA 5.0 (SNR 0.14.x) → OCP 5.0
- RHWA 4.22 (SNR 0.13.x) → OCP 4.22-0
- RHWA 4.21 (SNR 0.12.x) → OCP 4.21-0

**Always verify the correct OCP version with the user** - do not assume the mapping.

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

### Phase 3: Determine Release Type and Branch
```bash
# Extract current minor version from existing branch or files
CURRENT_MINOR=$(grep -oP 'snr-0-\K\d+' <<< "$CURRENT_BRANCH" || echo "")
NEW_MINOR="${version#*.}" && NEW_MINOR="${NEW_MINOR%%.*}"

# Y-stream release if minor version changes
if [[ "$NEW_MINOR" != "$CURRENT_MINOR" ]]; then
  IS_Y_STREAM=true
  NEW_BRANCH="${operator,,}-0-${NEW_MINOR}"
  git checkout -b "$NEW_BRANCH"
else
  IS_Y_STREAM=false
fi
```

### Phase 4: Run the Script
```bash
cd "$MIDSTREAM_DIR"
./scripts/update-version.sh
```

### Phase 5: Y-Stream Additional Steps
**Only for Y-stream releases** (when minor version changes):

```bash
if [[ "$IS_Y_STREAM" == "true" ]]; then
  # Rename Tekton files
  cd .tekton
  for old_file in *-0-${CURRENT_MINOR}-*.yaml; do
    new_file="${old_file//-0-${CURRENT_MINOR}-/-0-${NEW_MINOR}-}"
    git mv "$old_file" "$new_file"
  done
  cd ..
  
  # Update all internal references in Tekton files
  for file in .tekton/*.yaml; do
    # Branch triggers
    sed -i "s/snr-0-${CURRENT_MINOR}/snr-0-${NEW_MINOR}/g" "$file"
    
    # File path references
    sed -i "s/\.tekton\/snr-bundle-0-${CURRENT_MINOR}-/\.tekton\/snr-bundle-0-${NEW_MINOR}-/g" "$file"
    sed -i "s/\.tekton\/snr-operator-0-${CURRENT_MINOR}-/\.tekton\/snr-operator-0-${NEW_MINOR}-/g" "$file"
    
    # Component names
    sed -i "s/component: snr-bundle-0-${CURRENT_MINOR}/component: snr-bundle-0-${NEW_MINOR}/g" "$file"
    sed -i "s/component: snr-operator-0-${CURRENT_MINOR}/component: snr-operator-0-${NEW_MINOR}/g" "$file"
    
    # Pipeline names
    sed -i "s/name: snr-bundle-0-${CURRENT_MINOR}-/name: snr-bundle-0-${NEW_MINOR}-/g" "$file"
    sed -i "s/name: snr-operator-0-${CURRENT_MINOR}-/name: snr-operator-0-${NEW_MINOR}-/g" "$file"
    
    # Service account names
    sed -i "s/build-pipeline-snr-bundle-0-${CURRENT_MINOR}/build-pipeline-snr-bundle-0-${NEW_MINOR}/g" "$file"
    sed -i "s/build-pipeline-snr-operator-0-${CURRENT_MINOR}/build-pipeline-snr-operator-0-${NEW_MINOR}/g" "$file"
    
    # Image URLs
    sed -i "s/snr-bundle-0-${CURRENT_MINOR}:/snr-bundle-0-${NEW_MINOR}:/g" "$file"
    sed -i "s/snr-operator-0-${CURRENT_MINOR}:/snr-operator-0-${NEW_MINOR}:/g" "$file"
  done
fi
```

### Phase 6: Update RHWA Documentation URL
**Required for all releases**: Update the `rhwa-url` parameter to match the target OCP version.

```bash
# Prompt user for OCP version if not provided
if [[ -z "$OCP_VERSION" ]]; then
  echo "What OCP version does RHWA ${version} target? (e.g., 5.0, 4.22-0)"
  read OCP_VERSION
fi

# Update rhwa-url in all Tekton files
for file in .tekton/*.yaml; do
  # Find current OCP version in URL
  CURRENT_OCP=$(grep -oP 'workload_availability_for_red_hat_openshift/\K[0-9.]+(-[0-9]+)?' "$file" | head -1)
  
  if [[ -n "$CURRENT_OCP" ]]; then
    sed -i "s|workload_availability_for_red_hat_openshift/${CURRENT_OCP}|workload_availability_for_red_hat_openshift/${OCP_VERSION}|g" "$file"
    echo "Updated rhwa-url: ${CURRENT_OCP} → ${OCP_VERSION} in $file"
  fi
done
```

### Phase 7: Verify
```bash
# Check all Tekton files got updated
grep -A1 'name: semver-version' .tekton/*push*.yaml
grep -A1 'name: semver-version' .tekton/*pull-request*.yaml

# Check Containerfiles
grep 'CI_VERSION' Containerfile.*-bundle

# Check test scripts
grep 'CI_VERSION' bundle-hack/test_*.sh 2>/dev/null
```

### Phase 8: Report
Print summary of files changed and current version values, including:
- Version updates (semver-version, upstream-minor-version, CI_VERSION)
- RHWA URL updates (old OCP version → new OCP version)
- For Y-stream: Tekton file renames and internal reference updates

## Return Value
List of files updated with before/after version values.

## Examples

1. **Z-stream bump (patch release)**:
   ```
   /medik8s-release:version-bump SNR --version 0.13.1 --ocp-version 4.22-0
   ```
   Updates version on current branch, updates rhwa-url to 4.22-0

2. **Y-stream bump (minor release)**:
   ```
   /medik8s-release:version-bump SNR --version 0.14.0 --ocp-version 5.0
   ```
   Creates new branch `snr-0-14`, renames Tekton files, updates all references, sets rhwa-url to 5.0

3. **Manual branch specification**:
   ```
   /medik8s-release:version-bump FAR --version 0.8.0 --ocp-version 4.23-0 --branch far-0-8
   ```

## Arguments
- `$1`: Operator short name (SNR, NHC, FAR, MDR, NMO, SBR)
- `--version <version>`: Target version (without `v` prefix)
- `--ocp-version <ocp-version>`: OCP documentation version (e.g., `5.0`, `4.22-0`). **REQUIRED** - used to update rhwa-url in Tekton files
- `--branch <branch>`: Midstream branch (auto-derived if omitted; for Y-stream releases, a new branch is created)

## See Also
- `/medik8s-release:downstream` — Full midstream preparation (run after version bump)
- `/medik8s-release:verify-bundle` — Validate the resulting bundle
