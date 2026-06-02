# Daily base-image CVE patching with Copacetic (PoC)

This document describes a proof of concept that keeps ahead of CVEs landing in
our base image (SUSE BCI/SLES 16.0) by **automatically patching the base image
layer every day** with [Copacetic](https://github.com/project-copacetic/copacetic)
via the [`copa-action`](https://github.com/project-copacetic/copa-action).

The workflow lives at
[`.github/workflows/base-image-copa-patch.yaml`](../../.github/workflows/base-image-copa-patch.yaml).

## Why

- The controller image (`nginx-ingress-controller`) is built `FROM ${BASE_IMAGE}`
  (`rancher/nginx:<NGINX_TAG>`), which is itself built `FROM
  registry.suse.com/bci/bci-base:16.0` (see `images/nginx/rootfs/Dockerfile`).
- CVEs in that BCI/SLES layer are **OS package** vulnerabilities — exactly what
  Copa patches. App-layer (Go / Lua) CVEs are intentionally out of scope here.
- The existing `vulnerability-scans.yaml` workflow scans published images weekly
  and reports SARIF, but never acts. This workflow is the daily,
  action-taking counterpart, scoped to the base image layer.

## Trigger

- `schedule`: daily at `07:00 UTC`.
- `workflow_dispatch`: manual runs, with an optional `version` input (e.g.
  `v1.15.1`). When omitted, the release line is read from the repository `TAG`
  file.

## What it does

### Stage 1 — Scan & gate (`scan` job)

1. Logs in to the Prime registry (Vault for `rancher`, GitHub secrets for forks)
   and resolves the source image: the **latest `primeX` generation** (and its
   latest `patchN`, if any) for the target release line in the Prime repo.
2. Runs Trivy twice against that image:
   - a SARIF scan that is uploaded to GitHub code scanning (preserving current
     behavior); and
   - a **JSON** scan restricted to `--pkg-types os`, `--severity HIGH,CRITICAL`,
     `--ignore-unfixed`. This JSON is both the gate input and the report fed to
     Copa.
3. Counts the fixable HIGH/CRITICAL OS vulnerabilities. If zero, the workflow
   exits cleanly (nothing to patch). If greater than zero, it sets
   `should_patch=true` and uploads the JSON report as an artifact.

Restricting to `--pkg-types os` is what scopes the gate to the **base image
layer**; `--ignore-unfixed` ensures we only fire on CVEs Copa can actually fix,
which also avoids re-patch loops.

### Stage 2 — Compute the Prime tag (`patch` job)

The patched image is named:

```
nginx-ingress-controller:vX.Y.Z-primeX-patchN
```

- `vX.Y.Z` — the release line (e.g. `v1.15.1`).
- `primeX` — **the latest prime generation already present in the Prime repo**
  for that release line (discovered via `skopeo list-tags`). If no
  `vX.Y.Z-primeX` image exists, the workflow fails: we patch an existing
  released prime image, we do not invent one.
- `patchN` — the next patch iteration: `max(existing patchN for primeX) + 1`,
  starting at `patch1` when none exists yet. Patches accumulate by always
  sourcing the newest existing patch of the latest prime generation.

### Stage 3 — Patch & push to Prime only

1. `copa-action` patches the OS packages of the source image using the Trivy
   JSON report.
2. The resulting image is tagged `…-primeX-patchN` and pushed to the **Prime
   registry only**. There is deliberately no DockerHub login/push, consistent
   with the existing `-prime` suffix policy (prime-suffixed tags never go to
   DockerHub).
3. The patched image is re-scanned (OS, HIGH/CRITICAL, fixable) and a
   before/after CVE count is written to the job summary to prove the patch
   worked.

## Running it manually

From the GitHub Actions UI: select **Base Image Copa Patch (PoC)** →
**Run workflow** → optionally set `version` (e.g. `v1.15.1`) → **Run**.

The job summary will show:

- the source image scanned and the fixable HIGH/CRITICAL OS CVE count (gate);
- the computed `…-primeX-patchN` tag;
- the before/after CVE counts for the pushed, patched image.

## Required secrets

The Prime credentials mirror those already used by `release.yml`:

| For `rancher` (Vault path under `rancher-prime-registry/credentials`) | For forks (GitHub Actions secrets) |
| --------------------------------------------------------------------- | ---------------------------------- |
| `registry`                                                            | `PRIME_REGISTRY`                   |
| `username`                                                            | `PRIME_REGISTRY_USERNAME`          |
| `password`                                                            | `PRIME_REGISTRY_PASSWORD`          |

No DockerHub credentials are needed or used by this workflow.

## Known risks & follow-ups

- **Copa zypper/SLES support is the main feasibility risk.** Copa's OS patching
  matured first on apt/apk/dnf; zypper/SLES(BCI) support is newer. The first
  concrete goal of this PoC is to confirm the pinned `copa-action` version
  actually patches BCI 16.0-derived images end-to-end. If it cannot, documented
  fallbacks are: a Copa zypper plugin, or a `zypper up` rebuild of just the base
  image as a stopgap.
- **Only fixable, OS-level CVEs are addressed** — by design.
- **Provenance (out of PoC scope):** patched Prime images should eventually be
  signed and have their SBOM refreshed the same way releases are.
- The security-sensitive third-party scanner/patcher actions (`copa-action`,
  `trivy-action`) are pinned by commit SHA. First-party `actions/*` and
  `docker/*` actions follow the existing repository convention of referencing
  major-version tags (as in `release.yml`).
