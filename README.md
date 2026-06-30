# ebc-hosting-actions

Reusable GitHub Actions workflow to build a container image and push it to Google
Artifact Registry using keyless Workload Identity Federation.

## `build-push-image.yml`

```yaml
# .github/workflows/build-push.yml
on:
  push: { branches: [main] }
jobs:
  image:
    uses: elastic/ebc-hosting-actions/.github/workflows/build-push-image.yml@v1
    permissions: { contents: read, id-token: write }
    with:
      ar_repo: <artifact-registry-repo>
      image: <image-name>
      project: <gcp-project>
      wif_provider: <workload-identity-provider-resource-name>
```

Required inputs: `ar_repo`, `image`, `project`, `wif_provider`. Optional inputs (with
defaults): `region` (`us-west1`), `context` (`.`), `dockerfile` (`Dockerfile`).

Pin `@v1` (a moving major tag) to get fixes automatically, or an immutable `@vX.Y.Z`
for reproducibility.

## Security

Authentication is keyless via Workload Identity Federation — there are no secrets in
this repository. The workflow can push only where the calling repository's federated
identity has already been granted `artifactregistry.writer`; the workflow's contents
grant no access on their own.
