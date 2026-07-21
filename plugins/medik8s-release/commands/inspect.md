---
description: Inspect what a particular RHWA/OCP release is built from — upstream commits, go/base image versions, CVE fixes, bundle contents
argument-hint: <fbc-app-or-release> [--operator <op>] [--deep]
---

## Name
medik8s-release:inspect

## Synopsis
```
/medik8s-release:inspect <fbc-app-or-release> [--operator <op>] [--deep]
```

## Description
The `medik8s-release:inspect` command provides a comprehensive view of what a particular RHWA release or FBC catalog contains. It answers: "what upstream code are we shipping, what Go version built it, what base images are used, and are there known CVEs?"

This is essential for understanding exactly what customers receive and for auditing build provenance.

### Information Gathered

For each operator in the release:

**Source Provenance:**
- Upstream GitHub tag/commit the build was made from
- `upstream-version` label from container image (e.g., `release-0.13-d6e798a`)
- Git submodule commit in midstream repo vs upstream tag
- Whether upstream tag matches what midstream is tracking

**Build Details:**
- Go version used for compilation (from Containerfile / base image)
- UBI base image version and digest
- Build date and builder version (`io.buildah.version`)
- Architecture support (x86_64, s390x, aarch64, ppc64le)

**Bundle/OLM Details:**
- CSV version, replaces, skipRange
- Operator image reference in CSV
- OCP version annotations
- Channel membership (stable, candidate)

**Security:**
- Go version vs latest approved version (cross-reference with `ocp-build-data/streams.yml`)
- Known CVEs in base image (if `--deep` and trivy available)
- FIPS compliance status

## Implementation

### Phase 1: Resolve the Release
```bash
# Option A: FBC app name (e.g., rhwa-fbc-422)
# Look up latest FBC image from Konflux
oc get snapshot -n rhwa-tenant -l "appstudio.openshift.io/application=${fbc_app}" \
    --sort-by=.metadata.creationTimestamp -o json | jq -r '.items[-1]'

# Option B: Release name (e.g., 4.22-1)
# Discover rhwa-fbc repo locally — do NOT assume any fixed directory layout
FBC_DIR=$(find "$HOME" -maxdepth 6 -type d -name "rhwa-fbc" 2>/dev/null | while read dir; do
  [ -f "$dir/operator-versions.yaml" ] && echo "$dir" && break
done)
if [ -n "$FBC_DIR" ]; then
  yq ".releases[] | select(.name == \"${release}\")" "$FBC_DIR/releases.yaml"
fi
```

### Phase 2: Extract Bundle Information from FBC
```bash
# Pull FBC image and extract catalog
FBC_IMAGE="quay.io/redhat-user-workloads/rhwa-tenant/rhwa-fbc/${fbc_app}@sha256:..."
podman create --name fbc-inspect "$FBC_IMAGE"
podman cp fbc-inspect:/configs/ /tmp/fbc-inspect/
podman rm fbc-inspect

# List all bundles and their images
for catalog in /tmp/fbc-inspect/*/catalog.yaml; do
    yq '.entries[] | select(.schema == "olm.bundle") | {name: .name, image: .image}' "$catalog"
done
```

### Phase 3: Inspect Operator Images
For each operator:
```bash
# Get operator image from bundle CSV
# Use FBC_DIR discovered in Phase 1
BUNDLE_IMG=$(yq ".operators[] | select(.name == \"${repo}\") | .versions[-1].bundle-image" \
    "$FBC_DIR/operator-versions.yaml")

# Pull bundle, extract CSV, get operator image
podman create --name bundle-tmp "$BUNDLE_IMG"
podman cp bundle-tmp:/manifests/ /tmp/bundle-manifests/
podman rm bundle-tmp
OPERATOR_IMG=$(yq '.spec.install.spec.deployments[0].spec.template.spec.containers[0].image' \
    /tmp/bundle-manifests/*.clusterserviceversion.yaml)

# Inspect operator image labels
skopeo inspect "docker://${OPERATOR_IMG}" | jq '{
    upstream_version: .Labels["upstream-version"],
    version: .Labels["version"],
    release: .Labels["release"],
    build_date: .Labels["build-date"],
    vcs_ref: .Labels["vcs-ref"],
    source: .Labels["org.opencontainers.image.source"],
    buildah_version: .Labels["io.buildah.version"],
    cpe: .Labels["cpe"]
}'
```

### Phase 4: Check Go/Base Image Versions
```bash
# Discover midstream clone — do NOT assume any fixed directory layout
MIDSTREAM_DIR=$(find "$HOME" -maxdepth 6 -type d -name "${repo}" 2>/dev/null | while read dir; do
  git -C "$dir" remote -v 2>/dev/null | grep -qi "gitlab" && echo "$dir" && break
done)

# From midstream Containerfile, extract Go and UBI base image (if clone found)
if [ -n "$MIDSTREAM_DIR" ]; then
  grep '^FROM' "$MIDSTREAM_DIR/Containerfile"
fi

# Compare with OCP-approved Go version
# Reference: https://github.com/openshift-eng/ocp-build-data/blob/openshift-4.23/streams.yml
curl -s "https://raw.githubusercontent.com/openshift-eng/ocp-build-data/openshift-4.23/streams.yml" \
    | yq '.golang.el9.image'
```

### Phase 5: Cross-Reference Upstream
```bash
# Extract short commit from upstream-version label
UPSTREAM_SHORT=$(echo "$upstream_version" | sed 's/.*-//')

# Discover upstream clone — do NOT assume any fixed directory layout
UPSTREAM_DIR=$(find "$HOME" -maxdepth 6 -type d -name "${repo}" 2>/dev/null | while read dir; do
  git -C "$dir" remote -v 2>/dev/null | grep -q "github.com/medik8s/${repo}" && echo "$dir" && break
done)

# Verify against upstream repo (if clone found, otherwise use GitHub API)
if [ -n "$UPSTREAM_DIR" ]; then
  cd "$UPSTREAM_DIR"
  git log --oneline | grep "^${UPSTREAM_SHORT}"
  git tag --points-at $(git rev-parse ${UPSTREAM_SHORT}) 2>/dev/null
else
  # Fallback: use GitHub API
  gh api repos/medik8s/${repo}/commits/${UPSTREAM_SHORT} --jq '.sha' 2>/dev/null
fi
```

### Phase 6: Generate Report
```
RHWA BUILD INSPECTION: rhwa-fbc-422 (2026-07-20)
════════════════════════════════════════════════

SNR (self-node-remediation)
  Upstream tag:      v0.13.0 (79fb792)
  Upstream-version:  release-0.13-d6e798a (MISMATCH - newer than tag)
  Midstream branch:  snr-0-13
  Semver-version:    v0.13.1
  Go version:        1.24.11 (OCP approved: 1.26.5 for 4.23)
  UBI base:          ubi9-minimal:9.6-1466
  Build date:        2026-05-27T07:15:41Z
  Architectures:     amd64, arm64, s390x, ppc64le
  CSV version:       0.13.1
  CSV replaces:      self-node-remediation.v0.13.0
  CSV skipRange:     >=0.11.0 <0.13.1
  FIPS:              compliant (UBI 9)

NHC (node-healthcheck-operator)
  ...

SUMMARY:
  Operators inspected:  6
  Version mismatches:   1 (SNR: upstream-version newer than tag)
  Go version warnings:  1 (SNR: 1.24.11 vs OCP 1.26.5)
  Missing labels:       0
```

## Return Value
Detailed build provenance report for each operator in the release.

## Examples

1. **Inspect latest 4.22 FBC build**:
   ```
   /medik8s-release:inspect rhwa-fbc-422
   ```

2. **Inspect specific operator only**:
   ```
   /medik8s-release:inspect rhwa-fbc-422 --operator SNR
   ```

3. **Deep inspection with CVE scan**:
   ```
   /medik8s-release:inspect rhwa-fbc-422 --deep
   ```

## Arguments
- `$1`: FBC application name (e.g., `rhwa-fbc-422`) or release name (e.g., `4.22-1`)
- `--operator <op>`: Inspect only this operator (SNR, NHC, FAR, MDR, NMO, SBR)
- `--deep`: Include CVE scanning (requires trivy) and full dependency analysis

## See Also
- `/medik8s-release:status` — Quick release readiness overview
- `/medik8s-release:verify-bundle` — Validate bundle correctness
- `/medik8s-release:audit` — Full release checklist with progress tracking
