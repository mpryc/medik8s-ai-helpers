# Release Workflow — End-to-End Guide

Complete step-by-step guide for releasing medik8s operators, covering both Y-stream (minor) and Z-stream (patch) releases across upstream, midstream, and downstream.

## Overview

A medik8s release flows through three environments:

```
Upstream (GitHub)  →  Midstream (GitLab)  →  Downstream (Konflux + registry.redhat.io)
     tag, build          version bump           hermetic build, FBC, Errata
     community PRs       bundle-hack            Release CRs, stage/prod
```

### JIRA-Driven Release Process

Each release is tracked by a parent JIRA (e.g., RHWA-1333 "RHWA-4.22-1 - Patch Release") with 7 child tasks that map directly to the workflow phases below. The audit command (`/medik8s-release:audit`) tracks progress against these same phases.

```
Phase 0: Upstream Release (prerequisite — before JIRA creation)
  ├── JIRA 01: Konflux Update
  ├── JIRA 02: Update Downstream Repo
  ├── JIRA 03: Verify Images
  ├── JIRA 04: Final Tests Before Release
  ├── JIRA 05: QE Testing
  ├── JIRA 06: Docs Checklist
  └── JIRA 07: GA Release
Post-GA: FBC Bootstrap for Next OCP (Y-stream only)
```

### Acceptance Criteria (Parent JIRA)
- Release status set to "Release Pending" until GA
- All 7 child tasks completed
- Retrospective held
- Support Matrix updated
- All release tickets closed

## Phase 0: Upstream Release (Prerequisite)

Done before the JIRA is created. Use `/medik8s-release:upstream`.

### Step 0a: Create Upstream Release Branch (Y-stream only, at Code Freeze)
```bash
git checkout main && git pull
git checkout -b release-0.13
git push origin release-0.13
```

### Step 0b: Upstream Release
- Create tags on upstream repos (`git tag -s v0.13.1`)
- Build & push images to `quay.io/medik8s`
- Create community catalog PRs (K8s + OKD)
- Publish GitHub Releases

### Step 0c: Verify Upstream Artifacts
- Operator image on `quay.io/medik8s/<op>:v<version>`
- Bundle image on `quay.io/medik8s/<op>-bundle:v<version>`
- Catalog image on `quay.io/medik8s/<op>-catalog:v<version>`
- Community PRs created/merged

## JIRA 01: Konflux Update

**JIRA title**: `01- Konflux Update - Update Patch Version for RPs`
**Responsible**: Engineering | **Consulted**: Software Production

For Z-stream (patch) releases, reuse existing Konflux CRs from the minor release. No new KRD or product-definitions MRs needed.

### What to do
1. **Verify existing CRs** in `rhwa-tenant` namespace:
   - ReleasePlans (RP) exist for stage and prod
   - ReleasePlanAdmissions (RPA) exist
   - Applications with components exist
2. **Update patch version** in ReleasePlans (synopsis field)
3. **Verify at Konflux UI**: https://konflux-ui.apps.stone-prod-p02.hjvn.p1.openshiftapps.com/ns/rhwa-tenant/release
4. **Verify at ArgoCD** that application and components are synced

### For Y-stream (new minor) only
- Create new MRs for KRD (`releng/konflux-release-data`)
- Create new product-definitions entries
- Create new RP/RPA pairs for the new minor version
- Example RP structure: `far-0-6/far-0-6-releaseplans.yaml`

### Acceptance Criteria
- [ ] RPs visible at Konflux UI under `rhwa-tenant`
- [ ] Application with components exists at Konflux UI and ArgoCD
- [ ] RP/RPA approved by release engineers

### Dependencies
- SP & Prod Sec Onboarding
- Konflux Pipeline Onboarding
- Comet Entry, Product Listing Configuration

### Resources
- https://konflux.pages.redhat.com/docs/users/releasing/preparing-for-release.html
- Slack: `#konflux-users`

## JIRA 02: Update Downstream Repo

**JIRA title**: `02- Update Downstream Repo for rhwa-<version>`
**Priority**: Blocker | **Responsible**: Engineering

Use `/medik8s-release:version-bump` and `/medik8s-release:downstream`.

### What to do

1. **Create midstream branch** (Y-stream only):
   ```bash
   git checkout snr-0-12 && git checkout -b snr-0-13
   git push origin snr-0-13
   # Set as default branch on GitLab
   ```

2. **Update upstream references** in `.gitmodules` (branch field)

3. **Bump version** using `scripts/update-version.sh`:
   - `.tekton/*.yaml` → `semver-version` parameter
   - `.tekton/*.yaml` → `upstream-minor-version` parameter
   - `Containerfile.*-bundle` → `ENV CI_VERSION`
   - `bundle-hack/test_vars.sh` → `export CI_VERSION=`

4. **Update `bundle-hack/update_bundle.sh`**:
   - `PREVIOUS_VERSION` — set to last released version
   - `SKIP_RANGE_LOWER` — set to `>= last EUS version` (Y-stream) or comment out (first Z-stream)
   - `DOCS_RHWA_VERSION` — update to current RHWA version
   - CSV image reference → point to new private image
   - (Optional) Update Containerfiles description/summary

5. **Update Tekton parameters** (`.tekton/*.yaml`)

6. **Create/update Tekton pipeline files** (Y-stream only — new minor needs new pipeline file pair)

7. **Update FBC**:
   Use `/medik8s-release:fbc`:
   - Add version to `operator-versions.yaml` (with `update-bundle: true`)
   - Add version to `releases.yaml`
   - Run `gen_template.sh` and `gen_fbc.sh`
   - Verify with `verify-bundle-images.sh`

### Acceptance Criteria
- [ ] Containerfiles exist for operator and bundle
- [ ] `bundle-hack/` directory set up with CSV modification files
- [ ] Tekton files link branch with KRD data (pull and push per component)
- [ ] Code changes trigger automatic image builds
- [ ] Code pushes on master trigger automatic bundle metadata PRs
- [ ] Green builds for each component
- [ ] Pass IntegrationTestScenarios (ITS)
- [ ] Pass EnterpriseContractPolicy (ECP) referenced in RPA

### Resources
- https://github.com/konflux-ci/olm-operator-konflux-sample
- https://konflux.pages.redhat.com/docs/users/getting-started/passing-ec.html
- Slack: `#konflux-users`, `#forum-konflux-migration`

## JIRA 03: Verify Images

**JIRA title**: `03- Verify images creation for rhwa-<version>`
**Priority**: Major | **Responsible**: Engineering

Use `/medik8s-release:verify-bundle` and `/medik8s-release:inspect`.

### What to do

1. **Test build pipelines** (on-push and on-pr):
   - Create builds → Builds succeed
   - Verify pipeline executed after successful build

2. **Test non-FBC stage release**:
   - Create Release object with operator + bundle snapshot
   - Verify release passes RPA's ECP (e.g., `snr-0-10-stage.yaml`)
   - Release pipeline succeeds

3. **Test FBC stage release**:
   - Create Release object with catalog snapshot
   - Verify release passes RPA's ECP (e.g., `snr-0-10-fbc-stage.yaml`)
   - Release pipeline succeeds

4. **Verify advisory creation**:
   - Check stage advisory at https://gitlab.cee.redhat.com/rhtap-release/advisories/-/tree/main/data/advisories/rhwa-tenant
   - Note: In Konflux, advisories are created at release time (when Release object is created)

5. **Verify disconnected environment support**:
   - Related images properly listed in catalog
   - Follow https://konflux-ci.dev/docs/end-to-end/building-olm/#automating-updates-for-image-references-in-the-csv

6. **Verify FIPS compliance**:
   - Pass `fbc-fips-check-oci-ta` in catalog builds
   - Operator binary built with: `CGO_ENABLED=1 GOFLAGS="-tags=strictfipsruntime,openssl" GOEXPERIMENT="strictfipsruntime"`

7. **Check component nudges**: https://konflux-ci.dev/docs/building/component-nudges/

### Acceptance Criteria
- [ ] Builds succeed and pass Enterprise Integration Tests
- [ ] Stage release pipeline for non-FBC snapshot succeeded
- [ ] Containers pushed to stage registry
- [ ] Stage release pipeline for FBC snapshot succeeded
- [ ] New IIB created for QE testing
- [ ] Stage advisory exists in advisory data repository
- [ ] FIPS compliance verified
- [ ] Disconnected environment support verified

### Resources
- https://gitlab.cee.redhat.com/releng/advisories
- https://konflux.pages.redhat.com/docs/users/getting-started/passing-ec.html

## JIRA 04: Final Tests Before Release

**JIRA title**: `04- Final Tests Before Release for rhwa-<version>`
**Priority**: Major | **Responsible**: Engineering | **Consulted**: Software Production

### What to do

1. **Review RP/RPA alignment**:
   - Minimize differences between stage and prod ECP, RP, RPA
   - Test advisory creation on stage (modify RP), then replicate to prod RP

2. **Green flow verification**:
   - Green builds with no unresolved warnings
   - Green stage Releases with no unresolved warnings

3. **Verify advisory content**:
   - Check at https://gitlab.cee.redhat.com/rhtap-release/advisories
   - Verify synopsis, description, type (RHSA/RHBA/RHEA), fixed issues

4. **JIRA release notes**:
   - Go over JIRA issues for the release
   - Set release note type, text, and status fields

5. **CVE and advisory type determination**:
   - Check if Go version bump or dependency updates resolve CVEs
   - If CVEs are fixed → change advisory type from RHBA to RHSA
   - Add `cves` section to release manifests with CVE IDs and Konflux component names
   - Add ProdSec vulnerability tracker tickets (RHWA-xxxx) to `issues.fixed`
   - Exclude CVEs per ProdSec guidance (no Red Hat CVE record, or API not used by operators)
   - Example: Go 1.26.5 bump → NHC/NMO/SBR/SNR changed from RHBA to RHSA

7. **Create GA Release YAMLs**:
   - Run `releases/create_release.sh <release-name> <fbc-app> <iib-number>` to generate from staged IIB
   - Release YAMLs are saved to `rhwa-releases/<year>/<release-name>/`
   - Manually add CVE data, tracker tickets, and adjust advisory type (RHBA/RHSA) as needed
   - Use standard naming: `manual-stage-release-snr-0-10-1-7` (stage), `snr-0-10-1-prod-ga-2` (prod)

7. **Install from stage**:
   - Deploy catalog: `scripts/deploy_catalog.sh <sha256:DIGEST>`
   - Install operators: `helper_scripts/install_rhwa_operators.sh --catsrc <name>`
   - Check first lines of logs and build labels (build date, version, commit)
   - Test trivial usage with NHC/remediator
   - Run e2e tests on the cluster

7. **Verify build labels**:
   ```bash
   skopeo inspect docker://quay.io/redhat-user-workloads/rhwa-tenant/<repo>/<component>@sha256:... | jq .Labels
   ```

8. **Follow release checklist**: https://docs.google.com/document/d/1IWuana7NS52hUXcXitGOAsU8JeitgmCW0Tlje9FW7CU

### Acceptance Criteria
- [ ] Stage and prod RP/RPA have minimal differences
- [ ] All fixable failures and warnings resolved
- [ ] Manual advisory content verified (description, synopsis, type, fixed issues)
- [ ] Operator installable from stage FBC with all metadata
- [ ] GA Release YAMLs prepared

### Resources
- Konflux User Advisor Google NotebookLM (for troubleshooting)
- https://konflux.pages.redhat.com/docs/users/releasing/policy-exceptions.html
- https://konflux.pages.redhat.com/docs/users/getting-started/passing-ec.html

## JIRA 05: QE Testing

**JIRA title**: `05- QE Testing for rhwa-<version>`
**Responsible**: Quality Engineering

### What to do

QE tests against the **stage index image** (full `redhat-operator-index` after IIB merge), NOT the intermediate FBC fragment.

1. **Deploy catalog**: `helper_scripts/deploy_iib.sh <IIB_ID>` or `scripts/deploy_catalog.sh <IIB_ID>` and apply IDMS
2. **Install operators**: `helper_scripts/install_rhwa_operators.sh --catsrc <name>`
3. **Create/update Google Sheet** with test status (e.g., "RHWA 25.9 Testing Status")

### Testing priorities (in order)
1. Disconnected environment testing (part of regression)
2. Share DAST scanning results
3. Critical bug verification
4. Feature verification
5. Remaining bug verification

### OpenShift upgrade matrix
1. **Direct install**: Install OCP n, install operator n, run regression
2. **Upgrade from n-1**: Install OCP n-1 + operator n-1 → upgrade OCP to n → run regression → upgrade operator to n → run regression
3. **EUS skip** (EUS versions only): Same as above but with n-2

### Before closing
- [ ] QE has Konflux access
- [ ] Advisory details reviewed in stage advisory repository
- [ ] All bugs attached to advisory are in verified state

### Dependencies
- Stage release with IIB created (from JIRA 03)

### Resources
- Testing instructions: https://docs.google.com/document/d/1v87MEPFpwFpydY5jqNifrliBNzl2eQ1hCT6rVaD8g84

## JIRA 06: Docs Checklist

**JIRA title**: `06- Ecosystem Documentation Release Checklist for rhwa-<version>`
**Priority**: Major | **Responsible**: Documentation and Engineering

### What to do

1. **Develop RHWA user guide updates**:
   - New features documentation
   - Bug fix descriptions
   - Known issues

2. **Create Release Notes** for the RHWA release

3. **Verify advisory metadata**:
   - Check stage advisory at https://gitlab.cee.redhat.com/rhtap-release/advisories
   - Has the right fixed issues attached
   - Related JIRA issues are attached and public
   - RHWA documentation version is accurate
   - Synopsis, topic, description, solution fields correct

4. **Advisory metadata placement**:
   - Boilerplate metadata → ReleasePlanAdmission (doesn't change between releases)
   - Version-specific metadata → ReleasePlan
   - Docs team reviews advisory metadata with Eng team

### Known limitations
- Advisory ID not available in advance (KONFLUX-5287)
- Formatting issues with Konflux advisories (TUSC-173)

### Resources
- https://konflux.pages.redhat.com/docs/users/releasing/releasing-with-an-advisory.html
- Stage advisories: https://gitlab.cee.redhat.com/rhtap-release/advisories
- Prod advisories: https://gitlab.cee.redhat.com/releng/advisories

## JIRA 07: GA Release

**JIRA title**: `07- GA Release for rhwa-<version>`
**Responsible**: Engineering, Documentation, Product Experience Engineer

### Release Day Procedure

1. **Follow release checklist**: https://docs.google.com/document/d/1IWuana7NS52hUXcXitGOAsU8JeitgmCW0Tlje9FW7CU

2. **Use Konflux escalation process** early and frequently: https://source.redhat.com/groups/public/konflux/usersupport#tab-escalation-process

3. **Release non-FBC snapshots FIRST** (one Release object per Application):
   - Advisory created at https://gitlab.cee.redhat.com/releng/advisories
   - Containers pushed to `registry.redhat.io`
   - Visible in Comet/RH Catalog
   - Images pullable but NOT yet in OCP catalogs

4. **Release FBC snapshots** (one per FBC application per OCP version):
   - OCP catalog updated with channels, CSVs, related images
   - Without this, operators are NOT installable from OperatorHub
   - **FBC snapshot is tied to non-FBC snapshot** — ensure same components are used

5. **Notify Docs team** after release pipeline succeeds → Docs publishes

### Release object naming convention
- Stage testing: `manual-stage-release-snr-0-10-1-7`
- Production GA: `snr-0-10-1-prod-ga-2`

### Post-GA cleanup
- Update FBC `operator-versions.yaml`: change bundle images from `quay.io` to `registry.redhat.io`
- Remove `update-bundle: true` flag from released entries
- Regenerate FBC catalog
- Tag downstream repos: `releases/tag_downstream.sh <fbc-app-name>`
- Verify post-release: `scripts/verify-post-release.sh <release-name>`
- Publish GitHub Releases (if not done in Phase 0)
- Announce release

### Acceptance Criteria
- [ ] Non-FBC release pipelines succeed
- [ ] Advisory visible at https://gitlab.cee.redhat.com/releng/advisories and customer portal
- [ ] Visible in Comet/RH Catalog
- [ ] FBC release pipelines succeed
- [ ] New versions visible in OperatorHub for related OCP catalogs

### Important notes
- In Konflux, advisories are created at release time (when Release object is created) — plan to create Release objects on the planned release day
- Use `rh-advisories` pipeline with necessary data in RPA

## Post-GA: Pre-Bootstrap FBC for Next OCP Versions (Y-stream only)

**Critical — do this immediately after Y-stream GA:**

Use `/medik8s-release:fbc-bootstrap`.

- Prepare FBC catalogs for next OCP versions (e.g., 4.23, 5.0) with the just-released operator versions
- Release them before OCP release-engineering bootstraps the new catalogs
- Failure to do this results in stale/unsupported versions appearing in new OCP clusters (see RHWA-616)
- See the "OCP Catalog Bootstrap Problem" section in the FBC Management skill for full details

## Z-Stream (Patch) Release

Same 7-phase JIRA flow but with these differences:

| Phase | Y-stream | Z-stream |
|-------|----------|----------|
| Phase 0 | Create release branch | Reuse existing branch |
| JIRA 01 | Create new RP/RPA | Reuse existing, update version |
| JIRA 02 | Create midstream branch, new Tekton files | Reuse branch, bump version only |
| JIRA 02 (skipRange) | Set SKIP_RANGE_LOWER | Comment out (first Z-stream) |
| JIRA 02 (FBC) | New release block in releases.yaml | No change to releases.yaml |
| JIRA 03-06 | Same | Same |
| JIRA 07 | Same | Errata may be auto-created |
| Post-GA | FBC bootstrap for next OCP | Not needed |

Key Z-stream points:
- Every Z-stream needs a new upstream git tag
- Tekton `semver-version` must be bumped via `scripts/update-version.sh`
- Comment out `SKIP_RANGE_LOWER` for first Z-stream after Y-stream

See Z-Stream Process skill for detailed differences.

## Catalog Artifacts

**Important distinction:**
- **FBC fragment** (Konflux component) = intermediate, RHWA-only catalog
- **Stage index** (release artifact) = full `redhat-operator-index` after IIB merge

QE must test against the stage index, not the intermediate fragment, to catch merge issues.

## Audit Your Progress

Run `/medik8s-release:audit --version X.Y.Z --ocp 4.22` at any time to see a live checklist of what's done and what remains. The audit stages map directly to the JIRA child task structure.

## Dragonfly Tools Scripts

The `dragonfly/tools` repo contains scripts used during the release process. Each maps to a JIRA phase:

| Script | JIRA Phase | Purpose |
|--------|------------|---------|
| `releases/create_release.sh` | 04, 07 | Generate Release CR YAMLs from IIB/snapshots |
| `releases/tag_downstream.sh` | 07 (post-GA) | Tag downstream GitLab repos from prod releases |
| `scripts/verify-post-release.sh` | 07 (post-GA) | Verify images on registry.redhat.io and OCP catalogs |
| `scripts/check-catalog.sh` | 03, 05 | Inspect RHWA operators in OCP catalog index |
| `scripts/deploy_catalog.sh` | 03, 04 | Deploy FBC fragment or IIB on test cluster |
| `scripts/mintmaker-toggle.sh` | 02 | Toggle MintMaker/Renovate on Konflux components |
| `scripts/update_rpm_lockfile.sh` | 02 | Generate rpms.lock.yaml for hermetic builds (FAR) |
| `helper_scripts/install_rhwa_operators.sh` | 04, 05 | Install all 6 RHWA operators for testing |
| `helper_scripts/deploy_iib.sh` | 05 | Deploy IIB catalog for QE testing |
| `helper_scripts/remove_rhwa_operators.sh` | 04, 05 | Remove operators (cleanup) |

### Key script usage by phase

**JIRA 02 (Update Downstream Repo)**:
- `scripts/mintmaker-toggle.sh` — disable MintMaker during version bumps, re-enable after
- `scripts/update_rpm_lockfile.sh` — regenerate `rpms.lock.yaml` for FAR (hermetic builds)

**JIRA 03 (Verify Images)**:
- `scripts/deploy_catalog.sh <sha256:DIGEST>` — deploy FBC fragment for stage testing
- `scripts/check-catalog.sh <OCP_VERSION>` — verify operators in OCP catalog index

**JIRA 04 (Final Tests)**:
- `releases/create_release.sh <release-name> <fbc-app> <iib-number>` — generate Release CR YAMLs from staged IIB (saves to `rhwa-releases/`)
- `scripts/deploy_catalog.sh <sha256:DIGEST>` — deploy catalog for pre-release validation
- `helper_scripts/install_rhwa_operators.sh` — install operators from stage for testing

**JIRA 05 (QE Testing)**:
- `helper_scripts/deploy_iib.sh <IIB_ID>` — deploy IIB catalog for QE
- `helper_scripts/install_rhwa_operators.sh` — install operators for QE regression
- `scripts/check-catalog.sh` — verify catalog contents

**JIRA 07 (GA Release)**:
- `releases/create_release.sh` — generate prod Release CR YAMLs
- `releases/tag_downstream.sh <fbc-app>` — tag downstream repos after prod release
- `scripts/verify-post-release.sh <release-name>` — verify images on registry.redhat.io

## Related Tasks (not child tasks of the release JIRA)

These are tracked separately but related to the release:
- Git tag downstream repos
- CI setup for new branches
- Community catalog PRs (K8s, OKD)
- SAST review (3 weeks before Feature Freeze)

## Resources

- Konflux User Advisor Google NotebookLM
- Slack: `#konflux-users`, `#forum-konflux-migration`
- https://konflux.pages.redhat.com/docs/users/releasing/releasing-products.html
