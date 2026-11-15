---
name: troubleshoot_konflux
description: Troubleshoot Konflux bundle-nudge auto-merge failures in rhwa-fbc. Auto-triggered when the user mentions Konflux, nudge, rhwa-fbc, bundle-nudge, or validate-stage-release.
metadata:
  triggers:
    - Konflux
    - nudge
    - rhwa-fbc
    - bundle-nudge
    - validate-stage-release
    - ci-circuit-break
    - find_release
    - find_snapshot
    - retrigger
    - Mintmaker
---

# Troubleshoot Konflux Bundle-Nudge Auto-Merge

## Execution rules

**No required inputs** — run diagnostics automatically without asking
questions. If the user mentions a specific MR or operator, focus on that;
otherwise survey all open bundle-nudge MRs in `dragonfly/rhwa-fbc`.

Make every decision unilaterally:
- If a MR number is not given, list all open MRs with `bundle-nudge` label
  and diagnose each
- If the root cause is unambiguous (A/B problem, dropped nudge, circuit-break),
  apply the fix without asking
- If multiple fixes are possible, apply the most conservative one and note
  alternatives

**Surface warnings** during processing and in the final output:
- Print a one-line warning for each stuck MR or unexpected CI state
- Collect all warnings into a **Warnings** block at the end of the diagnosis

## Context: What this system does

`dragonfly/rhwa-fbc` is a GitLab repo that holds the File-Based Catalog (FBC) for RHWA operators.
Mintmaker creates **bundle-nudge MRs** (label: `bundle-nudge`, `configMap-nudging`) to update operator bundle digests in `update_bundle.sh`.
A scheduled retrigger job (every 30 min) auto-merges those MRs once all checks pass.

## The bundle-nudge lifecycle

```
Operator code change (CVE fix, feature)
  → Konflux builds operator + bundle in parallel (bundle is faster, 1-arch)
  → Snapshot A: bundle-triggered, old update_bundle.sh (references old operator digest)
  → Snapshot B: operator-triggered, same old bundle + new operator → EC FAILS (inconsistency)
  → Mintmaker detects new operator digest → creates 728-style MR in operator repo
  → 728-style MR merges → bundle rebuilds with updated update_bundle.sh
  → Snapshot C: bundle-triggered, settled — bundle CSV references new operator → EC PASSES
  → Mintmaker nudges rhwa-fbc with snapshot C's bundle digest
  → validate-stage-release finds snapshot C → Succeeded → regen-catalogs → auto-merge
```

**Snapshot naming convention used in this codebase:**
- **Snapshot A**: bundle-triggered, old bundle digest, old operator in snapshot → EC Succeeded (internally consistent)
- **Snapshot B**: operator/console-triggered, same old bundle + new operator → EC Failed (inconsistency)
- **Snapshot C**: bundle-triggered AFTER the 728-style MR merged, new settled bundle → EC Succeeded
- **Snapshot D**: bundle-triggered, same digest as C, may appear later from a no-op rebuild
- **Snapshot E**: operator-triggered, same digest as D's bundle + newer operator → EC Failed
- **Snapshot F**: next settled bundle after D/E cycle (from another 728-style MR)

## Key CI functions in `.gitlab-ci.yml`

### `find_snapshot()`
Fetches Konflux snapshots for `APP` containing the bundle `DIGEST`.

**MR!695 fix** (merged to main): adds `appstudio.openshift.io/component=${COMPONENT}` to the labelSelector, so only **bundle-triggered** snapshots are returned. This filters out operator/console-triggered snapshots (B, E) that would otherwise pollute the result.

```bash
# With MR!695:
labelSelector=appstudio.openshift.io/application=${APP},appstudio.openshift.io/component=${COMPONENT}
# Then jq-filters to snapshots containing the bundle digest
```

### `find_release()`
Finds the stage release object for each snapshot in `SNAP_NAMES` (newest first).

**Original bug**: stopped at first snapshot with ANY release, including Failed. Set `ci-circuit-break` on first Failed release found.

**MR!696 fix** (on main after reopen): skips snapshots with permanently Failed releases (`rel_st == "Failed"`), continues to older snapshots. Sets `ci-circuit-break` ONLY when ALL snapshots have permanently Failed releases.

```bash
find_release() {
  local rel_raw rel rname rel_st all_failed=true
  ...
  for snap in $SNAP_NAMES; do
    ...
    [ -z "$rname" ] && continue
    rel_st=$(...)
    if [ "$rel_st" = "Failed" ]; then
      continue          # skip Failed; try older snapshot
    fi
    all_failed=false    # found a non-Failed release
    SNAPSHOT="$snap"
    RELEASE_JSON="$rel"
    break
  done
  if [ -z "$RELEASE_JSON" ]; then
    if [ "$all_failed" = "true" ]; then
      # ALL snapshots have Failed releases → set ci-circuit-break
      curl ... --data '{"add_labels": "ci-circuit-break"}' ...
      fail_with_comment "All stage releases ... have permanently Failed."
    fi
    fail_with_comment "No release found ... Retry in 30 minutes."
  fi
}
```

### `retrigger-nudge-pipelines` (scheduled job, every 30 min)
- Skips MRs with `ci-circuit-break` label
- Detects new MR SHA (Mintmaker pushed update) → removes `ci-circuit-break`, retriggers
- Sets `ci-circuit-break` after 3 consecutive identical pipeline failures

### `regen-catalogs-on-nudge` (CI job on MR pipelines)
- Runs after `validate-stage-release` succeeds
- Triggers when `update_bundle.sh` changed in the MR
- Runs `gen_template.sh --gc <release>` to regenerate FBC catalog files
- Commits back to MR branch with `[ci skip]` so it doesn't loop
- If catalog already matches `update_bundle.sh` (pre-regenerated in same commit): exits cleanly, no commit added

## Why MRs get stuck: the A/B problem

1. Pipeline 1 runs on nudge MR → finds snapshot A (Succeeded) → `regen-catalogs` runs → adds commit
2. Snapshot B appears (operator-triggered, Failed release)
3. Pipeline 2 triggered by regen-catalogs commit → old code: finds snapshot B → Failed → sets `ci-circuit-break`
4. Retrigger skips the MR. Mintmaker sees non-Mintmaker commit (regen-catalogs) → "Edited/Blocked" → won't update MR
5. MR is permanently stuck

## Dropped-nudge problem: Succeeded bundles that never reach rhwa-fbc

**This is distinct from the A/B / circuit-break problem and is the more common gap.**

The rhwa-fbc nudge is **build-event driven**, writing the shared `update_bundle.sh`. When a bundle rebuilds in a **burst** (several Succeeded stage releases within minutes), only ONE nudge MR survives — the rest are dropped (concurrent nudges racing on the same file; see [[reference_troubleshoot_konflux_skill]] / RHWA-1986). The next nudge only fires on the **next isolated build**. So if a burst ends with a Succeeded bundle and then the component goes quiet for hours, that latest-green bundle is stranded: `update_bundle.sh` holds an older digest even though a newer green one exists.

**Confirmed repro (SNR, 2026-10-28):** 5 Succeeded SNR bundles that day — `9936eb6`(08:24), `5ba835c`(08:59), `1e857bd`(09:00), `25fc9bc`(09:01), `9b700cd`(09:15). Only the FIRST (`9936eb6`) was nudged (!746). The 4 newer ones — including the last green, `9b700cd` (commit 65f4e31) — got **no MR at all**. main sat at `9936eb6` ~22h until the next isolated build (`3e3750a`, 10-29 07:47 → !752).

### Authoritative procedure: find the latest Succeeded bundle digest

**NEVER use the snapshots list API to find the latest bundle.** The K8s list API returns arbitrary pages — `limit=100` on `/snapshots` gives 100 items in unpredictable order, NOT the 100 newest. You will silently miss the latest snapshot.

**Correct two-step procedure (validated 2026-11-08):**

```bash
KONFLUX_TOKEN=$(oc whoami -t)
K8S_BASE="https://api.stone-prod-p02.hjvn.p1.openshiftapps.com:6443/apis/appstudio.redhat.com/v1alpha1/namespaces/rhwa-tenant"

# Step 1: find the latest Succeeded stage release for each operator (use limit=200)
curl -sf --header "Authorization: Bearer ${KONFLUX_TOKEN}" \
  "${K8S_BASE}/releases?labelSelector=appstudio.openshift.io/application=<APP>&limit=200" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
items = d.get('items', [])
stage_ok = [r for r in items
            if r.get('spec',{}).get('releasePlan','').endswith('-stage')
            and next((c for c in r.get('status',{}).get('conditions',[]) if c.get('type')=='Released'), {}).get('reason') == 'Succeeded']
stage_ok.sort(key=lambda r: r.get('metadata',{}).get('creationTimestamp',''), reverse=True)
for r in stage_ok[:3]:
    print(r.get('metadata',{}).get('creationTimestamp','?')[:16],
          r.get('spec',{}).get('snapshot','?'),
          r.get('metadata',{}).get('name','?'))
"

# Step 2: fetch the top snapshot by name to get the bundle digest
curl -sf --header "Authorization: Bearer ${KONFLUX_TOKEN}" \
  "${K8S_BASE}/snapshots/<SNAPSHOT-NAME>" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for c in d.get('spec',{}).get('components',[]):
    if c.get('name') == '<BUNDLE-COMPONENT>':
        print(c.get('containerImage',''))
"
```

The release name printed in Step 1 is the Konflux UI release link:
`https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/rhwa-tenant/applications/<APP>/releases/<RELEASE-NAME>`

The snapshot name links to:
`https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/rhwa-tenant/applications/<APP>/snapshots/<SNAPSHOT-NAME>`

**Why limit=200 and not 100?** The releases list is also unordered server-side, but releases are fewer and sparser than snapshots. With 200 you reliably cover all recent activity for a single operator. If you get exactly 200 results, there may be a `continue` token — in practice, bursts of 10–20 releases per day mean 200 always covers at least the past week.

### Detect dropped nudges (the authoritative cross-check)
Compare every Succeeded stage release against the rhwa-fbc nudge MR digests.
Run the two-step procedure above for all 6 operators, then diff the resulting digests against `update_bundle.sh` on main.

**Rule for what counts as a real drop:** skipping an intermediate is fine when a newer bundle supersedes it *while an MR is open*; a drop is when the **latest-green bundle of a burst** has no MR and no newer MR supersedes it.

## Diagnosing a stuck bundle-nudge MR

### 1. Check labels — use the correct label
Nudge MRs are labeled **`konflux-nudge`** and **`configMap-nudging`** (sometimes also `bundle-nudge`). Do NOT filter on `bundle-nudge` alone — it silently misses most nudge MRs.
```bash
curl -s --header "PRIVATE-TOKEN: ${GITLAB_PRIVATE_TOKEN}" \
  "https://gitlab.cee.redhat.com/api/v4/projects/134358/merge_requests?state=opened&per_page=100" \
  | jq -r '.[] | select(.labels|index("konflux-nudge")) | "\(.iid) cb=\(.labels|index("ci-circuit-break")!=null) \(.title)"'
```

### ci-circuit-break is usually TRANSIENT — don't panic
The retrigger bot ADDS `ci-circuit-break` seconds after a fresh nudge MR is created (stage release not ready yet) and REMOVES it automatically at the next ~30-min tick once the release goes green. A circuit-break on a fresh morning build is normal self-healing. It only persists when the bundle's stage release **permanently Failed**. Check label history to tell them apart:
```bash
curl -s --header "PRIVATE-TOKEN: ${GITLAB_PRIVATE_TOKEN}" \
  "https://gitlab.cee.redhat.com/api/v4/projects/134358/merge_requests/<IID>/resource_label_events?per_page=100"
```
Manually removing a circuit-break on a genuinely-Failed bundle is pointless — the bot re-adds it within seconds after validate-stage-release fails again.

### 2. Check Konflux snapshots for the bundle digest
```bash
# Bundle-triggered snapshots only (needs MR!695 component filter logic)
curl -s --header "Authorization: Bearer ${KONFLUX_TOKEN}" \
  "${K8S_BASE}/snapshots?labelSelector=appstudio.openshift.io/application=<APP>,appstudio.openshift.io/component=<BUNDLE_COMPONENT>"
```
NOTE: the k8s list API does **not** sort server-side and paginates with a `continue` token — a small `limit` returns an arbitrary (often old) page. GET a specific snapshot by name when you know it; use the labeled release query (above) to find recent ones.

### 3. Check releases — fastest authoritative path is the MR's own CI
`validate-stage-release` on the MR prints the exact snapshot + release status + decision. To force a re-evaluation: remove `ci-circuit-break` and POST `/merge_requests/<IID>/pipelines`. It will NOT merge a non-Succeeded bundle, so this is safe. Konflux **prunes release records after ~a day**, so historical (>1 day) release status is often unrecoverable via the API — use the CI log or advisory records instead.

### 4. Identify snapshot type
- **Bundle-triggered**: `labels["appstudio.openshift.io/component"] = <bundle-component>`
- **Operator-triggered**: label has `<operator-component>`
- **Console-triggered**: label has `<console-component>`

## Fixing a stuck MR

### Case 1: ci-circuit-break set, new settled bundle not yet in MR

**Symptoms**: MR has old bundle digest, a 728-style downstream MR exists/has merged, no new Mintmaker nudge.

**Root cause**: Mintmaker blocked by regen-catalogs commit (Edited/Blocked notification).

**Fix**:
1. Find the settled bundle digest (snapshot C) from Konflux
2. Update `update_bundle.sh` + catalog files manually in ONE commit (pre-regenerate so CI regen is a no-op)
3. Remove `ci-circuit-break` label → retrigger picks it up

```bash
# Replace old digest with new across all affected files
MDR_OLD="<old_sha256>"
MDR_NEW="<new_sha256>"
for f in update_bundle.sh operator-versions.yaml \
         rhwa-catalogs/5.8-0/5.0/rhwa-template.yaml \
         rhwa-catalogs/5.8-0/5.0/catalog/<operator>/catalog.yaml; do
  sed -i "s/${MDR_OLD}/${MDR_NEW}/g" "$f"
done
git commit -S -m "chore(deps): update <bundle> to <short_digest> and regenerate catalogs"
```

### Case 2: ci-circuit-break set, all snapshots Failed

**Symptoms**: Every snapshot for this bundle digest has a Failed stage release.

**Root cause**: EC policy violation in the bundle itself (not an inconsistency). Requires upstream fix.

**Fix**: Fix the root cause, let Mintmaker push a new bundle digest with the fix.

### Case 3: No ci-circuit-break but MR not auto-merging

**Symptoms**: Recent pipelines all failing at `validate-stage-release`.

**Check**:
- Is the Konflux token valid? (`oc whoami -t` on the stone-prod-p02 context)
- Are the managed pipelineruns pruned? (look at release objects, not pipelineruns)
- Is `find_snapshot` returning empty? (wrong APP or COMPONENT name?)

## Operators and their bundle/component names (5.8 release)

| Operator | APP | Bundle component | Operator component |
|----------|-----|------------------|--------------------|
| MDR | `mdr-5-8` | `mdr-bundle-5-8` | `mdr-operator-5-8` |
| NHC | `nhc-5-8` | `nhc-bundle-5-8` | `nhc-operator-5-8`, `nhc-console-5-8` |
| SNR | `snr-5-8` | `snr-bundle-5-8` | `snr-operator-5-8` |
| FAR | `far-5-8` | `far-bundle-5-8` | `far-operator-5-8` |
| SBR | `sbr-5-8` | `sbr-bundle-5-8` | `sbr-operator-5-8` |
| NMO | `nmo-5-8` | `nmo-bundle-5-8` | `nmo-operator-5-8` |

## `update_bundle.sh` variable names (rhwa-fbc root)
```bash
export machine_deletion_remediation_5_8=quay.io/.../mdr-bundle-5-8@sha256:...
export node_healthcheck_operator_5_8=quay.io/.../nhc-bundle-5-8@sha256:...
# etc.
```

## Mintmaker "Edited/Blocked" behavior

Mintmaker (Renovate) won't rebase/update an MR that has commits from non-Mintmaker authors. It also won't create a duplicate open MR for the same component. So if an open bundle-nudge MR exists with a regen-catalogs commit:
- Mintmaker won't update it with a new bundle digest
- Mintmaker won't create a new MR
- Solution: manually update the MR with the settled bundle digest (pre-regen the catalogs inline)

## Relevant MRs in rhwa-fbc
- **MR!695** (merged): `find_snapshot` component filter — only considers bundle-triggered snapshots
- **MR!696** (merged): `find_release` skip-Failed — falls back to older Succeeded snapshot; circuit-breaks only when ALL Failed
- These two work together: !695 narrows candidates to bundle-triggered; !696 handles same-digest D/C scenarios

## Important notes
- New nudge MR pipelines use the `.gitlab-ci.yml` from the MR SOURCE BRANCH, not main
- MR branches predating MR!695/!696 need those fixes cherry-picked to be robust
- Retrigger detects new MR SHA → removes ci-circuit-break automatically
- The Konflux cluster context is `stone-prod-p02` (`oc config use-context <context>`)
- Namespace: `rhwa-tenant`
- GitLab project ID for rhwa-fbc: `134358`
