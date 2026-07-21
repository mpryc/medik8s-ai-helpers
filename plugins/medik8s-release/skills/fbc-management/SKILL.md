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
