# FBC Snapshot Table

Generate the operator snapshot table for a dragonfly/tools release MR from a
single Konflux FBC release-to-stage URL.

## Invocation

```
/medik8s-release:fbc-snapshot-table <release-url>
```

Where `<release-url>` is the Konflux UI link to an FBC stage release, e.g.:
```
https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/rhwa-tenant/applications/rhwa-fbc-500/releases/rhwa-fbc-500-20261108-070637-000-27c4e72-g25wq
```

Output: a markdown table (snapshot / downstream commit / downstream date /
upstream commit / upstream date) ready to paste into the MR description,
plus the FBC catalog row and the new IIB number.

---

## Execution rules

**Run all steps automatically with zero questions.** Make every decision
unilaterally:
- If `ver-suffix` is ambiguous, infer it from the app name in the URL
  (`rhwa-fbc-500` → `5-8`)
- If a snapshot lookup returns multiple matches, pick the clean name (no
  random suffix)
- If skopeo fails for an operator image, mark that row "inspect failed"
  and continue
- If the IIB number is not yet set in the release status, mark it "pending"

**Surface warnings** during processing and at the end of output:
- Print a one-line warning to the user whenever something is unexpected
  (skopeo failed, no release found, IIB pending, multiple snapshot matches,
  missing `upstream-version` label)
- Collect all warnings into a **Warnings** block appended after Block C,
  one bullet per warning, so they are visible before pasting into the MR

---

## Step 0 — Parse URL

Extract from the URL:
- `namespace` — path segment after `/ns/` (e.g. `rhwa-tenant`)
- `application` — path segment after `/applications/` (e.g. `rhwa-fbc-500`)
- `release_name` — path segment after `/releases/` (e.g. `rhwa-fbc-500-20261108-070637-000-27c4e72-g25wq`)

Derive the **FBC snapshot name**: strip the trailing `-<shortsha>-<suffix>` from
the release name. Pattern: everything up to and including the third `-000`
segment is the snapshot name.

Example: `rhwa-fbc-500-20261108-070637-000-27c4e72-g25wq` →
snapshot `rhwa-fbc-500-20261108-070637-000`.

Derive the **ver-suffix**: drop the `rhwa-fbc-` prefix and `-YYYYMMDD-…` suffix.
For `rhwa-fbc-500` the ver-suffix convention is `5-8` (derived from release
branch context, not the app name). If ambiguous, ask the user.

---

## Step 1 — Get FBC snapshot and rhwa-fbc commit

```bash
kubectl get snapshot <fbc-snapshot-name> -n rhwa-tenant -o json
```

From the snapshot:
- **FBC image digest**: `.spec.components[0].containerImage`
- **rhwa-fbc repo commit** (full SHA):
  `.metadata.annotations["build.appstudio.redhat.com/commit_sha"]`
- **rhwa-fbc short commit**: first 7 chars

---

## Step 2 — Get bundle digests from rhwa-fbc repo

Use the GitLab API (requires `$GITLAB_PRIVATE_TOKEN`):

```bash
curl -s --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/dragonfly%2Frhwa-fbc/repository/files/update_bundle.sh/raw?ref=<rhwa-fbc-commit>" \
  | grep '^export '
```

If the rhwa-fbc repo is already cloned locally, prefer:
```bash
git show <rhwa-fbc-commit>:update_bundle.sh
```

Parse the `export VAR=quay.io/…@sha256:<digest>` lines to extract:
- component name from the image URL path segment before `@sha256`
- full digest (`sha256:<hex>`)

The standard components and their operator app mapping:

| update_bundle.sh var | bundle component | operator app |
|---|---|---|
| `node_healthcheck_operator_5_8` | `nhc-bundle-5-8` | `nhc-5-8` |
| `self_node_remediation_5_8` | `snr-bundle-5-8` | `snr-5-8` |
| `fence_agents_remediation_5_8` | `far-bundle-5-8` | `far-5-8` |
| `machine_deletion_remediation_5_8` | `mdr-bundle-5-8` | `mdr-5-8` |
| `node_maintenance_operator_5_8` | `nmo-bundle-5-8` | `nmo-5-8` |
| `storage_based_remediation_5_8` | `sbr-bundle-5-8` | `sbr-5-8` |

---

## Step 3 — Find operator snapshots matching bundle digests

For each bundle digest (use first 20 chars of hex for the jq filter):

```bash
kubectl get snapshot -n rhwa-tenant -o json | jq -r '
  .items[] |
  .metadata.name as $name |
  .spec.components[]? |
  select(.containerImage | test("<digest-prefix>")) |
  $name'
```

Pick the **single** snapshot that matches. If multiple match, prefer the one
without a trailing `-XX` randomized suffix (the canonical clean name).

---

## Step 4 — Get downstream commit from snapshot annotations

For each operator snapshot:

```bash
kubectl get snapshot <snapshot-name> -n rhwa-tenant -o json | jq '{
  commit: .metadata.annotations["build.appstudio.redhat.com/commit_sha"],
  sha_url: .metadata.annotations["pac.test.appstudio.openshift.io/sha-url"],
  authored_date: .metadata.annotations["pac.test.appstudio.openshift.io/sha"]
}'
```

The downstream commit short ID is the first 8 chars of `commit`. The
downstream date is from the GitLab API for that commit:

```bash
curl -s --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/dragonfly%2F<ds-repo>/repository/commits/<commit-sha>" \
  | jq '.authored_date'
```

Downstream repo mapping (derive from operator app name):

| app prefix | dragonfly repo |
|---|---|
| `nhc-` | `node-healthcheck-operator` |
| `snr-` | `self-node-remediation` |
| `far-` | `fence-agents-remediation` |
| `mdr-` | `machine-deletion-remediation` |
| `nmo-` | `node-maintenance-operator` |
| `sbr-` | `storage-based-remediation` |

---

## Step 5 — Get upstream commit via skopeo inspect

For each operator snapshot, find the **operator image** (not the bundle image)
in `.spec.components`:

| app | operator component name |
|---|---|
| `nhc-5-8` | `nhc-operator-5-8` |
| `snr-5-8` | `snr-operator-5-8` |
| `far-5-8` | `far-operator-5-8` |
| `mdr-5-8` | `mdr-operator-5-8` |
| `nmo-5-8` | `nmo-operator-5-8` |
| `sbr-5-8` | `sbr-operator-5-8` |

Run (parallel with `&` + `wait`):

```bash
skopeo inspect docker://<operator-image-ref> | jq '{
  upstream_version: .Labels["upstream-version"],
  vcs_ref: .Labels["vcs-ref"]
}'
```

- `upstream-version`: GitHub commit SHA (7-char) → upstream commit short ID
- `vcs-ref`: GitLab downstream commit SHA — use to cross-check Step 4 value

**Upstream GitHub repo mapping**:

| app prefix | medik8s repo |
|---|---|
| `nhc-` | `node-healthcheck-operator` |
| `snr-` | `self-node-remediation` |
| `far-` | `fence-agents-remediation` |
| `mdr-` | `machine-deletion-remediation` |
| `nmo-` | `node-maintenance-operator` |
| `sbr-` | `storage-based-remediation` |

Fetch the upstream commit date:

```bash
curl -s "https://api.github.com/repos/medik8s/<up-repo>/commits/<upstream-version>" \
  | jq '.commit.author.date'
```

**Key note**: if the downstream snapshot trigger commit is an automated
Konflux commit (title contains `mintmaker`, `chore(deps)`, `component-updates`,
or is a `Merge branch 'konflux/`), the operator code did NOT change and the
upstream commit reflects a base-image rebuild. This is normal and expected.

---

## Step 6 — Get IIB number from release status

```bash
kubectl get release <release-name> -n rhwa-tenant -o json | jq '
  .status.artifacts.advisory.url // .status.artifacts // .'
```

Look for a URL containing `iib.engineering.redhat.com` or an IIB build number
in the release status. The IIB number is the integer in that URL path.

Alternatively, check the release conditions:
```bash
kubectl get release <release-name> -n rhwa-tenant -o json | jq '
  .status.conditions[] | select(.type == "Released") | .message'
```

---

## Step 7 — Format output

Output three blocks:

### Block A — MR title fragment
```
(IIB <iib-number>)
```

### Block B — Markdown table

```markdown
| Snapshot | Downstream commit | Downstream date | Upstream commit | Upstream date |
|---|---|---|---|---|
| [nhc-5-8-YYYYMMDD-HHMMSS-000](<konflux-snapshot-url>) | [<short>](<gitlab-tree-url>) | YYYY-MM-DD HH:MM UTC | [<short>](<github-commit-url>) | YYYY-MM-DD HH:MM UTC |
...
```

Konflux snapshot URL: `https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/rhwa-tenant/applications/<app>/snapshots/<snapshot-name>`

GitLab tree URL: `https://gitlab.cee.redhat.com/dragonfly/<ds-repo>/-/tree/<full-downstream-sha>`

GitHub commit URL: `https://github.com/medik8s/<up-repo>/commit/<full-upstream-sha>`

For the full upstream SHA, expand the 7-char short ID using the GitHub API
response `.sha` field.

Order rows: NHC, SNR, FAR, MDR, NMO, SBR (matches dragonfly/tools MR convention).

### Block C — FBC catalog row

```markdown
FBC catalog:
- [<fbc-snapshot-name>](<konflux-snapshot-url>) ([<rhwa-fbc-short-commit>](<gitlab-tree-url>))
```

Where the GitLab tree URL is:
`https://gitlab.cee.redhat.com/dragonfly/rhwa-fbc/-/tree/<rhwa-fbc-full-commit>`

---

## Gotchas

- Multiple snapshots may match a bundle digest (retries create duplicates with
  random suffix like `-d9`, `-tq`). Use the clean name without suffix — it's
  the canonical one that passed all tests.
- `skopeo inspect` requires registry credentials; run from a machine with
  `podman login quay.io` or equivalent already configured.
- GitHub API rate-limits unauthenticated requests to 60/hour. Use
  `GITHUB_TOKEN` if available: `--header "Authorization: token $GITHUB_TOKEN"`.
- Upstream commit dates are in UTC; downstream dates may have timezone offset —
  normalize to UTC (`HH:MM UTC`) in the table.
- If `upstream-version` label is absent (older images), fall back to checking
  the most recent non-automated commit on the operator branch in the downstream
  repo (see `pac.test.appstudio.openshift.io/source-branch` annotation for
  branch name).
