# Medik8s Release Plugin

The Medik8s Release plugin provides Claude Code commands for automating the full release lifecycle of medik8s / RHWA operators across upstream (GitHub), midstream (GitLab/dragonfly), and downstream (Konflux) environments.

## Commands

| Command | Description |
|---------|-------------|
| `/medik8s-release:status` | Check release readiness across all operators |
| `/medik8s-release:upstream` | Run upstream community release (tag, build, PRs) |
| `/medik8s-release:downstream` | Prepare midstream repos for downstream release |
| `/medik8s-release:fbc` | Update FBC catalog (versions, generate, verify) |
| `/medik8s-release:verify-bundle` | Validate bundle CSV, images, OLM metadata |
| `/medik8s-release:version-bump` | Bump version across midstream files |
| `/medik8s-release:sync` | Sync all local repos with remotes |
| `/medik8s-release:inspect` | Deep inspection of what a build contains (provenance, Go version, labels) |
| `/medik8s-release:audit` | Full release checklist with live progress tracking |

## Skills

| Skill | Description |
|-------|-------------|
| [release-workflow](skills/release-workflow/SKILL.md) | End-to-end release procedure (Y-stream and Z-stream) |
| [fbc-management](skills/fbc-management/SKILL.md) | FBC catalog generation and management |
| [downstream-prep](skills/downstream-prep/SKILL.md) | Midstream repository preparation guide |
| [errata-process](skills/errata-process/SKILL.md) | Errata advisory lifecycle and steps |
| [z-stream-process](skills/z-stream-process/SKILL.md) | Patch release specifics |

## Key Concepts

### Three-Repo Model
1. **Upstream** (GitHub/medik8s) — public development, community releases
2. **Midstream** (GitLab/dragonfly) — downstream modifications, Konflux config
3. **Downstream** (Konflux + registry.redhat.io) — hermetic builds, FBC, production

### FBC vs IIB
- **FBC fragment** = intermediate Konflux output, RHWA operators only
- **Stage index** = full `redhat-operator-index` after IIB merge (what QE tests)
- RHWA uses FBC exclusively for catalog definition; IIB is the merge step at release time

### Upstream Source Tracking
Container images carry `upstream-version` labels (e.g., `release-0.13-d6e798a`) linking back to source commits. Set by shared Tekton pipeline `rhwa-build-pipeline-multi.yaml`.

### Operator Short Names
SNR, NHC, FAR, MDR, NMO (version 5.x), SBR

## Prerequisites
- `gh` CLI (GitHub), `oc` CLI (Konflux), `podman`/`docker`, `yq`
- VPN for GitLab access
- Konflux auth for downstream operations

## Recommended Permissions

Claude Code cannot bundle permissions with plugins. Add these read-only patterns to your project's `.claude/settings.local.json` to avoid confirmation prompts during audit and status checks:

```json
{
  "permissions": {
    "allow": [
      "Bash(curl -sf:*)",
      "Bash(curl -s:*)",
      "Bash(skopeo inspect:*)",
      "Bash(gh api:*)",
      "Bash(gh pr:*)",
      "Bash(gh release:*)",
      "Bash(gh repo:*)",
      "Bash(git:*)",
      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(head:*)",
      "Bash(grep:*)",
      "Bash(find:*)",
      "Bash(yq:*)",
      "Bash(jq:*)",
      "Bash(oc get:*)",
      "Bash(oc describe:*)",
      "Bash(kubectl get:*)",
      "Bash(# Check:*)",
      "Bash(# Stage:*)",
      "Bash(# Audit:*)",
      "Bash(declare -A:*)"
    ]
  }
}
```
