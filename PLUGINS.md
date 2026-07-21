# Available Plugins

This document lists all available Claude Code plugins and their commands in the medik8s-ai-helpers repository.

- [Medik8s Release](#medik8s-release-plugin)

### Medik8s Release Plugin

Release workflows for medik8s/RHWA operators across upstream, midstream, and downstream environments.

**Commands:**
- **`/medik8s-release:status` `[--version <version>] [--operator <op>]`** — Check release readiness across all operators (upstream tags, quay.io images, community PRs, midstream branches, FBC entries)
- **`/medik8s-release:upstream` `<operator> --version <version> --previous <prev> --ocp <ocp_version> [--skip-range-lower <ver>] [--dry-run]`** — Run upstream community release (tag, build & push images, create community catalog PRs)
- **`/medik8s-release:downstream` `<operator> --version <version> [--branch <midstream-branch>] [--dry-run]`** — Prepare midstream repo (update bundle-hack, verify Tekton files, commit and push)
- **`/medik8s-release:fbc` `--release <release-name> [--verify-only] [--dry-run]`** — Update FBC catalog (operator-versions.yaml, releases.yaml, generate templates and catalogs, verify bundle images)
- **`/medik8s-release:verify-bundle` `<operator> --version <version> [--registry <registry>]`** — Validate bundle CSV fields, skipRange/replaces consistency, image pullability, OLM annotations
- **`/medik8s-release:version-bump` `<operator> --version <version> [--branch <midstream-branch>]`** — Bump version across midstream files (Tekton semver-version, Containerfile CI_VERSION, bundle-hack test scripts)
- **`/medik8s-release:sync` `[--upstream] [--downstream] [--forks] [--dry-run]`** — Sync all local repos with remotes, report repos on non-default branches
- **`/medik8s-release:inspect` `<fbc-app-or-release> [--operator <op>] [--deep]`** — Inspect what a build contains: upstream commits, Go version, base images, labels, security posture
- **`/medik8s-release:audit` `--version <version> --ocp <ocp_version> [--type y|z] [--operator <op>]`** — Full release checklist with live progress tracking — shows what's done, what's remaining, and actionable next steps

See [plugins/medik8s-release/README.md](plugins/medik8s-release/README.md) for detailed documentation.
