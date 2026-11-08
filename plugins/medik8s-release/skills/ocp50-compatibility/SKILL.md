---
name: ocp50-compatibility
description: Run and document an OCP 5.0 operator compatibility test cycle for RHWA operators. Covers fresh-install e2e, FBC upgrade, and filling the Jira epic closing comment. Trigger with /ocp50-compat or when user asks to test OCP 5.0 compatibility for an RHWA operator.
metadata:
  triggers:
    - ocp50-compat
    - ocp 5.0 compat
    - compatibility report
    - rhwa-system-tests
    - FBC upgrade test
---

# OCP 5.0 Operator Compatibility — Test & Report Skill

## Purpose

Runs the full RHWA OCP 5.0 compatibility test cycle for a given operator and fills the
Jira epic description + closing comment. Covers:

1. **All rhwa-system-tests** — run everything via `ECO_TEST_FEATURES`; some fail with known infra reasons
2. **Epic description** — populate task checkboxes and narrative in Jira via ADF REST API
3. **Closing comment** — Jira HTML draft for the operator's OCP 5.0 epic

---

## Execution rules

**Collect all required inputs upfront in a single message before doing
anything else** (before reading files, touching the cluster, or running any
command). Once inputs are confirmed, run all steps automatically with zero
questions.

Make every decision unilaterally:
- If `RELEASE_URL` is missing, skip the closing-comment link (it's optional)
- If a test fails for a known infra reason (RHWA-known-failures list), mark
  it as "expected failure" and continue — no question needed
- If the Jira epic description already has content, append/merge rather than
  overwrite
- **One mandatory stop**: before posting the closing comment to Jira — show
  the HTML draft and wait for explicit approval

**Surface warnings** during processing and in the final output:
- Print a one-line warning for unexpected test failures, missing prereqs, or
  Jira API errors
- Collect all warnings into a **Warnings** block at the end of the report

## Required Inputs — Collect Before Proceeding

**Before doing anything else** — including reading files, touching the cluster, or running any command — confirm all required inputs are present. If any are missing from the user's command, ask for them all in a single message. Do not proceed until all required inputs are provided.

| Input | Required | Example | Notes |
|---|---|---|---|
| `JIRA_EPIC` | Yes | `RHWA-919` | Full Jira key |
| `IIB_NUMBER` | Yes | `1246085` | Index image number (integer only) |
| `SNAPSHOT_NAME` | Yes | `rhwa-fbc-500-20261105-113444-000` | From Konflux UI URL or snapshot tab |
| `KUBECONFIG` | Yes | `~/Downloads/kubeconfig-test-cluster3` | Path to test cluster kubeconfig |
| `RELEASE_URL` | No | Konflux release artifacts URL | For closing comment link; skip if not provided |

**Input validation before proceeding:**
- `JIRA_EPIC` matches `RHWA-\d+`
- `IIB_NUMBER` is a plain integer (no prefix)
- `SNAPSHOT_NAME` matches `rhwa-fbc-500-YYYYMMDD-HHMMSS-NNN`
- `KUBECONFIG` path exists on disk (`ls <path>`)

If the user provides a full Konflux URL (snapshot or release), extract the name from the path segment — do not ask again.

**Derive before running tests:**

```bash
# 1. Read the Jira ticket to confirm operator name and component
curl -sf -u "oraz@redhat.com:$JIRA_API_TOKEN" \
  "https://redhat.atlassian.net/rest/api/3/issue/$JIRA_EPIC?fields=summary,components" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); f=d['fields']; print(f['summary'], [c['name'] for c in f['components']])"
# Example output: "MDR v5.8.0 OCP 5.0 Compatibility ['Machine Deletion Remediation']"
# → operator = mdr, ECO_TEST_FEATURES = mdr-operator, FBC prefix = MDR_FBC_

# 2. Derive FBC env var prefix: uppercase operator abbreviation + _FBC_
# FAR → FAR_FBC_*, MDR → MDR_FBC_*, SNR → SNR_FBC_*, NHC → NHC_FBC_*, NMO → NMO_FBC_*, SBR → SBR_FBC_*

# 3. Get catalog image digest from snapshot (tag = full 40-char git SHA, NOT short hash)
COMMIT=$(curl -sf "https://quay.io/v2/redhat-user-workloads/rhwa-tenant/rhwa-fbc/rhwa-fbc-500/tags/list" \
  | python3 -c "import sys,json; tags=json.load(sys.stdin)['tags']; print([t for t in tags if len(t)==40 and 'linux' not in t and '.git' not in t and '.life' not in t][-1])")
FBC_CATALOG_DIGEST=$(skopeo inspect --raw "docker://quay.io/redhat-user-workloads/rhwa-tenant/rhwa-fbc/rhwa-fbc-500:$COMMIT" \
  | python3 -c "import sys,hashlib; raw=sys.stdin.buffer.read(); print('sha256:'+hashlib.sha256(raw).hexdigest())")
export ${OP}_FBC_CATALOG_IMAGE="quay.io/redhat-user-workloads/rhwa-tenant/rhwa-fbc/rhwa-fbc-500@$FBC_CATALOG_DIGEST"

# 4. Get operator image digest from IIB packagemanifest (filter to correct catalog)
oc get packagemanifest -n openshift-marketplace -l catalog=rhwa-iib-$IIB_NUMBER -o json \
  | python3 -c "
import sys,json; d=json.load(sys.stdin)
for item in d['items']:
    if '<package-name>' in item['metadata']['name']:   # e.g. machine-deletion-remediation
        ch=item['status']['channels'][0]
        print('CSV:', ch['currentCSV'])
        for img in ch.get('currentCSVDesc',{}).get('relatedImages',[]):
            print(img if isinstance(img,str) else img.get('image',''))
"
# Pick the registry.redhat.io/workload-availability/<op>-rhel9-operator@sha256:... line
export ${OP}_FBC_CANDIDATE_IMAGE="registry.redhat.io/workload-availability/<op>-rhel9-operator@sha256:<digest>"
export ${OP}_FBC_CANDIDATE_VERSION="5.8.0"   # NO "v" prefix — csv.Spec.Version.String() returns bare semver
```

---

## Prerequisites

| Resource | Where to find |
|---|---|
| OCP 5.0 kubeconfig | `~/.kube/kubeconfig_bot` has latest IIBs; `kubeconfig-test-cluster3` may have stale IIBs — always add the new IIB catalog source if missing |
| `rhwa-system-tests` clone | `/tmp/claude-*/repos/rhwa-system-tests` or `~/Documents/dragonfly/medik8s/rhwa-system-tests`; check both before cloning |
| ADF scripts (FAR only) | `~/Documents/rhwa/ocp50/far/far_adf.py` (epic description) + `far_comment_adf.py` (closing comment); no equivalent scripts exist yet for MDR/SNR/NHC/NMO/SBR |
| Go ≥ 1.21 | `export PATH=$PATH:/usr/local/go/bin` |
| `JIRA_API_TOKEN` | env var, used for description + comment POST |

---

## Step 0 — Add IIB catalog source if missing

```bash
export KUBECONFIG=<kubeconfig-path>

# Check if IIB already on cluster
oc get catalogsource rhwa-iib-$IIB_NUMBER -n openshift-marketplace 2>/dev/null || \
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: rhwa-iib-$IIB_NUMBER
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: brew.registry.redhat.io/rh-osbs/iib:$IIB_NUMBER
  displayName: RHWA IIB $IIB_NUMBER
  updateStrategy:
    registryPoll:
      interval: 10m
EOF

# Wait for READY
until [ "$(oc get catalogsource rhwa-iib-$IIB_NUMBER -n openshift-marketplace \
  -o jsonpath='{.status.connectionState.lastObservedState}')" = "READY" ]; do sleep 5; done
```

---

## Step 1 — Two-phase test run

The FBC upgrade test must run **first on a clean cluster** (no operator pre-installed).
Running it after a manual install fails with `namespace already exists; this run does not own it`.

### Phase 1 — FBC upgrade test (clean cluster)

```bash
export KUBECONFIG=<kubeconfig-path>
export PATH=$PATH:/usr/local/go/bin
cd /path/to/rhwa-system-tests

export WORKLOAD_IMAGE=registry.access.redhat.com/ubi9/ubi-minimal:latest
export ECO_TEST_FEATURES="<op>-operator"      # e.g. mdr-operator, far-operator
export ECO_VERBOSE_LEVEL=100
export ECO_REPORTS_DUMP_DIR=$(mktemp -d -p $TMPDIR)
export ECO_IS_DOWNSTREAM=true
export ECO_TEST_LABELS="tier:upgrade-operator"

export <OP>_FBC_CATALOG_IMAGE="quay.io/redhat-user-workloads/rhwa-tenant/rhwa-fbc/rhwa-fbc-500@sha256:<digest>"
export <OP>_FBC_CANDIDATE_VERSION="5.8.0"     # NO "v" prefix
export <OP>_FBC_CANDIDATE_IMAGE="registry.redhat.io/workload-availability/<op>-rhel9-operator@sha256:<digest>"

make run-tests 2>&1 | tee $TMPDIR/<op>-fbc.log
```

After Phase 1 completes, the test framework deletes `openshift-workload-availability` (SkipCleanup:false).

### Phase 2 — All remaining tests (after FBC upgrade cleans up)

```bash
# Re-create namespace + OperatorGroup + operator subscription
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-workload-availability
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-workload-availability-og
  namespace: openshift-workload-availability
spec:
  targetNamespaces: []
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: <op>-sub
  namespace: openshift-workload-availability
spec:
  channel: stable
  name: <package-name>          # e.g. machine-deletion-remediation, fence-agents-remediation
  source: rhwa-iib-$IIB_NUMBER
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF

# For MDR and SNR: also install NHC (required for nhc-interop / nhc-triggered remediation tests)
# cat <<EOF | oc apply -f -
# apiVersion: operators.coreos.com/v1alpha1
# kind: Subscription
# metadata:
#   name: nhc-sub
#   namespace: openshift-workload-availability
# spec:
#   channel: stable
#   name: node-healthcheck-operator
#   source: rhwa-iib-$IIB_NUMBER
#   sourceNamespace: openshift-marketplace
#   installPlanApproval: Automatic
# EOF

# Wait for CSV Succeeded
until oc get csv -n openshift-workload-availability \
  -o jsonpath='{range .items[*]}{.metadata.name}={.status.phase}{"\n"}{end}' \
  | grep -q "Succeeded"; do sleep 15; done

# Run all remaining specs (exclude FBC upgrade)
unset ECO_TEST_LABELS
make run-tests 2>&1 | tee $TMPDIR/<op>-main.log
```

---

## Step 2 — Per-operator expected test outcomes

### FAR (`far-operator`)

| Test file | Tier/Labels | Infra needed | Expected |
|---|---|---|---|
| `far.go` | `TierSmoke, NonDestructive` | None | PASS |
| `far_controller_lifecycle.go` | `TierAcceptance, Controller` | None | PASS |
| `far_negative_validation.go` | — | None (webhook) | PASS |
| `upgrade_operator_fbc.go` | `tier:upgrade-operator` | FBC vars | PASS |
| `far_destructive.go` | `TierAcceptance, Destructive` | `aws-cloud-fencing-credentials-secret` | FAIL expected (`test_id:61229`, CI covers it) |
| `far_observability.go` | `PlatformAWS` | AWS CCO creds | FAIL expected (`test_id:70873`, CI covers it) |
| `far_nhc_interop.go` | `TierAcceptance, Interop` | NHC installed | FAIL if NHC absent |
| `upgrade.go` | `TierUpgrade, PlatformAWS, Weekly` | 4.22 cluster | FAIL expected (CI-only) |

### MDR (`mdr-operator`)

| Test file | Tier/Labels | Infra needed | Expected |
|---|---|---|---|
| `mdr.go` | `TierSmoke, NonDestructive` | None | PASS |
| `mdr_controller_lifecycle.go` / `upgrade_lifecycle.go` | `TierAcceptance` | None | PASS |
| `mdr_negative.go` | — | None (webhook) | PASS |
| `mdr_conditions.go` | `TierAcceptance` | None | PASS |
| `must_gather.go` | — | None | PASS |
| `upgrade_operator_fbc.go` | `tier:upgrade-operator` | FBC vars | PASS |
| `mdr_remediation.go` | `Destructive, Weekly, test_id:66138` | NHC controller running + Machine API | FAIL if NHC absent; PASS if NHC installed on AWS cluster |

> **MDR note:** NHC CRD alone is not enough — NHC controller must be running (CSV Succeeded). Without it, `mdr_remediation.go` times out waiting for NHC to trigger the MDR CR.

### SNR / NHC / NMO / SBR

Test expectations not yet documented. Add rows as each operator is tested.

---

## Step 3 — Find FBC params and verify IDMS

```bash
# Verify IDMS covers operator image (note: --no-creds not supported on this skopeo version)
skopeo inspect "docker://quay.io/redhat-user-workloads/rhwa-tenant/<op>/<op>-operator-5-8@sha256:<digest>"
# Must NOT return "manifest unknown"
```

**Critical gotchas for the FBC upgrade test:**

| Gotcha | Fix |
|---|---|
| `<OP>_FBC_CANDIDATE_VERSION=v5.8.0` with `v` prefix | Drop `v` — `csv.Spec.Version.String()` returns bare `5.8.0`; mismatch → 15-min polling timeout |
| `namespace already exists; this run does not own it` | Phase 1 must run on clean cluster. Delete namespace first if needed: `oc delete namespace openshift-workload-availability --wait=true` |
| `TooManyOperatorGroups` — CSV fails Installing | Namespace already has an OperatorGroup. Check with `oc get og -n openshift-workload-availability` before creating another |
| OperatorGroup missing after FBC upgrade test cleanup | FBC test deletes namespace (including OG). Re-create with `spec: {}` (AllNamespaces) before subscribing in Phase 2 |
| CatalogSource stuck `TRANSIENT_FAILURE` | Cluster lacks pull creds for `brew.registry.redhat.io`; fall back to quay.io snapshot image |
| Quay.io snapshot tag = full 40-char SHA | Tag is the full git commit hash, not the short one (e.g. `0dc0074b7a1d7a1e5768c160aae65435c51bcdf5`, not `0dc0074`) |
| `skopeo inspect --no-creds` fails | Drop `--no-creds` — not supported on this version |
| CSV stuck in Installing >5 min | Cold image cache via IDMS — wait or pre-pull |

**Cluster state check after upgrade:**
```bash
oc get csv -n openshift-workload-availability \
  -o jsonpath='{range .items[*]}{.metadata.name}={.status.phase}{"\n"}{end}'
```

---

## Step 4 — Populate the Jira epic description

The epic description (`fields.description`) is an ADF document.

**FAR only**: Use `~/Documents/rhwa/ocp50/far/far_adf.py` as the template.
**All other operators**: ADF scripts do not exist yet — construct ADF manually or adapt `far_adf.py`.

```bash
python3 far_adf.py > $TMPDIR/<op>-description.json

curl -sf -u "oraz@redhat.com:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" -X PUT \
  "https://redhat.atlassian.net/rest/api/3/issue/$JIRA_EPIC" \
  -d "$(cat $TMPDIR/<op>-description.json)"
```

**Task checkboxes to tick DONE (generic):**
- Operator installed on OCP cluster → DONE (prior GA version from redhat-operators)
- Operator survives upgrade without data loss → DONE (UID preserved after FBC upgrade)
- Workloads functioning after upgrade → DONE
- No unexpected errors → DONE
- Platform upgrade (4.22 → 5.0) → DONE — covered by `tier:upgrade` weekly CI
- Day 2 ops → DONE: operator upgrade (CI), config changes (webhook e2e), scaling (HA tests)
- Coordination + Communication Template → remove from epic; Communication Template goes in RHWA-785 description

> See [[feedback_jira_use_description]] — always use ADF via REST API v3, not jira-cli.
> See [[feedback_jira_heading_hierarchy]] — H1 major sections, H2 supporting, H3 sub-sections.

---

## Step 5 — Fill the Jira closing comment

Build a draft HTML file and open in Firefox for copy review before posting.

**Test Environment section:**
- Cluster: `oc version` → OCP version, K8s version, RHCOS version
- Operator image: from deployment spec
- Downstream commit: GitLab dragonfly repo, branch `<op>-5-8`, last commit
- Upstream commit: GitHub medik8s repo, branch `medik8s-5.8`
- Catalog source: IIB or Konflux snapshot
- Konflux release: link from `RELEASE_URL`
- CSV installed: `<op>.v5.8.0 (Succeeded), replaces v<prev>`

**Test Results section:**
- N passed, M failures — panel-success/warn per failure
- For each failure: test_id label, root cause, whether it blocks compat
- FBC upgrade goes in a panel-success block (not in failures list)

**Decision Tree:**
- Q1 (kernel/OS deps): `oc get deployment -o jsonpath='{.spec.template.spec}'` — no hostPID, hostNetwork, privileged
- Q3 (deprecated K8s APIs): `oc api-resources --api-group=<group>` on OCP 5.0 RC; confirm CRDs present

**Remaining Open Items:**
- RHWA-1637 (GA release), per-operator OPGM ticket (FBC prod release, linked to RHWA-785), PLMCORE-17275 (lifecycle), TELCODOCS-2951 (docs)

```python
import os, requests
headers = {"Authorization": f"Bearer {os.environ['JIRA_API_TOKEN']}",
           "Content-Type": "application/json"}
payload = {"body": adf_doc}
r = requests.post(
    f"https://redhat.atlassian.net/rest/api/3/issue/{jira_epic}/comment",
    headers=headers, json=payload)
```

> Per CLAUDE.md: **always show draft and get explicit approval before posting.**

---

## Step 6 — Jira epic blocking links

Each OCP 5.0 epic (RHWA-1708/1709/1710/1711/1147/919) should be "is blocked by":
- TELCODOCS-2951, PLMCORE-17275, per-operator OPGM ticket, RHWA-1637

```python
payload = {"type": {"name": "Blocks"},
           "inwardIssue": {"key": jira_epic},
           "outwardIssue": {"key": blocker}}
requests.post(".../rest/api/3/issueLink", headers=headers, json=payload)
```

---

## Checklist

- [ ] Jira ticket read — operator name and component confirmed
- [ ] IIB catalog source on cluster and READY
- [ ] FBC params derived: `<OP>_FBC_CATALOG_IMAGE` (digest-pinned), `<OP>_FBC_CANDIDATE_VERSION` (no `v`), `<OP>_FBC_CANDIDATE_IMAGE` (digest-pinned)
- [ ] IDMS on cluster covers operator image digests
- [ ] Phase 1 (FBC upgrade) run on clean cluster — PASS
- [ ] Phase 2 (all remaining tests) run with operator + NHC pre-installed — results documented
- [ ] All failures explained (expected vs. blocking)
- [ ] Epic description updated via ADF REST API (not comment, not jira-cli)
- [ ] Communication Template moved to RHWA-785 description
- [ ] Closing comment draft reviewed in Firefox before posting
- [ ] Jira comment posted with explicit user approval
- [ ] Blocking links created (6 epics × 4 blockers)
