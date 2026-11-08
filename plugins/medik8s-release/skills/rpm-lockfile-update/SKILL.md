---
name: rpm-lockfile-update
description: Use when adding new architectures to rpms.lock.yaml for Konflux hermetic builds, or when RPM dependencies change and the lockfile needs regeneration.
---

# RPM Lockfile Update

Regenerate `rpms.lock.yaml` for RHWA/Medik8s operators using `rpm-lockfile-prototype`.

Reference: [Konflux Hermetic Builds](https://konflux-ci.dev/docs/advanced-how-tos/building/hermetic-builds/)
Tool repo: [rpm-lockfile-prototype](https://github.com/konflux-ci/rpm-lockfile-prototype)

## Execution rules

**Collect any missing required inputs upfront in a single message before
doing anything else.** Required inputs that cannot be inferred:
- `RHEL_HOST` — SSH hostname/IP of the registered RHEL box with subscriptions
- `REPO_PATH` — path to the operator repo on the RHEL box
- `BRANCH` — git branch to operate on

If the operator name and branch are already in context, infer
`REPO_PATH` from the convention `~/repos/<operator>/` and ask only for
`RHEL_HOST` if missing.

Once inputs are confirmed, run all steps automatically with zero questions.

Make every decision unilaterally:
- If adding arches and packages are arch-specific, generate partial lockfile
  for new arches only then merge
- If `rpms.in.yaml` already contains the new arch, skip adding it and proceed
  to regeneration
- Commit `rpms.in.yaml` + `rpms.lock.yaml` together in one commit

**Surface warnings** during processing and in the final output:
- Print a one-line warning for any `rpm-lockfile-prototype` resolution errors
- Collect all warnings into a **Warnings** block at the end of output

## When to Use

- Adding new CPU architectures (aarch64, ppc64le, s390x) to an operator build
- Updating RPM dependencies after package changes in `rpms.in.yaml`
- After modifying Containerfile `RUN microdnf install` lines that affect resolved packages

## Prerequisites

- SSH access to a registered RHEL box with valid subscriptions (container environments won't work — subscription-manager detects `/run/.containerenv`)
- `rpm-lockfile-prototype` installed on the RHEL box (`pip install rpm-lockfile-prototype`)
- The operator repo cloned on the RHEL box, on the correct branch

## Quick Reference

```
rpms.in.yaml   → input config (packages, arches, repo, containerfile)
rpms.lock.yaml → output lockfile (committed; resolved RPM URLs per arch)
redhat.repo    → DNF repo definitions for Red Hat CDN
```

## Workflow

### Step 1: Prepare rpms.in.yaml

If adding arches to an existing lockfile, check whether any packages are arch-specific. Example: FAR has x86_64-only cloud fence agents (`fence-agents-aws`, `fence-agents-azure-arm`, `fence-agents-gce`) that are conditionally installed in the Containerfile.

For arch-specific packages:
1. Back up: `cp rpms.in.yaml rpms.in.yaml.bak`
2. Create a modified `rpms.in.yaml` with:
   - Only the NEW arches (remove existing ones to avoid re-resolving)
   - Arch-specific packages removed
3. Generate lockfile for new arches only, then merge

For universal packages (all arches get the same packages):
1. Simply add the new arches to `rpms.in.yaml`
2. Regenerate the full lockfile

### Step 2: Run rpm-lockfile-prototype on the RHEL box

```bash
ssh root@<rhel-host>
cd <repo-path>
rpm-lockfile-prototype -f <Containerfile> rpms.in.yaml --outfile /tmp/new-arches.lock.yaml
```

The tool reads `rpms.in.yaml`, resolves all listed packages for each arch via DNF against Red Hat CDN, and writes the lockfile.

### Step 3: Merge lockfiles (if generating partial)

If you generated only new arches, merge them into the existing lockfile:

```bash
# The lockfile structure is:
# ---
# lockfileVersion: 1
# lockfileVendor: redhat
# arches:
# - arch: x86_64
#   packages: [...]
# - arch: s390x
#   packages: [...]

# Append new arch sections (skip header lines 1-4 of new file)
tail -n +5 /tmp/new-arches.lock.yaml >> rpms.lock.yaml
```

Verify all expected arches are present:
```bash
grep '^- arch:' rpms.lock.yaml
```

### Step 4: Restore and commit

1. Restore original `rpms.in.yaml` (it should have ALL arches and ALL packages): `cp rpms.in.yaml.bak rpms.in.yaml`
2. Restore `redhat.repo` if modified: `git checkout redhat.repo`
3. Commit both `rpms.in.yaml` (with new arches added) and `rpms.lock.yaml`

### Step 5: Push to GitLab MR branch

If the lockfile is large (>100KB base64), use a file-based payload for the GitLab API:

```python
import json
content = open('lockfile-b64.txt').read().strip()
payload = {
    'branch': '<mr-branch>',
    'content': content,
    'encoding': 'base64',
    'commit_message': 'Add <arch> RPM lockfile entries',
    'author_name': '...', 'author_email': '...'
}
with open('payload.json', 'w') as f:
    json.dump(payload, f)
```

```bash
curl --request PUT \
  --header "PRIVATE-TOKEN: ${GITLAB_PRIVATE_TOKEN}" \
  --header "Content-Type: application/json" \
  --data @payload.json \
  "https://gitlab.cee.redhat.com/api/v4/projects/<id>/repository/files/rpms.lock.yaml"
```

## Common Mistakes

- **Running in a container**: subscription-manager fails silently or errors. Always use a real RHEL host.
- **Forgetting arch-specific packages**: resolution fails with "missing packages" for arches that don't have them. Check the Containerfile for conditional `if [ "$(uname -m)" = ... ]` installs.
- **Using `$SSL_CLIENT_CERT` in redhat.repo**: these are not DNF variables — they're literal strings. If you must edit redhat.repo, use actual cert file paths.
- **Curl argument too long**: base64-encoded lockfiles exceed shell argument limits. Always use `--data @file.json`.
- **Not restoring rpms.in.yaml**: the committed version must list ALL arches and ALL packages (including arch-specific ones), since the Containerfile handles conditional installation.
