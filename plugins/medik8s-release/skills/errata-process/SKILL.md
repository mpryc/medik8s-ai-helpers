# Errata Advisory Process

Detailed guide for the RHWA Errata advisory lifecycle. Maps to JIRA steps 03 (Verify Images), 04 (Final Tests), 06 (Docs Checklist), and 07 (GA Release).

## Overview

RHWA releases ship through Red Hat Errata advisories. The product ID is 180 (Red Hat Workload Availability).

**Key Konflux difference**: In Konflux, advisories are created at release time (when a Release object is created), NOT in advance. Plan to create Release objects on the planned release day.

## Advisory Types

| Type | Use Case | JIRA 04 check |
|------|----------|---------------|
| **RHBA** | Bug fix advisory (most common) | Advisory ships bug fixes (plus potentially enhancements) |
| **RHEA** | Enhancement advisory (new features, Y-stream) | Advisory ships enhancements only |
| **RHSA** | Security advisory (CVE fixes — requires `security-impact` field) | Advisory ships security fixes |

## Lifecycle in Konflux

```
Create Release object  →  Advisory auto-created  →  Stage testing  →  Prod release  →  Customer portal
```

Unlike the old CPaaS model, there is no separate advisory creation step. The Release object triggers both the release pipeline and advisory creation simultaneously.

## Advisory Data Repositories

| Environment | Repository |
|-------------|-----------|
| **Stage** | https://gitlab.cee.redhat.com/rhtap-release/advisories |
| **Production** | https://gitlab.cee.redhat.com/releng/advisories |

## Steps by JIRA Phase

### JIRA 03: Verify Images — Stage Advisory
1. **Test non-FBC stage release**: Create Release object → advisory auto-created in stage repo
2. **Verify stage advisory content** at `rhtap-release/advisories/-/tree/main/data/advisories/rhwa-tenant`
3. **Verify advisory metadata**: synopsis, description, type, fixed issues

### JIRA 04: Final Tests — Advisory Preparation
1. **Review advisory content** (description, synopsis, type)
2. **Verify JIRA fixed issues** are attached with correct release note type/text/status
3. **Verify advisory type** matches content:
   - RHSA if security fixes included (e.g., Go version bump resolving CVEs)
   - RHBA if bug fixes (plus potentially enhancements)
   - RHEA if enhancements only
4. **CVE handling** (when Go or dependency bumps resolve CVEs):
   - Add `cves` section to release manifests with CVE IDs and Konflux component names
   - Add ProdSec vulnerability tracker tickets to `issues.fixed`
   - Change advisory type from RHBA → RHSA for affected operators
   - Exclude CVEs per ProdSec guidance (no Red Hat CVE record, or API not used)
5. **Minimize stage/prod RP differences** — test advisory creation on stage (modify RP), then replicate to prod

### JIRA 06: Docs Checklist — Advisory Metadata
1. **Advisory metadata placement**:
   - Boilerplate (synopsis, topic, description, solution) → ReleasePlanAdmission
   - Version-specific info → ReleasePlan
2. **Docs team reviews** advisory metadata with Eng team
3. **Known limitation**: Advisory ID not available in advance (KONFLUX-5287)
4. **Known limitation**: Formatting issues with Konflux advisories (TUSC-173)

### JIRA 07: GA Release — Production Advisory
1. **Release non-FBC snapshot FIRST** → advisory created in prod repo
2. **Verify advisory** at `releng/advisories/-/tree/main/data/advisories/rhwa-tenant`
3. **Verify customer portal visibility** at `https://access.redhat.com/errata/RHBA-20XY:ABCD`
4. **Release FBC snapshots** → OCP catalogs updated
5. **Notify Docs team** → publish documentation

## Post-Ship

After production release:
- Update FBC `operator-versions.yaml`: change bundle images from `quay.io` to `registry.redhat.io`
- Remove `update-bundle: true` flag from released entries
- Regenerate FBC catalog

## Release Channels

- **stable**: GA operators (default for customers)
- **candidate**: Tech preview operators

## Release Object Naming Convention

| Purpose | Pattern | Example |
|---------|---------|---------|
| Stage testing | `manual-stage-release-<op>-<ver>` | `manual-stage-release-snr-0-10-1-7` |
| Production GA | `<op>-<ver>-prod-ga-<n>` | `snr-0-10-1-prod-ga-2` |

## Monitoring Channels

- `#team-release-framework-openshift-svc` — release framework support
- `#announce-cpaas` — CPaaS/Konflux announcements
- `#rel-eng` — release engineering
- `#forum-errata-guild` — errata questions
- CVP Google Space — container verification pipeline

## Common Issues

| Problem | Solution |
|---------|----------|
| Missing CDN repos | Update errata_releases→product_versions |
| Container grade < A | Investigate CVE/compliance issues |
| QE testing wrong image | Ensure QE uses stage index, not FBC fragment |
| Docs not approved | Follow up with docs team before ship day |
| Advisory ID not available in advance | Known Konflux limitation (KONFLUX-5287) — plan release day accordingly |
| Advisory formatting issues | Known issue (TUSC-173) — Teamnado working on it |
| `fbc-target-index-pruning-check` failure | Expected when old/unsupported versions exist in OCP catalog from bootstrap. Submit exception MR to `releng/konflux-release-data` with per-arch image digests. See RHWA-616 and FBC Management skill for details |

## Resources

- Releasing with advisory: https://konflux.pages.redhat.com/docs/users/releasing/releasing-with-an-advisory.html
- Releasing products: https://konflux.pages.redhat.com/docs/users/releasing/releasing-products.html
- Konflux User Advisor Google NotebookLM (for troubleshooting)
- Konflux escalation: https://source.redhat.com/groups/public/konflux/usersupport#tab-escalation-process
