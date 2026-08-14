# FBC (File-Based Catalog) Management

Detailed guide for managing the RHWA File-Based Catalog in `dragonfly/rhwa-fbc`.

## Architecture

RHWA uses FBC to define which operator versions are available in each OCP release. The FBC repo produces **catalog fragment images** — intermediate artifacts containing only RHWA operators. When released to stage/prod, these fragments are merged into the global `redhat-operator-index` via IIB.

### Two Catalog Artifacts

| Artifact | Registry | Contains | Use |
|----------|----------|----------|-----|
| FBC fragment | `quay.io/redhat-user-workloads/rhwa-tenant/rhwa-fbc/` | RHWA operators only | Development, CI |
| Stage index | Release artifact (IIB-merged) | Full `redhat-operator-index` | QE testing, production |

### Image Registry Rules

- **Pre-GA**: Bundle images use `quay.io/redhat-user-workloads/` with `update-bundle: true`
- **Post-GA**: Bundle images use `registry.redhat.io/workload-availability/` — remove `update-bundle: true`
- `gen_fbc.sh` has a `fix_prod_image()` function that handles the quay→registry.redhat.io mapping

## Key Files

### releases.yaml
Maps RHWA release names to OCP versions and operator versions:
```yaml
releases:
  - name: "4.22-1"
    ocp:
      - version: "4.22"
        operators:
          - name: "self-node-remediation"
            versions:
              - version: "0.13.0"
              - version: "0.13.1"
```

### operator-versions.yaml
Master list of all operator versions with OLM metadata:
```yaml
operators:
  - name: self-node-remediation
    versions:
      - name: 0.13.1
        replaces: 0.13.0
        skip-range: ">=0.11.0 <0.13.1"
        bundle-image: quay.io/redhat-user-workloads/rhwa-tenant/.../snr-bundle-0-13@sha256:...
        update-bundle: true
```

### update_bundle.sh
Exports latest Konflux-built bundle digests. Auto-updated by MintMaker/Renovate:
```bash
export self_node_remediation_0_13=quay.io/redhat-user-workloads/rhwa-tenant/.../snr-bundle-0-13@sha256:...
```

## Generation Pipeline

1. `gen_template.sh` → reads `releases.yaml` + `operator-versions.yaml` + `update_bundle.sh` → produces `rhwa-template.yaml`
2. `gen_fbc.sh` → reads template → runs `opm render-template` per OCP version → produces catalogs in `rhwa-catalogs/<release>/<ocp>/catalog/`
3. `verify-bundle-images.sh` → validates all bundle images are pullable and use correct registries

## Per-OCP Containerfiles

Each OCP version has its own Containerfile:
```dockerfile
FROM registry.redhat.io/openshift4/ose-operator-registry-rhel9:v4.22
ENTRYPOINT ["/bin/opm"]
CMD ["serve", "/configs", "--cache-dir=/tmp/cache"]
ADD rhwa-catalogs/4.22-1/4.22/catalog /configs
RUN ["/bin/opm", "serve", "/configs", "--cache-dir=/tmp/cache", "--cache-only"]
```

## Adding a New Version

1. Add entry to `operator-versions.yaml` with `update-bundle: true`
2. Add version to appropriate release in `releases.yaml`
3. Run `./gen_template.sh && ./gen_fbc.sh`
4. Run `./verify-bundle-images.sh`
5. Commit and push — Tekton builds FBC image automatically

## Post-GA Cleanup

After images are mirrored to `registry.redhat.io`:
1. Update `operator-versions.yaml`: change `bundle-image` to `registry.redhat.io/...`
2. Remove `update-bundle: true`
3. Regenerate catalog

## OCP Catalog Bootstrap Problem (RHWA-616)

### The Problem

When a new OCP version (e.g., 4.23) is bootstrapped by release-engineering, the catalog is created from the previous OCP catalog. The bootstrap happens ~2-3 weeks after the previous OCP branches out (~5 months before GA, ~1 month before the previous OCP GA).

**IIB Opt-in Logic:** A package is considered FBC opted-in only when its first release is done via Konflux. This triggers removal of the package from the hidden `index.db`, enabling proper pruning of older bundles. If the package is NOT opted-in at bootstrap time, IIB falls back to including ALL historically supported bundles based on the `com.redhat.openshift.versions` label — resulting in very old, unsupported versions (e.g., NHC v0.3.1 from 2022) appearing in new OCP catalogs.

**The timing mismatch:** RHWA releases are scheduled around OCP GA date (Platform Aligned support), but OCP catalogs are bootstrapped ~5 months earlier. This creates a window where stale versions are available to customers.

**`fbc-target-index-pruning-check` failures:** FBC releases will fail this Konflux task when old versions exist in the OCP catalog that get purged by the new release. This can be bypassed with an exception MR to `releng/konflux-release-data`, but signals the underlying problem.

This has happened repeatedly: RHWA-616 (general), RHWA-658 (4.21 fix), RHWA-721 (4.22 fix), RHWA-1111 (4.23/5.0 fix).

### The Fix: Pre-Bootstrap FBC Release

After each RHWA Y-stream release, immediately prepare FBC catalogs for the **next** OCP versions (before they are bootstrapped):

1. Create FBC entries for upcoming OCP versions (e.g., 4.23, 5.0) containing the **current supported operator versions** from the just-released Y-stream
2. Do NOT include the upcoming Y-stream versions (they won't be ready yet)
3. Release these FBC catalogs via Konflux Release CRs with an exception request
4. This must happen **before** OCP release-engineering bootstraps the new OCP catalogs

### Timing

```
RHWA 4.22-0 GA  →  Immediately prepare FBC for 4.23/5.0  →  OCP 4.23 bootstrap
     ↑                        ↑                                    ↑
  our release         our pre-bootstrap fix              their catalog seeding
```

If you miss this window (OCP bootstrap happens first), a corrective FBC release is needed — same procedure but requires additional coordination since the catalog already has stale versions.

### Post-Bootstrap Corrective Release

If OCP bootstrapping already happened with stale versions:

1. Create a corrective FBC release with all supported versions for that OCP
2. Request an exception via `konflux-release-data` MR (see RHWA-658 for example)
3. Use `skopeo inspect --raw` to get per-architecture digests for the FBC snapshot image
4. Include both per-arch digests AND the OCI index (manifest list) digest in the exception
5. Get two approvals on the `konflux-release-data` MR

### Monitoring

Watch these channels for bootstrap announcements:
- `operatorframework-pgm@redhat.com` Google Group — OCP catalog bootstrap notifications
- `#forum-hello-operator-guild` — operator framework guidance
- `#announce-cpaas` — CPaaS/Konflux announcements

### Reference JIRAs

| JIRA | Description |
|------|-------------|
| RHWA-616 | Root cause: outdated operator versions in new OCP catalogs |
| RHWA-658 | Fix for OCP 4.21 catalog (before 4.21 GA) |
| RHWA-721 | Fix for OCP 4.22 catalog (after bootstrap) |
| RHWA-1111 | Fix for OCP 5.0 + 4.23 catalogs |
