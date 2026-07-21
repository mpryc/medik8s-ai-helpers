---
description: Audit release progress — show checklist of completed and remaining steps for a Y-stream or Z-stream release
argument-hint: --version <version> --ocp <ocp_version> [--type y|z] [--operator <op>]
---

## Name
medik8s-release:audit

## Synopsis
```
/medik8s-release:audit --version <version> --ocp <ocp_version> [--type y|z] [--operator <op>]
```

## Description
The `medik8s-release:audit` command provides a full release checklist with live progress tracking. It examines the actual state of upstream repos, midstream repos, FBC, Konflux, and community catalogs to determine what has been done and what remains.

This command answers: "where are we in the release process and what do I need to do next?"

### Checklist Stages

The audit checks the following stages in order. Each stage can be: DONE, IN PROGRESS, NOT STARTED, or BLOCKED.

**1. Upstream Preparation**
- [ ] Code merged to main / release branch
- [ ] CI passing on release branch
- [ ] Release tag created (`v<version>`)
- [ ] GitHub Release exists (draft or published)

**2. Upstream Images**
- [ ] Operator image on `quay.io/medik8s/<op>:v<version>`
- [ ] Bundle image on `quay.io/medik8s/<op>-bundle:v<version>`
- [ ] Catalog image on `quay.io/medik8s/<op>-catalog:v<version>`

**3. Community Catalog PRs**
- [ ] K8s community PR created (skip MDR)
- [ ] K8s community PR merged
- [ ] OKD community PR created
- [ ] OKD community PR merged

**4. Midstream Preparation**
- [ ] Midstream branch exists (Y-stream) / is correct (Z-stream)
- [ ] `scripts/update-version.sh` has target version
- [ ] `.tekton/*.yaml` has correct `semver-version`
- [ ] `bundle-hack/update_bundle.sh` has correct `PREVIOUS_VERSION`
- [ ] `bundle-hack/update_bundle.sh` has correct `SKIP_RANGE_LOWER`

**5. Konflux Builds**
- [ ] Operator image built in `quay.io/redhat-user-workloads/rhwa-tenant/`
- [ ] Bundle image built in `quay.io/redhat-user-workloads/rhwa-tenant/`
- [ ] Bundle image digest from `operator-versions.yaml` is pullable

**6. FBC Catalog**
- [ ] Version in `operator-versions.yaml`
- [ ] Version in `releases.yaml`
- [ ] Catalog generated
- [ ] Bundle images verified (`verify-bundle-images.sh`)
- [ ] `update-bundle: true` set (pre-GA) / removed (post-GA)

**7. Release & Errata**
- [ ] Konflux Release CRs created
- [ ] Downstream tags created
- [ ] Errata advisory exists
- [ ] Images mirrored to `registry.redhat.io`

## Implementation

### Phase 1: Determine Release Scope
```bash
# Map operator to all required info
declare -A OP_REPO=( ... )
declare -A OP_BRANCH_PREFIX=( ... )

# If --type not specified, infer from version:
# x.y.0 = Y-stream, x.y.z (z>0) = Z-stream
PATCH=$(echo $version | cut -d. -f3)
[[ "$PATCH" == "0" ]] && TYPE="y" || TYPE="z"
```

### Phase 2: Check Each Stage
For each operator (or single if `--operator` specified):

#### Stage 1: Upstream
```bash
# Tag exists?
gh api repos/medik8s/${repo}/git/refs/tags/v${version} 2>/dev/null && echo "DONE" || echo "NOT STARTED"

# GitHub Release?
gh release view v${version} --repo medik8s/${repo} --json isDraft,isPrerelease 2>/dev/null

# Release branch? (Y-stream only)
gh api repos/medik8s/${repo}/branches --jq ".[].name" | grep "^release-${major}.${minor}$"
```

#### Stage 2: Images
```bash
for suffix in "" "-bundle" "-catalog"; do
    curl -sf "https://quay.io/api/v1/repository/medik8s/${repo}${suffix}/tag/?specificTag=v${version}" \
        | jq -e '.tags | length > 0' > /dev/null 2>&1 && echo "DONE" || echo "NOT STARTED"
done
```

#### Stage 3: Community PRs
```bash
# K8s
gh pr list --repo k8s-operatorhub/community-operators \
    --head "add-${repo}-${version}-k8s" --state all --json state,url

# OKD
gh pr list --repo redhat-openshift-ecosystem/community-operators-prod \
    --head "add-${repo}-${version}-okd-${ocp_version}" --state all --json state,url
```

#### Stage 4: Midstream
Discover the midstream clone. **Do NOT assume any fixed directory layout.**
```bash
# Discover midstream clone by repo name + GitLab remote
MIDSTREAM_DIR=$(find "$HOME" -maxdepth 6 -type d -name "${repo}" 2>/dev/null | while read dir; do
  git -C "$dir" remote -v 2>/dev/null | grep -qi "gitlab" && echo "$dir" && break
done)

if [ -n "$MIDSTREAM_DIR" ]; then
  # Check version in update-version.sh
  grep "^VERSION=" "$MIDSTREAM_DIR/scripts/update-version.sh"
  # Check Tekton semver
  grep -A1 'name: semver-version' "$MIDSTREAM_DIR/.tekton/"*push*.yaml | head -2
  # Check bundle-hack
  grep 'PREVIOUS_VERSION=' "$MIDSTREAM_DIR/bundle-hack/update_bundle.sh"
  grep 'SKIP_RANGE_LOWER=' "$MIDSTREAM_DIR/bundle-hack/update_bundle.sh"
fi
```
If the midstream clone is not found locally, skip this stage and report it as "N/A — no local clone found".

#### Stage 5: Konflux
```bash
# Check if downstream images exist (use quay.io API — no auth needed, faster than skopeo)
for type in operator bundle; do
    component="${prefix}-${type}-${minor}"
    result=$(curl -sf "https://quay.io/api/v1/repository/redhat-user-workloads/rhwa-tenant%2F${repo}%2F${component}/tag/?limit=1&onlyActiveTags=true" \
        | jq -r '.tags[0].last_modified // "not found"')
    echo "${component}: ${result}"
done

# Verify bundle image digests from operator-versions.yaml are actually pullable
# This confirms the pinned @sha256: reference in FBC points to a real, accessible image
img=$(python3 -c "
import yaml
with open('$FBC_DIR/operator-versions.yaml') as f:
    data = yaml.safe_load(f)
for operator in data['operators']:
    if operator['name'] == '$repo':
        for v in operator['versions']:
            if v['name'] == '$version':
                print(v.get('bundle-image', ''))
                break
")
if [ -n "$img" ]; then
    skopeo inspect --no-tags "docker://${img}" >/dev/null 2>&1 \
        && echo "Bundle digest VERIFIED: pullable" \
        || echo "Bundle digest FAILED: not pullable"
fi
```

#### Stage 6: FBC
Discover the FBC repo locally. **Do NOT assume any fixed directory layout.**
```bash
# Discover rhwa-fbc repo
FBC_DIR=$(find "$HOME" -maxdepth 6 -type d -name "rhwa-fbc" 2>/dev/null | while read dir; do
  [ -f "$dir/operator-versions.yaml" ] && echo "$dir" && break
done)

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
If the FBC repo is not found locally, skip this stage and report it as "N/A — no local clone found".

#### Stage 7: Release & Errata
```bash
# Check downstream tags (uses MIDSTREAM_DIR discovered in Stage 4)
if [ -n "$MIDSTREAM_DIR" ]; then
  git -C "$MIDSTREAM_DIR" tag -l "v${version}"
fi

# Konflux releases (requires oc login)
oc get release -n rhwa-tenant -l "operator=${prefix}" --sort-by=.metadata.creationTimestamp -o json 2>/dev/null | jq '.items[-1].metadata.name'
```

### Phase 3: Generate Checklist Report
```
RELEASE AUDIT: RHWA 4.22-1 (Z-stream)
══════════════════════════════════════

SNR v0.13.1 (replaces 0.13.0)
┌─────────────────────────────────────────────────────────┐
│ 1. Upstream Preparation                                 │
│    [x] Release tag v0.13.1                              │
│    [x] GitHub Release (published)                       │
│                                                         │
│ 2. Upstream Images                                      │
│    [x] quay.io/medik8s/self-node-remediation:v0.13.1    │
│    [x] quay.io/medik8s/snr-bundle:v0.13.1              │
│    [x] quay.io/medik8s/snr-catalog:v0.13.1             │
│                                                         │
│ 3. Community Catalog PRs                                │
│    [x] K8s PR #847 (merged)                             │
│    [ ] OKD PR — NOT STARTED                             │
│                                                         │
│ 4. Midstream Preparation                                │
│    [x] Branch snr-0-13 (semver: v0.13.1)                │
│    [x] PREVIOUS_VERSION=0.13.0                          │
│    [!] SKIP_RANGE_LOWER still set (should be commented  │
│        out for first Z-stream)                          │
│                                                         │
│ 5. Konflux Builds                                       │
│    [x] Operator image built (2026-07-20)                │
│    [x] Bundle image built (2026-07-20)                  │
│                                                         │
│ 6. FBC Catalog                                          │
│    [x] In operator-versions.yaml                        │
│    [x] In releases.yaml                                 │
│    [ ] update-bundle: true (not yet released to prod)   │
│                                                         │
│ 7. Release & Errata                                     │
│    [ ] Konflux Release CR — NOT STARTED                 │
│    [ ] Downstream tag — NOT STARTED                     │
│    [ ] Errata advisory — NOT STARTED                    │
└─────────────────────────────────────────────────────────┘

NEXT STEPS:
  1. Create OKD community PR for SNR 0.13.1
  2. Fix SKIP_RANGE_LOWER in bundle-hack (comment out for Z-stream)
  3. Create Konflux Release CRs (use dragonfly/tools/releases/create_release.sh)
  4. Tag downstream repos (use dragonfly/tools/releases/tag_downstream.sh)

NHC v0.12.1 (replaces 0.12.0)
  ...
```

### Phase 4: Provide Actionable Next Steps
Based on the first incomplete stage for each operator, suggest the exact command or action needed.

## Return Value
Formatted checklist with progress indicators and actionable next steps.

## Examples

1. **Full audit for 4.22-1 patch release**:
   ```
   /medik8s-release:audit --version 0.13.1 --ocp 4.22
   ```

2. **Audit single operator**:
   ```
   /medik8s-release:audit --version 0.13.1 --ocp 4.22 --operator SNR
   ```

3. **Y-stream audit**:
   ```
   /medik8s-release:audit --version 0.13.0 --ocp 4.22 --type y
   ```

## Arguments
- `--version <version>`: Target version (without `v` prefix)
- `--ocp <ocp_version>`: Target OCP version (e.g., 4.22)
- `--type y|z`: Release type (auto-detected from version if omitted)
- `--operator <op>`: Audit only this operator (SNR, NHC, FAR, MDR, NMO, SBR)

## See Also
- `/medik8s-release:status` — Quick readiness check (less detailed than audit)
- `/medik8s-release:inspect` — Deep inspection of what a build contains
- `/medik8s-release:upstream` — Execute upstream release steps
- `/medik8s-release:downstream` — Execute midstream preparation steps
