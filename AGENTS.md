# AGENTS.md - Guide for AI Coding Agents

This guide helps AI coding agents navigate and contribute to the medik8s-ai-helpers repository, which contains Claude Code plugins for automating medik8s / RHWA release workflows.

## Repository Overview

This repository contains Claude Code plugins for medik8s release operations, organized under the `plugins/` directory. The primary plugin (`medik8s-release`) provides slash commands for managing the full release lifecycle across upstream GitHub repos, midstream GitLab repos, and downstream Konflux builds.

**Repository Structure:**
```
medik8s-ai-helpers/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace registration
├── plugins/
│   └── medik8s-release/
│       ├── .claude-plugin/
│       │   └── plugin.json       # Plugin metadata
│       ├── commands/
│       │   ├── status.md         # Release readiness check
│       │   ├── upstream.md       # Community release
│       │   ├── downstream.md     # Midstream preparation
│       │   ├── fbc.md            # FBC catalog management
│       │   ├── verify-bundle.md  # Bundle validation
│       │   ├── version-bump.md   # Version bumping
│       │   └── sync.md           # Repo syncing
│       ├── skills/
│       │   ├── release-workflow/
│       │   │   └── SKILL.md      # End-to-end release guide
│       │   ├── fbc-management/
│       │   │   └── SKILL.md      # FBC catalog details
│       │   ├── downstream-prep/
│       │   │   └── SKILL.md      # Midstream preparation guide
│       │   ├── errata-process/
│       │   │   └── SKILL.md      # Errata advisory lifecycle
│       │   └── z-stream-process/
│       │       └── SKILL.md      # Patch release specifics
│       └── README.md
├── AGENTS.md                      # This file
├── PLUGINS.md                     # Plugin command reference
└── README.md                      # Project overview

```

## Domain Knowledge: Medik8s / RHWA

### What is Medik8s?
Medik8s is a collection of Kubernetes operators for automated node remediation. The downstream product is Red Hat Workload Availability (RHWA), Errata product ID 180.

### Three-Repo Model
1. **Upstream** (GitHub/medik8s): Public development, community releases
2. **Midstream** (GitLab/dragonfly): Downstream modifications, Konflux pipeline config
3. **Downstream** (Konflux + registry.redhat.io): Hermetic builds, FBC catalogs, production images

### GA Operators
SNR (self-node-remediation), NHC (node-healthcheck-operator), FAR (fence-agents-remediation), MDR (machine-deletion-remediation), NMO (node-maintenance-operator, version scheme 5.x), SBR (storage-based-remediation)

### FBC Architecture (No IIBs)
RHWA uses File-Based Catalogs exclusively. The FBC repo (`dragonfly/rhwa-fbc`) manages:
- `releases.yaml` — release-to-operator-version mapping
- `operator-versions.yaml` — master version list with bundle images
- `update_bundle.sh` — latest Konflux-built bundle digests
- Per-OCP Containerfiles and Tekton pipelines

Key rule: staging images use `quay.io/redhat-user-workloads/` with `update-bundle: true`; released images use `registry.redhat.io/workload-availability/`.

### Konflux Build System
- Replaced CPaaS pipeline
- Hermetic builds with RPM lockfiles (`rpms.in.yaml` / `rpms.lock.yaml`)
- Tekton PipelineRuns via Pipelines-as-Code (`.tekton/` directories)
- MintMaker/Renovate for automatic dependency updates
- Release CRs created in `rhwa-tenant` namespace on `stone-prod-p02`

### Upstream Source Tracking
Container images carry an `upstream-version` label (e.g., `release-0.13-79fb792`) linking back to the upstream commit. This is set by the shared Tekton pipeline `rhwa-build-pipeline-multi.yaml`.

### Branch Naming Convention
Midstream branches: `<op-short>-<major>-<minor>` (e.g., `snr-0-13`, `far-0-8`, `nmo-5-7`)

### Release Types
- **Y-stream** (minor): New features, new OCP alignment (e.g., 0.12.0 → 0.13.0)
- **Z-stream** (patch): Bug fixes, CVE patches (e.g., 0.13.0 → 0.13.1)

### Network Requirements
- GitHub repos: public internet
- GitLab repos (`gitlab.cee.redhat.com`): requires Red Hat VPN
- Konflux cluster (`stone-prod-p02`): requires Red Hat VPN + `oc login`

## Plugin Conventions

### Command Definition Format
All commands in `plugins/{plugin-name}/commands/` use Markdown with YAML frontmatter:

```markdown
---
description: Brief description
argument-hint: [optional] [arguments]
---

## Name
plugin-name:command-name

## Synopsis
## Description
## Implementation
## Return Value
## Examples
## Arguments
## See Also
```

### Skills
Complex implementation details in `plugins/{plugin-name}/skills/{skill-name}/SKILL.md`.

## Contributing

### Adding a New Command
1. Create `plugins/medik8s-release/commands/{command-name}.md`
2. Follow the command definition format above
3. Update PLUGINS.md

### Adding a New Skill
1. Create `plugins/medik8s-release/skills/{skill-name}/SKILL.md`
2. Document step-by-step procedures
3. Include error handling and debugging

### Best Practices
1. **Be explicit**: Don't assume knowledge of medik8s internals
2. **Provide complete commands**: Include full bash commands with paths
3. **Handle network failures**: GitLab repos need VPN — detect and report
4. **Distinguish environments**: Always specify whether a step targets upstream (GitHub), midstream (GitLab), or downstream (Konflux)
5. **Show real examples**: Use actual operator names, versions, and branches
6. **Cross-reference**: Link to related commands and skills

## Available Plugins

| Plugin | Purpose | Key Commands |
|--------|---------|--------------|
| `medik8s-release` | Release automation | `/medik8s-release:status`, `/medik8s-release:upstream`, `/medik8s-release:downstream`, `/medik8s-release:fbc` |
