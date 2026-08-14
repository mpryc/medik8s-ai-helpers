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

The audit stages map directly to the JIRA child task structure (e.g., RHWA-1333). Each stage can be: DONE, IN PROGRESS, NOT STARTED, or BLOCKED.

**Phase 0. Upstream Release** (prerequisite — before JIRA creation)
- [ ] Code merged to main / release branch
- [ ] CI passing on release branch
- [ ] Release tag created (`v<version>`)
- [ ] GitHub Release exists (draft or published)
- [ ] Operator image on `quay.io/medik8s/<op>:v<version>`
- [ ] Bundle image on `quay.io/medik8s/<op>-bundle:v<version>`
- [ ] Catalog image on `quay.io/medik8s/<op>-catalog:v<version>`
- [ ] K8s community PR created/merged (skip MDR)
- [ ] OKD community PR created/merged

**JIRA 01. Konflux Update** — Update patch version for RPs
- [ ] ReleasePlans exist for stage and prod in `rhwa-tenant`
- [ ] ReleasePlanAdmissions exist
- [ ] Application with components visible at Konflux UI
- [ ] ArgoCD sync healthy

**JIRA 02. Update Downstream Repo** — Midstream preparation + FBC
- [ ] Midstream branch exists (Y-stream) / is correct (Z-stream)
- [ ] `scripts/update-version.sh` has target version
- [ ] `.tekton/*.yaml` has correct `semver-version`
- [ ] `bundle-hack/update_bundle.sh` has correct `PREVIOUS_VERSION`
- [ ] `bundle-hack/update_bundle.sh` has correct `SKIP_RANGE_LOWER`
- [ ] Version in FBC `operator-versions.yaml` (with `update-bundle: true`)
- [ ] Version in FBC `releases.yaml`
- [ ] FBC catalog generated and verified

**JIRA 03. Verify Images** — Builds, Enterprise Contract, stage release
- [ ] Operator image built in `quay.io/redhat-user-workloads/rhwa-tenant/`
- [ ] Bundle image built in `quay.io/redhat-user-workloads/rhwa-tenant/`
- [ ] Bundle image digest from `operator-versions.yaml` is pullable
- [ ] Enterprise Contract (ECP) passing
- [ ] Non-FBC stage release succeeded
- [ ] FBC stage release succeeded
- [ ] Stage advisory exists in advisory data repository
- [ ] FIPS compliance verified (`fbc-fips-check-oci-ta`)
- [ ] Disconnected environment support verified

**JIRA 04. Final Tests** — Pre-release validation
- [ ] Stage/prod RP/RPA alignment reviewed (minimal differences)
- [ ] JIRA release notes (type, text, status) filled in
- [ ] CVE data added to release manifests (if Go/dep bump resolves CVEs → RHSA)
- [ ] ProdSec vulnerability trackers added to `issues.fixed`
- [ ] GA Release YAMLs prepared (`create_non_fbc_release`, `create_fbc_release`)
- [ ] Operator installable from stage FBC
- [ ] Build labels verified (version, commit, date)
- [ ] E2e tests passed on stage cluster

**JIRA 05. QE Testing** — Quality Engineering validation
- [ ] CatalogSource with IIB from stage created
- [ ] Testing status sheet created/updated
- [ ] Regression tests passed
- [ ] Disconnected environment tested
- [ ] OCP upgrade tests passed (n-1 → n)
- [ ] All advisory bugs in verified state

**JIRA 06. Docs Checklist** — Documentation and advisory content
- [ ] Release notes created
- [ ] User guide updates for new features/bug fixes
- [ ] Advisory metadata reviewed (synopsis, topic, description, solution)
- [ ] Fixed issues attached and public in JIRA
- [ ] RHWA documentation version correct

**JIRA 07. GA Release** — Production release day
- [ ] Non-FBC Release CRs created and succeeded
- [ ] Advisory visible in prod advisory repo and customer portal
- [ ] Images accessible from `registry.redhat.io`
- [ ] FBC Release CRs created and succeeded (per OCP version)
- [ ] Operators visible in OperatorHub for target OCP versions
- [ ] Downstream repos tagged
- [ ] Post-GA FBC cleanup (remove `update-bundle: true`, switch to `registry.redhat.io`)

**Post-GA. Next OCP Catalog Preparation** (Y-stream only) — use `/medik8s-release:fbc-bootstrap`
- [ ] FBC catalogs prepared for next OCP versions (e.g., 4.23, 5.0)
- [ ] FBC snapshots built in Konflux
- [ ] `konflux-release-data` pruning exception MR submitted with per-arch digests
- [ ] Pruning exception MR approved (2 approvals required) and merged
- [ ] Pre-bootstrap FBC released before next OCP catalog bootstrap (see RHWA-616)

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

#### Phase 0: Upstream
```bash
# Tag exists?
gh api repos/medik8s/${repo}/git/refs/tags/v${version} 2>/dev/null && echo "DONE" || echo "NOT STARTED"

# GitHub Release?
gh release view v${version} --repo medik8s/${repo} --json isDraft,isPrerelease 2>/dev/null

# Release branch? (Y-stream only)
gh api repos/medik8s/${repo}/branches --jq ".[].name" | grep "^release-${major}.${minor}$"

# Upstream images
for suffix in "" "-bundle" "-catalog"; do
    curl -sf "https://quay.io/api/v1/repository/medik8s/${repo}${suffix}/tag/?specificTag=v${version}" \
        | jq -e '.tags | length > 0' > /dev/null 2>&1 && echo "DONE" || echo "NOT STARTED"
done

# Community PRs
gh pr list --repo k8s-operatorhub/community-operators \
    --head "add-${repo}-${version}-k8s" --state all --json state,url
gh pr list --repo redhat-openshift-ecosystem/community-operators-prod \
    --head "add-${repo}-${version}-okd-${ocp_version}" --state all --json state,url
```

#### JIRA 01: Konflux Update
```bash
# Check ReleasePlans exist (requires oc login to Konflux)
oc get releaseplan -n rhwa-tenant -l "operator=${prefix}" -o json 2>/dev/null \
    | jq -r '.items[].metadata.name' || echo "N/A — not logged in to Konflux"

# Check ReleasePlanAdmissions
oc get releaseplanadmission -n rhwa-tenant -l "operator=${prefix}" -o json 2>/dev/null \
    | jq -r '.items[].metadata.name' || echo "N/A"
```

#### JIRA 02: Midstream + FBC
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

#### JIRA 03: Verify Images
```bash
# Check if downstream images exist (use quay.io API — no auth needed, faster than skopeo)
for type in operator bundle; do
    component="${prefix}-${type}-${minor}"
    result=$(curl -sf "https://quay.io/api/v1/repository/redhat-user-workloads/rhwa-tenant%2F${repo}%2F${component}/tag/?limit=1&onlyActiveTags=true" \
        | jq -r '.tags[0].last_modified // "not found"')
    echo "${component}: ${result}"
done

# Verify bundle image digests from operator-versions.yaml are actually pullable
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

# Check stage advisory exists
curl -sf "https://gitlab.cee.redhat.com/api/v4/projects/rhtap-release%2Fadvisories/repository/tree?path=data/advisories/rhwa-tenant&ref=main" 2>/dev/null \
    | jq -r '.[].name' || echo "N/A — GitLab API not accessible (VPN required)"
```

#### JIRA 04-06: Manual Steps
Stages 04 (Final Tests), 05 (QE Testing), and 06 (Docs Checklist) require human verification. The audit reports their status based on JIRA state if accessible, otherwise marks them as "MANUAL CHECK REQUIRED".

#### JIRA 07: GA Release
```bash
# Check downstream tags (uses MIDSTREAM_DIR discovered earlier)
if [ -n "$MIDSTREAM_DIR" ]; then
  git -C "$MIDSTREAM_DIR" tag -l "v${version}"
fi

# Konflux releases (requires oc login)
oc get release -n rhwa-tenant -l "operator=${prefix}" --sort-by=.metadata.creationTimestamp -o json 2>/dev/null | jq '.items[-1].metadata.name'

# Check if images are on registry.redhat.io (post-GA)
skopeo inspect --no-tags "docker://registry.redhat.io/workload-availability/${registry_name}-rhel9-operator:v${version}" >/dev/null 2>&1 \
    && echo "DONE: image on registry.redhat.io" \
    || echo "NOT YET: not on registry.redhat.io"
```

### Phase 3: Generate Checklist Report
```
RELEASE AUDIT: RHWA 4.22-1 (Z-stream)
══════════════════════════════════════
JIRA: RHWA-1333

SNR v0.13.1 (replaces 0.13.0)
┌─────────────────────────────────────────────────────────┐
│ Phase 0. Upstream Release (prerequisite)                │
│    [x] Release tag v0.13.1                              │
│    [x] GitHub Release (published)                       │
│    [x] quay.io/medik8s/self-node-remediation:v0.13.1    │
│    [x] K8s PR #847 (merged)                             │
│    [ ] OKD PR — NOT STARTED                             │
│                                                         │
│ JIRA 01. Konflux Update                                 │
│    [x] ReleasePlans exist (snr-0-13-stage, snr-0-13-prod)│
│    [x] RPA approved                                     │
│                                                         │
│ JIRA 02. Update Downstream Repo                         │
│    [x] Branch snr-0-13 (semver: v0.13.1)                │
│    [x] PREVIOUS_VERSION=0.13.0                          │
│    [!] SKIP_RANGE_LOWER still set (should be commented  │
│        out for first Z-stream)                          │
│    [x] In operator-versions.yaml                        │
│    [x] In releases.yaml                                 │
│                                                         │
│ JIRA 03. Verify Images                                  │
│    [x] Operator image built (2026-07-20)                │
│    [x] Bundle image built (2026-07-20)                  │
│    [x] Bundle digest pullable                           │
│    [ ] Stage release — NOT VERIFIED                     │
│                                                         │
│ JIRA 04. Final Tests                                    │
│    [ ] MANUAL CHECK — verify in JIRA                    │
│                                                         │
│ JIRA 05. QE Testing                                     │
│    [ ] MANUAL CHECK — verify in JIRA                    │
│                                                         │
│ JIRA 06. Docs Checklist                                 │
│    [ ] MANUAL CHECK — verify in JIRA                    │
│                                                         │
│ JIRA 07. GA Release                                     │
│    [ ] Konflux Release CR — NOT STARTED                 │
│    [ ] Downstream tag — NOT STARTED                     │
│    [ ] registry.redhat.io — NOT YET                     │
└─────────────────────────────────────────────────────────┘

NEXT STEPS:
  1. Create OKD community PR for SNR 0.13.1
  2. Fix SKIP_RANGE_LOWER in bundle-hack (comment out for Z-stream)
  3. Complete stage release test (JIRA 03)
  4. QE testing with IIB from stage (JIRA 05)
  5. Create GA Release YAMLs (JIRA 04)
  6. Create Konflux Release CRs on release day (JIRA 07)

NHC v0.12.1 (replaces 0.12.0)
  ...
```

### Phase 4: Provide Actionable Next Steps
Based on the first incomplete stage for each operator, suggest the exact command or action needed. Map each action to its JIRA child task number.

## Return Value
Formatted checklist with progress indicators and actionable next steps, organized by JIRA child task.

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
- `/medik8s-release:fbc-bootstrap` — Prepare FBC for next OCP versions (post-GA)
