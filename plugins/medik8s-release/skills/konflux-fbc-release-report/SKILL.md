# konflux-fbc-release-report

Generate a release validation report for an RHWA Konflux release.

Namespace: always `rhwa-tenant` on `stone-prod-p02`.

## Invocation

```
/konflux-fbc-release-report
```

Args (space-separated, all optional):
- `<ver-suffix>` — app version suffix, e.g. `5-8` (derives apps nhc-5-8 … rhwa-fbc-500)
- `--jira <TICKET>` — Jira issue to post the comment to, e.g. `RHWA-1640`
- `--ec-logs <dir>` — directory containing `*-verify.log` files downloaded from EC PipelineRun logs

## Execution rules

**Run all steps automatically with zero questions.** Make every decision unilaterally:
- If `ver-suffix` is missing, infer it from the most recent snapshot timestamp in `rhwa-tenant`
- If a stage release has no advisory yet, mark that cell "pending" and continue
- If skopeo fails for an image, mark that row "inspect failed" and continue
- If `--ec-logs` not given, fill Conforma Result as "see EC PipelineRun log"
- If FBC advisory is IIB (not Errata), use the IIB link — no question needed

**One mandatory stop**: before posting to Jira (Step 6). Show the HTML preview first. Only post after the user says so explicitly.

**Surface warnings** during processing and in the HTML:
- Print a one-line warning to the user whenever something is unexpected (skopeo failed, no release found, advisory missing, EC log absent for an operator, GA MR still open, component warning count above 2, CVE-type warnings in EC results)
- Collect all warnings into a **Warnings** section at the bottom of the HTML, styled visibly (orange banner), so they are not missed before posting

## Workflow

### Step 0 — Parse args

Extract `ver-suffix`, `--jira`, `--ec-logs` from args. If `ver-suffix` missing, ask the user.

Operator apps (derived from `ver-suffix`):
```
nhc-${ver-suffix}   snr-${ver-suffix}   far-${ver-suffix}
mdr-${ver-suffix}   nmo-${ver-suffix}   sbr-${ver-suffix}
rhwa-fbc-500
```

Output HTML path: `$TMPDIR/release-report-${ver-suffix}.html`

---

### Step 1 — Discover snapshots

For each operator app, find the latest snapshot:

```bash
oc get snapshot -n rhwa-tenant -l appstudio.openshift.io/application=<app> \
  --sort-by=.metadata.creationTimestamp -o json | jq '.items[-1]'
```

From each snapshot extract:
- **Snapshot name**: `.metadata.name`
- **Build PLR**: `.metadata.labels["appstudio.openshift.io/build-pipelinerun"]`
- **EC PLR**: parse `.metadata.annotations["test.appstudio.openshift.io/status"]` JSON → find the entry with `scenarioName` matching `enterprise-contract` or `conforma`; its `testPipelineRunName` is the EC PLR name
- **Component images**: `.spec.components[*].{name, containerImage}`

Build PLR link: `https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/rhwa-tenant/pipelinerun/<plr>`

EC PLR link: `https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/rhwa-tenant/applications/<app>/pipelineruns/<plr>/logs?task=verify`

---

### Step 2 — Discover stage releases and advisories

Find stage release for each snapshot:

```bash
oc get release -n rhwa-tenant --field-selector=spec.snapshot=<snapshot-name> -o json
# If field-selector unsupported:
oc get release -n rhwa-tenant -o json | jq '.items[] | select(.spec.snapshot == "<snapshot-name>")'
```

From the release object:
- **Stage release name**: `.metadata.name`
- **Advisory URL**: `.status.artifacts.advisory.url`
  - If URL contains `iib.engineering.redhat.com` → IIB build (FBC path); extract build number
  - If URL contains `errata` → extract advisory number for GitLab link
  - Advisory YAML path: `https://gitlab.cee.redhat.com/rhtap-release/advisories/-/blob/main/data/advisories/rhwa-tenant/<year>/<advisory-number>/advisory.yaml`
  - Verify the path exists via GitLab API: `GET /api/v4/projects/rhtap-release%2Fadvisories/repository/tree?path=data/advisories/rhwa-tenant/<year>/<num>`

Stage release link: `https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/rhwa-tenant/applications/<app>/releases/<release-name>`

---

### Step 3 — skopeo inspect (parallel)

For each component image digest from Step 1, run in parallel:

```bash
skopeo inspect docker://<image-ref> | jq '{
  name: .Labels["name"],
  version: .Labels["version"],
  release: .Labels["release"],
  build_date: .Labels["build-date"],
  upstream: (.Labels["upstream-version"] // "N/A"),
  vcs_ref: .Labels["vcs-ref"],
  cpe: .Labels["cpe"],
  vendor: .Labels["vendor"],
  scope: .Labels["distribution-scope"],
  url: .Labels["url"]
}'
```

**Upstream repo mapping** (derive from component name):
| Component prefix | upstream-version repo |
|---|---|
| `nhc-operator`, `nhc-bundle` | `medik8s/node-healthcheck-operator` |
| `nhc-console` | `medik8s/node-remediation-console` |
| `nhc-must-gather` | `medik8s/must-gather` |
| `snr-*` | `medik8s/self-node-remediation` |
| `far-*` | `medik8s/fence-agents-remediation` |
| `mdr-*` | `medik8s/machine-deletion-remediation` |
| `nmo-*` | `medik8s/node-maintenance-operator` |
| `sbr-*` | `medik8s/storage-based-remediation` |
| `rhwa-fbc-*` | N/A |

**Downstream GitLab repo mapping** (derive from component name for vcs-ref links):
| Component prefix | dragonfly repo |
|---|---|
| `nhc-*` | `node-healthcheck-operator` |
| `snr-*` | `self-node-remediation` |
| `far-*` | `fence-agents-remediation` |
| `mdr-*` | `machine-deletion-remediation` |
| `nmo-*` | `node-maintenance-operator` |
| `sbr-*` | `storage-based-remediation` |
| `rhwa-fbc-*` | `rhwa-fbc` |

vcs-ref link: `https://gitlab.cee.redhat.com/dragonfly/<ds-repo>/-/commit/<full-sha>`
upstream-version link: `https://github.com/medik8s/<up-repo>/commit/<sha7-from-label>`

---

### Step 4 — Parse EC logs (if --ec-logs provided)

For each `*-verify.log` file in the EC logs directory:

```python
import re

text = open(logfile).read()

# 1. Find manifest-level entries (no arch suffix in Name):
pattern = r'- Name: ([a-z][a-z0-9-]+-\d+-\d+|rhwa-fbc-\d+)\n  ImageRef:[^\n]*\n  Violations:\s+(\d+),\s+Warnings:\s+(\d+)'
manifest_matches = re.findall(pattern, text)
# → {component: {violations, warnings}}

# 2. For non-standard counts (>2 warnings), get warning types:
#    Find manifest containerImage from the components JSON block at top,
#    then search for [Warning] lines with that ImageRef
comp_img_pattern = r'"name":\s*"<comp>".*?"containerImage":\s*"([^"]+)"'
warn_detail_pattern = r'\[Warning\]\s+(\S+)\s+ImageRef:\s+<img>\s+Reason:\s+([^\n]+)'
```

If `--ec-logs` not provided: show "see EC PipelineRun log" for Conforma Result column.

**Standard warnings** (2 per manifest-level component): `test.no_failed_informative_tests`, `test.no_test_warnings`

FBC-500: manifest-level typically shows 0 warnings; per-arch images carry CVE warnings — report per-arch counts separately.

---

### Step 5 — Render HTML

Write to `$TMPDIR/release-report-${ver-suffix}.html` with:

**Section 1** — Push PipelineRuns and Conforma Results  
Table: Operator | Snapshot | Build PipelineRun | EC PipelineRun | Conforma Result (bullet list per component)

**Section 2** — Stage Advisories  
Table: Operator | Snapshot | Stage Release | Stage Advisory (linked to GitLab advisory YAML or IIB log)

**Section 3** — Image Label Verification  
Per-operator h2 header + table: Component | name | version | release | build-date | upstream-version | vcs-ref | vendor | scope | url

**Section 4** — Additional Validation  
Table: GA Release YAML MR (link dragonfly/tools MR) | Install test | e2e tests  
(User fills in status for install/e2e; GA MR link from dragonfly/tools latest open MR for this release)

**Warnings section** (append only if warnings list is non-empty) — orange banner, one bullet per warning:
```
⚠ Warnings
• snr-operator-5-8: 3 EC warnings (above baseline of 2) — test.no_failed_informative_tests ×2
• nmo-bundle-5-8: 8 EC warnings including 6 unpatched CVEs
• EC logs not provided for FAR, MDR — Conforma Result shows placeholder
• dragonfly/tools !426 (GA Release YAMLs) is still open
• sbr-5-8-20261108-022913-000: snapshot found but no stage release — used 023055 instead
```

Then open in Firefox:
```bash
DISPLAY=:0 firefox "file://$TMPDIR/release-report-${ver-suffix}.html" &>/dev/null & disown
```

---

### Step 6 — Post to Jira (if --jira provided)

Convert report to ADF and POST to `https://redhat.atlassian.net/rest/api/3/issue/<TICKET>/comment`:

```bash
curl -s -X POST \
  -u oraz@redhat.com:$JIRA_API_TOKEN \
  -H "Content-Type: application/json" \
  "https://redhat.atlassian.net/rest/api/3/issue/<TICKET>/comment" \
  -d "<adf-json>"
```

**ADF structure**: doc → heading(1) per section → table per section → bulletList for Conforma cells.

Jira host must be `redhat.atlassian.net` directly (issues.redhat.com redirects with 301, losing POST body).

---

## Known manual steps

| Step | Reason | Workaround |
|---|---|---|
| EC log download | Tekton Results API is SSO-gated; PipelineRuns GC'd after 3 runs | User downloads logs from Konflux UI, passes `--ec-logs <dir>` |
| GA Release YAML MR status | Requires dragonfly/tools MR to be open/merged | Check `gl mr list --repo dragonfly/tools` or GitLab API |

## Key gotchas

- FBC-500 advisory goes through IIB (index image builder), not Errata — look for IIB build number in release status
- Two SBR snapshots may exist for the same date — use the one with a stage release (advisory) attached
- `issues.redhat.com` → `redhat.atlassian.net` 301 redirect drops POST body; always POST to Atlassian URL directly
- GPG sandbox: `git commit -S` requires `dangerouslyDisableSandbox: true`
- skopeo inspect in parallel with `&` + `wait` for speed; 15+ images take ~60s sequentially
- Advisory directory numbering in GitLab is numeric only (e.g. `176312`), not `RHBA-2026:176312`
- vcs-ref in operator images is the **downstream** GitLab commit; upstream-version label holds the **upstream** GitHub commit
- nhc-console and nhc-must-gather upstream repos differ from nhc-operator (see mapping table above)
