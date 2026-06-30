# ebc-hosting-actions

Shared CI for the EBC/GPS hosting platform. This repo is **public** so source repos
in **any** GitHub org can call its reusable workflow — a private repo's reusable
workflow is not callable cross-org, and that is the reason this lives on its own.

## `build-push-image.yml`

Builds a workload's container and pushes it to its Artifact Registry Image Repository
keylessly via Workload Identity Federation (no service-account keys). Call it from your
source repo once the platform has onboarded you:

```yaml
# .github/workflows/build-push.yml
on:
  push: { branches: [main] }
jobs:
  image:
    uses: elastic/ebc-hosting-actions/.github/workflows/build-push-image.yml@v1
    permissions: { contents: read, id-token: write }
    with:
      ar_repo: <tenant>-<owner>-<repo>   # your Image Repository id
      image: <image>                     # image name inside it
```

Optional inputs (with defaults): `context` (`.`), `dockerfile` (`Dockerfile`),
`region` (`us-west1`), `project` (`elastic-ce-tools`), `wif_provider`.

Pin `@v1` (a moving major tag) to get fixes automatically, or an immutable `@vX.Y.Z`
for reproducibility.

## Security

This workflow exposes identifiers (project, region, WIF provider name), not access:
push is authorised by a per-repo Artifact Registry writer binding and the GitHub-signed
`repository` OIDC claim, neither of which the workflow's contents can grant. The full
rationale is in the platform repo's
[ADR 0010](https://github.com/elastic/ebc-hosting-platform/blob/main/docs/adr/0010-reusable-build-workflow-in-public-repo.md).

Because consumers run this repo's code via `@v1`, it is part of their build's trust
boundary: protect `main` with required review and publish immutable version tags.
