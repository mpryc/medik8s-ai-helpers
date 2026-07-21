# Errata Advisory Process

Detailed guide for the RHWA Errata advisory lifecycle.

## Overview

RHWA releases ship through Red Hat Errata advisories. The product ID is 180 (Red Hat Workload Availability).

## Advisory Types

| Type | Use Case |
|------|----------|
| **RHBA** | Bug fix advisory (most common) |
| **RHEA** | Enhancement advisory (new features, Y-stream) |
| **RHSA** | Security advisory (CVE fixes — requires `security-impact` field) |

## Lifecycle States

```
NEW FILES  →  QA  →  REL_PREP  →  SHIPPED LIVE
```

## Steps

1. **Ensure Konflux Release CRs are created and successful**
   - Use `create_release.sh` from `dragonfly/tools/releases/`
   - Verify all operator and FBC releases succeeded

2. **Verify container builds are attached to the advisory**
   - Check at `errata.devel.redhat.com/products/180`
   - Each operator and bundle should be listed

3. **Set CDN repos for advisory metadata**
   - Configure CDN content delivery repos

4. **Check Container Grade**
   - Target grade: **A**
   - Lower grades may require investigation

5. **Switch advisory state: NEW FILES → QA**
   - This makes builds available on CDN Staging

6. **Push to CDN Staging for QE testing**
   - QE should test against the **stage index image** (full `redhat-operator-index` after IIB merge)
   - NOT the intermediate FBC fragment from `quay.io/redhat-user-workloads/`

7. **QE testing and approval**
   - After QE signs off, state moves to `REL_PREP`

8. **Ensure Docs Approval**
   - Documentation team must approve the advisory

9. **Ship day: SHIPPED LIVE**
   - Advisory becomes publicly available
   - Images accessible from `registry.redhat.io`

## Post-Ship

After SHIPPED LIVE:
- Update FBC `operator-versions.yaml`: change bundle images from `quay.io` to `registry.redhat.io`
- Remove `update-bundle: true` flag from released entries
- Regenerate FBC catalog

## Release Channels

- **stable**: GA operators (default for customers)
- **candidate**: Tech preview operators

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
