# Medik8s AI Helpers

Claude Code plugins for automating medik8s / RHWA (Red Hat Workload Availability) release and maintenance workflows.

## Overview

This project provides [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugins for the medik8s operator ecosystem. It encapsulates domain knowledge needed to release, verify, and maintain 6 Kubernetes operators across upstream (GitHub), midstream (GitLab/dragonfly), and downstream (Konflux/registry.redhat.io) environments.

### What is Medik8s?

Medik8s is a collection of Kubernetes operators for automated node remediation, built with Go using the Kubebuilder / Operator SDK framework. The downstream product is **Red Hat Workload Availability (RHWA)**.

### GA Operators

| Short | Full Name | Notes |
|-------|-----------|-------|
| SNR | self-node-remediation | Self-remediation agent on each node |
| NHC | node-healthcheck-operator | Monitors node health, triggers remediation |
| FAR | fence-agents-remediation | Remediation via fence agents (IPMI, etc.) |
| MDR | machine-deletion-remediation | Remediation by deleting Machine objects |
| NMO | node-maintenance-operator | Node maintenance lifecycle (version scheme: 5.x) |
| SBR | storage-based-remediation | Storage-based remediation (newest) |

### Three-Repo Model

```
Upstream (GitHub/medik8s)  -->  Midstream (GitLab/dragonfly)  -->  Downstream (Konflux + registry.redhat.io)
  public development              downstream modifications            hermetic builds, FBC catalogs
```

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed and configured
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated with access to medik8s repos
- `oc` CLI authenticated to `stone-prod-p02` Konflux cluster (for downstream operations)
- `podman` or `docker` (for FBC generation and image inspection)
- `yq` (for YAML processing)
- VPN access (for GitLab/dragonfly midstream repos)

### Install the Plugin

From within a Claude Code session:

```
/plugin marketplace add mpryc/medik8s-ai-helpers
/plugin install medik8s-release@medik8s-ai-helpers
/reload-plugins
```

Verify:

```
/skills
```

You should see the `medik8s-release:*` commands listed.

### Uninstall

```
/plugin uninstall medik8s-release
/reload-plugins
```

## Usage

### Check release readiness

```
/medik8s-release:status --version 0.13.0
```

### Run upstream community release

```
/medik8s-release:upstream SNR --version 0.13.0 --previous 0.12.1 --ocp 4.22
```

### Bump version in midstream

```
/medik8s-release:version-bump SNR --version 0.13.1
```

### Prepare downstream midstream repos

```
/medik8s-release:downstream SNR --version 0.13.1
```

### Update FBC catalog

```
/medik8s-release:fbc --release 4.22-1
```

### Verify bundle correctness

```
/medik8s-release:verify-bundle SNR --version 0.13.1
```

### Sync all local repos

```
/medik8s-release:sync --upstream
```

## Available Commands

| Command | Description |
|---------|-------------|
| `/medik8s-release:status` | Check release readiness across all operators (images, tags, PRs, FBC entries) |
| `/medik8s-release:upstream` | Run upstream community release (tag, build, community PRs) |
| `/medik8s-release:downstream` | Prepare midstream repos (version bump, bundle-hack, Tekton update) |
| `/medik8s-release:fbc` | Update FBC catalog (operator-versions, releases, generate, verify) |
| `/medik8s-release:verify-bundle` | Validate bundle CSV, skipRange/replaces, image pullability |
| `/medik8s-release:version-bump` | Bump version across midstream files (wraps update-version.sh) |
| `/medik8s-release:sync` | Sync all local repos with remotes |

See **[PLUGINS.md](PLUGINS.md)** for argument details on each command.

## Quick Start: Full Y-Stream Release

### 1. Pre-release checks
```
/medik8s-release:status --version 0.13.0
```

### 2. Upstream release (for each operator)
```
/medik8s-release:upstream SNR --version 0.13.0 --previous 0.12.1 --ocp 4.22
```

### 3. Downstream midstream preparation
```
/medik8s-release:version-bump SNR --version 0.13.0
/medik8s-release:downstream SNR --version 0.13.0
```

### 4. FBC update
```
/medik8s-release:fbc --release 4.22-0
```

### 5. Verify everything
```
/medik8s-release:verify-bundle SNR --version 0.13.0
```

## Key Concepts

### FBC (File-Based Catalog)

RHWA uses File-Based Catalogs (FBC) exclusively — IIBs (Index Image Bundles) are no longer generated. The FBC repo (`dragonfly/rhwa-fbc`) contains:
- `releases.yaml` — which operator versions belong to each RHWA release
- `operator-versions.yaml` — master list of versions with bundle images and OLM metadata
- `update_bundle.sh` — latest Konflux-built bundle digests (auto-updated by MintMaker)
- Per-OCP Containerfiles and Tekton pipelines

### Konflux

Konflux replaced CPaaS as the build system. Key facts:
- Builds are hermetic (RPM lockfiles, source pinning)
- Images land in `quay.io/redhat-user-workloads/rhwa-tenant/`
- After GA, images are mirrored to `registry.redhat.io/workload-availability/`
- Release CRs are created in the `rhwa-tenant` namespace on `stone-prod-p02`
- The `create_release.sh` script in `dragonfly/tools` orchestrates Release CR creation

### Upstream Source Tracking

Container images include an `upstream-version` label (e.g., `release-0.13-79fb792`) set by the shared Tekton pipeline (`rhwa-build-pipeline-multi.yaml`). This links the built image back to the upstream source commit.

## Additional Documentation

- **[PLUGINS.md](PLUGINS.md)** — Complete list of all available commands with arguments
- **[AGENTS.md](AGENTS.md)** — Guide for AI agents working with this repository
- **[plugins/medik8s-release/skills/](plugins/medik8s-release/skills/)** — Detailed implementation guides

## Related Repositories

### Upstream (GitHub)
- [medik8s/.github](https://github.com/medik8s/.github) — Release workflows, community release script
- [medik8s/self-node-remediation](https://github.com/medik8s/self-node-remediation), [node-healthcheck-operator](https://github.com/medik8s/node-healthcheck-operator), etc.
- [medik8s/community-operators](https://github.com/medik8s/community-operators) — K8s OperatorHub fork
- [medik8s/community-operators-prod](https://github.com/medik8s/community-operators-prod) — OKD/OCP catalog fork

### Midstream (GitLab)
- `gitlab.cee.redhat.com/dragonfly/<operator>` — Midstream repos with downstream modifications
- `gitlab.cee.redhat.com/dragonfly/rhwa-fbc` — File-Based Catalog
- `gitlab.cee.redhat.com/dragonfly/tools` — Release scripts, shared Tekton pipelines
- `gitlab.cee.redhat.com/dragonfly/rhwa-release-info` — Release tracking and CVE analysis
