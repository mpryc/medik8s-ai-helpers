---
description: Check release readiness across all operators (upstream tags, images, community PRs, midstream state, FBC entries)
argument-hint: [--version <version>] [--operator <op>]
---

## Name
medik8s-release:status

## Synopsis
```
/medik8s-release:status [--version <version>] [--operator <op>]
```

## Description
The `medik8s-release:status` command provides an overview of release readiness across all medik8s operators. It checks upstream tags, container images on quay.io, community catalog PRs, midstream branch state, and FBC catalog entries.

### Information Gathered
For each operator:
- **Upstream**: latest tag, release branch existence, GitHub Release status (draft/published)
- **Images**: whether operator, bundle, and catalog images exist on `quay.io/medik8s` for the target version
- **Community**: open/merged PRs on `k8s-operatorhub/community-operators` and `redhat-openshift-ecosystem/community-operators-prod`
- **Midstream**: current branch, latest commit, `semver-version` in Tekton files
- **FBC**: whether version appears in `operator-versions.yaml`, whether `update-bundle: true` is set
- **Downstream**: Konflux build status in `quay.io/redhat-user-workloads/rhwa-tenant/`

## Implementation

### Phase 1: Determine Scope
1. Parse `--version` and `--operator` arguments
2. If `--operator` is provided, check only that operator
3. If `--version` is provided, check that specific version across operators
4. Default: check all 6 GA operators (SNR, NHC, FAR, MDR, NMO, SBR)

### Operator Mapping
```bash
declare -A OP_REPO=(
    [SNR]=self-node-remediation
    [NHC]=node-healthcheck-operator
    [FAR]=fence-agents-remediation
    [MDR]=machine-deletion-remediation
    [NMO]=node-maintenance-operator
    [SBR]=storage-based-remediation
)
declare -A OP_BRANCH_PREFIX=(
    [SNR]=snr [NHC]=nhc [FAR]=far [MDR]=mdr [NMO]=nmo [SBR]=sbr
)
```

### Phase 2: Check Upstream (GitHub)
For each operator:
```bash
# Latest tags
gh api repos/medik8s/${repo}/tags --jq '.[].name' | head -5

# GitHub Releases
gh release list --repo medik8s/${repo} --limit 3

# Release branch
gh api repos/medik8s/${repo}/branches --jq '.[].name' | grep "^release-"
```

### Phase 3: Check Images (quay.io)
```bash
# Community images
for suffix in "" "-bundle" "-catalog"; do
    curl -s "https://quay.io/api/v1/repository/medik8s/${repo}${suffix}/tag/?specificTag=v${version}" | jq '.tags | length'
done

# Downstream images (requires auth)
skopeo inspect "docker://quay.io/redhat-user-workloads/rhwa-tenant/${repo}/${prefix}-operator-${minor}:latest" 2>/dev/null
```

### Phase 4: Check Community PRs
```bash
# K8s community
gh pr list --repo k8s-operatorhub/community-operators --search "${repo}" --state all --limit 3 --json number,title,state,url

# OKD/OCP community
gh pr list --repo redhat-openshift-ecosystem/community-operators-prod --search "${repo}" --state all --limit 3 --json number,title,state,url
```

### Phase 5: Check Midstream (GitLab — requires VPN)
For each operator, discover and read the local clone. **Do NOT assume any fixed directory layout.** Search dynamically:
```bash
# Discover midstream clone by repo name + GitLab remote
MIDSTREAM_DIR=$(find "$HOME" -maxdepth 6 -type d -name "${repo}" 2>/dev/null | while read dir; do
  git -C "$dir" remote -v 2>/dev/null | grep -qi "gitlab" && echo "$dir" && break
done)

# If found, check branch and version
if [ -n "$MIDSTREAM_DIR" ]; then
  git -C "$MIDSTREAM_DIR" branch --show-current
  grep 'semver-version' "$MIDSTREAM_DIR/.tekton/"*push*.yaml | head -1
fi
```
If the midstream clone is not found locally, **skip this check** and report it as "N/A — no local clone found".

### Phase 6: Check FBC
Discover the FBC repo locally. **Do NOT assume any fixed directory layout.**
```bash
# Discover rhwa-fbc repo
FBC_DIR=$(find "$HOME" -maxdepth 6 -type d -name "rhwa-fbc" 2>/dev/null | while read dir; do
  [ -f "$dir/operator-versions.yaml" ] && echo "$dir" && break
done)

# If found, check operator versions
if [ -n "$FBC_DIR" ]; then
  yq ".operators[] | select(.name == \"${repo}\") | .versions[] | select(.name == \"${version}\")" "$FBC_DIR/operator-versions.yaml"
fi
```
If the FBC repo is not found locally, **skip this check** and report it as "N/A — no local clone found".

### Phase 7: Generate Report
Organize into a table per operator showing status of each layer.

## Return Value
A formatted table showing release readiness for each operator across all layers (upstream, images, community, midstream, FBC, downstream).

## Examples

1. **Check all operators**:
   ```
   /medik8s-release:status
   ```

2. **Check specific version**:
   ```
   /medik8s-release:status --version 0.13.0
   ```

3. **Check single operator**:
   ```
   /medik8s-release:status --operator SNR --version 0.13.1
   ```

## Arguments
- `--version <version>`: Target version to check (without `v` prefix)
- `--operator <op>`: Operator short name (SNR, NHC, FAR, MDR, NMO, SBR)

## See Also
- `/medik8s-release:upstream` — Run upstream release
- `/medik8s-release:verify-bundle` — Validate bundle correctness
