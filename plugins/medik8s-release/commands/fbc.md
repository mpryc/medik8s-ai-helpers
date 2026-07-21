---
description: Update FBC catalog (operator-versions.yaml, releases.yaml, generate templates and catalogs, verify bundle images)
argument-hint: --release <release-name> [--verify-only] [--dry-run]
---

## Name
medik8s-release:fbc

## Synopsis
```
/medik8s-release:fbc --release <release-name> [--verify-only] [--dry-run]
```

## Description
The `medik8s-release:fbc` command manages the RHWA File-Based Catalog (FBC). It updates `operator-versions.yaml` and `releases.yaml`, generates catalog templates and catalogs, and verifies bundle image availability.

RHWA uses FBC exclusively — IIBs (Index Image Bundles) are no longer generated. The FBC pipeline produces catalog images that are pushed to Konflux and mirrored to `registry.redhat.io`.

### Key Files in rhwa-fbc
| File | Purpose |
|------|---------|
| `releases.yaml` | Maps RHWA releases to OCP versions and operator versions |
| `operator-versions.yaml` | Master list: version, replaces, skipRange, bundle-image, update-bundle flag |
| `update_bundle.sh` | Latest Konflux-built bundle digests (auto-updated by MintMaker) |
| `gen_template.sh` | Generates `rhwa-template.yaml` from releases + operator-versions |
| `gen_fbc.sh` | Creates per-OCP catalogs from templates using `opm render-template` |
| `verify-bundle-images.sh` | Validates bundle images are pullable and use correct registry |
| `Containerfile.rhwa-catalog-v<OCP>` | Per-OCP Containerfile for catalog image |

### Image Registry Rules
- **Staging** (pre-GA): `quay.io/redhat-user-workloads/rhwa-tenant/` with `update-bundle: true` in operator-versions.yaml
- **Released** (post-GA): `registry.redhat.io/workload-availability/` — remove `update-bundle: true`

## Implementation

### Phase 1: Locate FBC Repo
Discover the FBC repo locally. **Do NOT assume any fixed directory layout.**
```bash
# Discover rhwa-fbc repo by looking for operator-versions.yaml
FBC_DIR=$(find "$HOME" -maxdepth 6 -type d -name "rhwa-fbc" 2>/dev/null | while read dir; do
  [ -f "$dir/operator-versions.yaml" ] && echo "$dir" && break
done)

if [ -z "$FBC_DIR" ]; then
  echo "ERROR: rhwa-fbc repo not found locally. Clone it first."
  exit 1
fi

cd "$FBC_DIR"
git fetch origin
git status
```

### Phase 2: Update operator-versions.yaml
For each operator being released, add or update the version entry:

```yaml
operators:
  - name: self-node-remediation
    versions:
      - name: 0.13.1
        replaces: 0.13.0
        skip-range: ">=0.11.0 <0.13.1"
        bundle-image: quay.io/redhat-user-workloads/rhwa-tenant/self-node-remediation/snr-bundle-0-13@sha256:...
        update-bundle: true    # staging — remove after GA
```

The bundle-image digest comes from `update_bundle.sh` (maintained by MintMaker/Renovate).

### Phase 3: Update releases.yaml
Add the version to the appropriate release block:

```yaml
releases:
  - name: "4.22-1"
    ocp:
      - version: "4.22"
        operators:
          - name: "self-node-remediation"
            versions:
              - version: "0.13.0"
              - version: "0.13.1"    # add new version
```

### Phase 4: Generate Catalog
```bash
# Source latest bundle digests
source update_bundle.sh

# Generate template
./gen_template.sh

# Generate catalog
./gen_fbc.sh
```

### Phase 5: Verify
```bash
# Verify bundle images are pullable and use correct registry
./verify-bundle-images.sh
```

Key checks:
- All `update-bundle: true` entries use `quay.io` registry
- All other entries use `registry.redhat.io` registry
- All images are pullable
- Version labels match expected versions

### Phase 6: Report
Print summary:
- Operators/versions added or updated
- Bundle image digests used
- Catalog generation status
- Verification results

## Return Value
Summary of FBC changes and verification results.

## Examples

1. **Update FBC for RHWA 4.22-1 release**:
   ```
   /medik8s-release:fbc --release 4.22-1
   ```

2. **Verify only (no changes)**:
   ```
   /medik8s-release:fbc --release 4.22-0 --verify-only
   ```

3. **Dry-run**:
   ```
   /medik8s-release:fbc --release 4.22-1 --dry-run
   ```

## Arguments
- `--release <release-name>`: RHWA release name (e.g., `4.22-0`, `4.22-1`)
- `--verify-only`: Only run verification, don't make changes
- `--dry-run`: Show what would change without modifying files

## See Also
- `/medik8s-release:downstream` — Prepare midstream repos (run before FBC update)
- `/medik8s-release:verify-bundle` — Validate individual operator bundles
- `/medik8s-release:status` — Check overall release readiness
