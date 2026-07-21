# Downstream Midstream Preparation

Detailed guide for preparing midstream (GitLab/dragonfly) operator repositories for a downstream release.

## Repository Structure

Each midstream operator repo contains:
```
├── scripts/
│   └── update-version.sh          # Version bump script
├── bundle-hack/
│   ├── update_bundle.sh           # CSV customization (replaces, skipRange, branding)
│   ├── add-files/                 # OCP-specific manifests
│   ├── test_vars.sh               # Test CI_VERSION
│   └── test_update_bundle.sh      # Alternative test script
├── .tekton/
│   ├── <op>-operator-<ver>-push.yaml
│   ├── <op>-operator-<ver>-pull-request.yaml
│   ├── <op>-bundle-<ver>-push.yaml
│   └── <op>-bundle-<ver>-pull-request.yaml
├── Containerfile                  # Operator image
├── Containerfile.*-bundle         # Bundle image
├── rpms.in.yaml                   # RPM dependencies (FAR)
└── rpms.lock.yaml                 # RPM lockfile (hermetic builds)
```

## Version Bump Process

### scripts/update-version.sh

This script updates the hardcoded version in multiple files:
- `.tekton/*.yaml` → `semver-version` parameter (e.g., `v0.13.1`)
- `.tekton/*.yaml` → `upstream-minor-version` parameter (e.g., `13`)
- `Containerfile.*-bundle` → `ENV CI_VERSION`
- `bundle-hack/test_vars.sh` → `export CI_VERSION=`

**Critical**: The `.tekton/` files hardcode the full semver version. This MUST be bumped for every release (including Z-stream patches).

### bundle-hack/update_bundle.sh

This script customizes the CSV for downstream:
- `DOCS_RHWA_VERSION` — RHWA docs version (e.g., `4.22-0`)
- `PREVIOUS_VERSION` — OLM `replaces` field
- `SKIP_RANGE_LOWER` — OLM `skipRange` lower bound
- `OPERATOR_COMPONENT` — operator image reference
- Branding changes (Medik8s → Dragonfly)
- Documentation URL updates to Red Hat docs
- OCP-specific annotations and labels

### skipRange Rules
- **Y-stream**: set `SKIP_RANGE_LOWER` to last EUS version
- **First Z-stream after Y-stream**: comment out `SKIP_RANGE_LOWER`
- **PREVIOUS_VERSION**: always set to the last released version

## Upstream Source Tracking

Container images carry an `upstream-version` label set by the shared Tekton pipeline (`rhwa-build-pipeline-multi.yaml`). Format: `release-<minor>-<short-commit>` (e.g., `release-0.13-d6e798a`).

This label links the built image to the upstream source commit, enabling traceability from any shipped image back to its source code.

## Branch Naming

Midstream branches follow: `<op-short>-<major>-<minor>`
- `snr-0-13`, `nhc-0-12`, `far-0-8`, `mdr-0-7`, `nmo-5-7`, `sbr-0-3`

## RPM Management (FAR)

FAR has RPM dependencies for fence agents:
- `rpms.in.yaml` — lists required packages (some architecture-specific)
- `rpms.lock.yaml` — generated lockfile for hermetic Konflux builds
- MintMaker/Renovate handles automatic updates
