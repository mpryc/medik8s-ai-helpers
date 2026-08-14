---
description: Prepare FBC catalogs for upcoming OCP versions and generate konflux-release-data pruning exceptions (RHWA-616)
argument-hint: --ocp <ocp_versions> [--snapshot <image>] [--generate-exceptions] [--dry-run]
---

## Name
medik8s-release:fbc-bootstrap

## Synopsis
```
/medik8s-release:fbc-bootstrap --ocp <ocp_versions> [--snapshot <image>...] [--generate-exceptions] [--dry-run]
```

## Description
The `medik8s-release:fbc-bootstrap` command prepares FBC catalogs for upcoming OCP versions to prevent stale/unsupported operator versions from appearing in new OCP clusters (RHWA-616). It also generates the `konflux-release-data` pruning exception YAML needed for the release.

This command should be run **immediately after each Y-stream GA release** and **before OCP release-engineering bootstraps the next OCP catalogs**.

### What it does

1. **Prepares FBC entries** for upcoming OCP versions using current supported operator versions
2. **Generates pruning exception YAML** from FBC snapshot images (per-arch + OCI index digests)
3. **Validates** that all required digests are present

### Background (RHWA-616)

OCP catalogs are bootstrapped ~5 months before GA from the previous OCP catalog. If RHWA hasn't released for that OCP version via Konflux yet, the package is considered "not opted-in" and IIB falls back to including ALL historically supported bundles — resulting in very old, unsupported versions appearing in OperatorHub.

The fix is to release corrected FBC catalogs for upcoming OCP versions before they are bootstrapped, with a pruning check exception.

## Implementation

### Phase 1: Discover Repos

```bash
# Discover rhwa-fbc repo — do NOT assume any fixed directory layout
FBC_DIR=$(find "$HOME" -maxdepth 6 -type d -name "rhwa-fbc" 2>/dev/null | while read dir; do
  [ -f "$dir/operator-versions.yaml" ] && echo "$dir" && break
done)

# Discover konflux-release-data repo
KRD_DIR=$(find "$HOME" -maxdepth 6 -type d -name "konflux-release-data" 2>/dev/null | while read dir; do
  [ -d "$dir/exceptions" ] && echo "$dir" && break
done)

if [ -z "$FBC_DIR" ]; then
  echo "ERROR: rhwa-fbc repo not found locally."
  exit 1
fi
```

### Phase 2: Identify Current Supported Versions

Read the latest release from `releases.yaml` to determine which operator versions should be in the upcoming OCP catalogs:

```bash
cd "$FBC_DIR"
git fetch origin

# Get the latest release and its operator versions
python3 -c "
import yaml
with open('releases.yaml') as f:
    data = yaml.safe_load(f)
latest = data['releases'][-1]
print(f'Latest release: {latest[\"name\"]}')
for ocp in latest['ocp']:
    print(f'  OCP {ocp[\"version\"]}:')
    for op in ocp['operators']:
        versions = [v['version'] for v in op['versions']]
        print(f'    {op[\"name\"]}: {versions}')
"
```

### Phase 3: Prepare FBC for Upcoming OCP Versions

For each target OCP version (e.g., 4.23, 5.0):

1. Check if entries already exist in `releases.yaml`
2. If not, add a new release block with the current supported versions (NOT upcoming Y-stream versions)
3. Generate the catalog: `./gen_template.sh && ./gen_fbc.sh`
4. Create or update Containerfile if needed
5. Verify with `./verify-bundle-images.sh`

### Phase 4: Generate Pruning Exception YAML

For each snapshot image provided via `--snapshot`:

```bash
# Extract per-arch digests from the OCI manifest list
extract_digests() {
    local snapshot_image="$1"
    local ocp_label="$2"

    echo "# for ocp ${ocp_label} (OCI index + 4 per-arch manifests)"

    # OCI index (manifest list) digest — the part after @
    index_digest=$(echo "$snapshot_image" | sed 's/.*@//')
    echo "- value: test.no_failed_tests:fbc-target-index-pruning-check"
    echo "  imageRef: ${index_digest}"

    # Per-arch digests
    skopeo inspect --raw "docker://${snapshot_image}" \
        | jq -r '.manifests[] | .digest' \
        | while read digest; do
            echo "- value: test.no_failed_tests:fbc-target-index-pruning-check"
            echo "  imageRef: ${digest}"
        done
}

# Generate the full exceptions file
{
    echo "---"
    for snapshot in "${SNAPSHOTS[@]}"; do
        # Extract OCP version from snapshot name (e.g., rhwa-fbc-423 -> 4.23)
        if [[ "$snapshot" =~ rhwa-fbc-([0-9])([0-9]{2}) ]]; then
            ocp_ver="${BASH_REMATCH[1]}.${BASH_REMATCH[2]}"
        fi
        extract_digests "$snapshot" "$ocp_ver"
        echo ""
    done
} > /tmp/fbc-rhwa-prod-exceptions.yaml
```

### Phase 5: Apply Exceptions (if --generate-exceptions)

```bash
if [ -n "$KRD_DIR" ]; then
    cp /tmp/fbc-rhwa-prod-exceptions.yaml "$KRD_DIR/exceptions/fbc-rhwa-prod.yaml"
    echo "Updated $KRD_DIR/exceptions/fbc-rhwa-prod.yaml"
    echo ""
    echo "Next steps:"
    echo "  1. cd $KRD_DIR"
    echo "  2. git checkout -b rhwa-fbc-pruning-exceptions-<ocp_versions>"
    echo "  3. git add exceptions/fbc-rhwa-prod.yaml"
    echo "  4. git commit -m 'Update RHWA FBC pruning exceptions for OCP <versions>'"
    echo "  5. git push origin <branch>"
    echo "  6. Create MR on gitlab.cee.redhat.com/releng/konflux-release-data"
    echo "  7. Get TWO approvals on the MR"
else
    echo "konflux-release-data repo not found locally."
    echo "Generated exceptions at: /tmp/fbc-rhwa-prod-exceptions.yaml"
    echo "Copy this to exceptions/fbc-rhwa-prod.yaml in the konflux-release-data repo."
fi
```

### Phase 6: Validation

Verify the generated exceptions:

```bash
# Count entries per OCP version (should be 5 each: 1 index + 4 arch)
entries=$(grep -c 'imageRef:' /tmp/fbc-rhwa-prod-exceptions.yaml)
ocp_count=$(grep -c '^#' /tmp/fbc-rhwa-prod-exceptions.yaml)
echo "Total entries: $entries (expected: ${ocp_count} OCP versions x 5 = $((ocp_count * 5)))"

# Verify architectures are complete
for snapshot in "${SNAPSHOTS[@]}"; do
    echo "Checking $snapshot:"
    skopeo inspect --raw "docker://${snapshot}" \
        | jq -r '.manifests[] | "\(.digest) (\(.platform.architecture))"'
done
```

Expected architectures: amd64, s390x, arm64, ppc64le (4 total).

### Phase 7: Report

```
FBC BOOTSTRAP PREPARATION
══════════════════════════

Target OCP versions: 4.23, 5.0
Source release: rhwa-4.22-1

Operator versions included:
  SNR: v0.13.1
  NHC: v0.12.1
  FAR: v0.8.1
  MDR: v0.7.1
  NMO: v5.7.1
  SBR: v0.3.1

Pruning exceptions generated:
  OCP 4.23: 5 entries (1 index + 4 arch)
    Index: sha256:2b2375ed...
    amd64: sha256:0204871e...
    s390x: sha256:40e3a70d...
    arm64: sha256:232b62ac...
    ppc64le: sha256:69263a7a...

  OCP 5.0: 5 entries (1 index + 4 arch)
    Index: sha256:cf73a241...
    amd64: sha256:2dbd08e0...
    s390x: sha256:b8c8b85d...
    arm64: sha256:64195b93...
    ppc64le: sha256:23ec4dc2...

Output: /tmp/fbc-rhwa-prod-exceptions.yaml

NEXT STEPS:
  1. Submit MR to gitlab.cee.redhat.com/releng/konflux-release-data
  2. Get TWO approvals
  3. Merge MR
  4. Verify FBC release succeeds in Konflux
```

## Return Value
Summary of FBC bootstrap preparation, generated exception content, and next steps.

## Examples

1. **Generate pruning exceptions from snapshots**:
   ```
   /medik8s-release:fbc-bootstrap --ocp 4.23,5.0 \
     --snapshot quay.io/redhat-user-workloads/rhwa-tenant/rhwa-fbc/rhwa-fbc-423@sha256:2b2375ed... \
     --snapshot quay.io/redhat-user-workloads/rhwa-tenant/rhwa-fbc/rhwa-fbc-500@sha256:cf73a241... \
     --generate-exceptions
   ```

2. **Dry-run to see what would be generated**:
   ```
   /medik8s-release:fbc-bootstrap --ocp 4.23,5.0 --dry-run
   ```

3. **Prepare FBC entries only (no exceptions)**:
   ```
   /medik8s-release:fbc-bootstrap --ocp 4.23,5.0
   ```

## Arguments
- `--ocp <ocp_versions>`: Comma-separated target OCP versions (e.g., `4.23,5.0`)
- `--snapshot <image>`: FBC snapshot image with digest (repeatable, one per OCP version)
- `--generate-exceptions`: Generate and apply the `konflux-release-data` pruning exception file
- `--dry-run`: Show what would be done without making changes

## See Also
- `/medik8s-release:fbc` — Update FBC catalog for current releases
- `/medik8s-release:audit` — Stage 8 checks pre-bootstrap FBC readiness
- FBC Management skill — "OCP Catalog Bootstrap Problem" section for full background
