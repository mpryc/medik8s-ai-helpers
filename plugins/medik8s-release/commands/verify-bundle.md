---
description: Validate bundle CSV fields, skipRange/replaces consistency, image pullability, OLM annotations
argument-hint: <operator> --version <version> [--registry <registry>]
---

## Name
medik8s-release:verify-bundle

## Synopsis
```
/medik8s-release:verify-bundle <operator> --version <version> [--registry <registry>]
```

## Description
The `medik8s-release:verify-bundle` command validates that an operator bundle is correctly built and consistent across upstream and downstream. It checks CSV fields, OLM metadata, image references, and cross-references with FBC entries.

### Checks Performed
1. **CSV fields**: `spec.version`, `spec.replaces`, `metadata.annotations.olm.skipRange`
2. **Image references**: operator image tag matches version, uses correct registry
3. **OLM annotations**: `com.redhat.openshift.versions` (OKD), display name, description
4. **Bundle image pullability**: can the bundle image be pulled and inspected
5. **FBC consistency**: does `operator-versions.yaml` match the bundle's replaces/skipRange
6. **Upstream-version label**: does the container carry the `upstream-version` label linking to source commit
7. **Upstream tag**: does `v<version>` tag exist in the upstream GitHub repo
8. **Community release**: do community images exist on `quay.io/medik8s/` for this version

## Implementation

### Phase 1: Locate and Pull Bundle
```bash
# Community bundle
skopeo inspect "docker://quay.io/medik8s/${repo}-bundle:v${version}"

# Downstream bundle (Konflux)
# Digest from update_bundle.sh or operator-versions.yaml
skopeo inspect "docker://${bundle_image}"
```

### Phase 2: Extract and Validate CSV
```bash
# Pull bundle contents (use export+tar — more reliable than podman cp for batch)
podman create --name bundle-check "${bundle_image}"
podman export bundle-check | tar -C /tmp/bundle-check/ -xf - manifests/
podman rm bundle-check

# Check CSV using python3+yaml (yq may not be installed)
CSV_FILE=$(ls /tmp/bundle-check/manifests/*.clusterserviceversion.yaml)
python3 -c "
import yaml
with open('${CSV_FILE}') as f:
    csv = yaml.safe_load(f)
print('version:', csv['spec']['version'])
print('replaces:', csv['spec'].get('replaces', 'null'))
print('skipRange:', csv['metadata']['annotations'].get('olm.skipRange', 'null'))
deploys = csv['spec']['install']['spec']['deployments']
print('image:', deploys[0]['spec']['template']['spec']['containers'][0]['image'])
print('disconnected:', csv['metadata']['annotations'].get('features.operators.openshift.io/disconnected', 'null'))
print('relatedImages:', len(csv['spec'].get('relatedImages', [])))
"
```

### Phase 3: Cross-Reference with FBC
Discover the FBC repo locally. **Do NOT assume any fixed directory layout.**
```bash
# Discover rhwa-fbc repo
FBC_DIR=$(find "$HOME" -maxdepth 6 -type d -name "rhwa-fbc" 2>/dev/null | while read dir; do
  [ -f "$dir/operator-versions.yaml" ] && echo "$dir" && break
done)

# If found, cross-reference
if [ -n "$FBC_DIR" ]; then
  python3 -c "
import yaml
with open('$FBC_DIR/operator-versions.yaml') as f:
    data = yaml.safe_load(f)
for op in data['operators']:
    if op['name'] == '${repo}':
        for v in op['versions']:
            if v['name'] == '${version}':
                print(v)
                break
"
fi
```
If the FBC repo is not found locally, skip this check and note it as "N/A — no local clone found".

Verify:
- `replaces` in CSV matches `replaces` in operator-versions.yaml
- `skipRange` in CSV matches `skip-range` in operator-versions.yaml
- Bundle image digest in operator-versions.yaml is pullable

### Phase 4: Check Operator Image
```bash
# Inspect operator image for upstream-version label
skopeo inspect "docker://quay.io/redhat-user-workloads/rhwa-tenant/${repo}/${prefix}-operator-${minor}:latest" 2>/dev/null \
    | jq '.Labels["upstream-version"]'
```

### Phase 5: Check Upstream Tag and Community Release
Verify that the upstream release steps were completed for this version.
```bash
# Check upstream tag exists
gh api repos/medik8s/${repo}/git/refs/tags/v${version} 2>/dev/null && echo "TAG EXISTS" || echo "TAG MISSING"

# Check community images exist on quay.io/medik8s
for suffix in "" "-bundle" "-catalog"; do
    skopeo inspect "docker://quay.io/medik8s/${repo}${suffix}:v${version}" > /dev/null 2>&1 \
        && echo "OK: ${repo}${suffix}:v${version}" \
        || echo "MISSING: ${repo}${suffix}:v${version}"
done
```

**Why this matters:** The downstream Z-stream version (e.g., 0.13.1) is set in the
midstream bundle-hack/CSV and built by Konflux. But the corresponding upstream steps
— tagging the release branch, building community images, creating community-operators
PRs — must also be completed. Without them, the community and downstream versions
diverge, and community users cannot access fixes shipped in the Z-stream.

### Phase 6: Report
```
BUNDLE VERIFICATION: ${repo} v${version}
  CSV version:        PASS (0.13.1)
  CSV replaces:       PASS (self-node-remediation.v0.13.0)
  CSV skipRange:      PASS (>=0.11.0 <0.13.1)
  Operator image:     PASS (quay.io/medik8s/self-node-remediation:v0.13.1)
  Bundle pullable:    PASS
  FBC replaces match: PASS
  FBC skipRange match:PASS
  Upstream-version:   PASS (release-0.13-d6e798a)
  Upstream tag:       PASS/FAIL (v0.13.1 exists/missing on github.com/medik8s/${repo})
  Community images:   PASS/FAIL (quay.io/medik8s/${repo}[-bundle|-catalog]:v${version})
  Overall:            VERIFIED / ISSUES FOUND
```

**Severity levels:**
- Upstream tag or community images **MISSING** → report as **WARN** (bundle itself is valid,
  but the upstream release process is incomplete — flag for follow-up)
- CSV/FBC field mismatches → report as **FAIL** (bundle is broken)

## Return Value
Verification result table with PASS/FAIL per check.

## Examples

1. **Verify SNR 0.13.1 bundle**:
   ```
   /medik8s-release:verify-bundle SNR --version 0.13.1
   ```

2. **Verify against downstream registry**:
   ```
   /medik8s-release:verify-bundle NHC --version 0.12.0 --registry registry.redhat.io
   ```

## Arguments
- `$1`: Operator short name (SNR, NHC, FAR, MDR, NMO, SBR)
- `--version <version>`: Version to verify (without `v` prefix)
- `--registry <registry>`: Override registry (default: auto-detect from FBC)

## See Also
- `/medik8s-release:fbc` — Update FBC catalog
- `/medik8s-release:status` — Check overall release readiness
- `/medik8s-release:inspect` — Deep inspection of what a build contains
