# ebc-hosting-actions

Reusable GitHub Actions workflow to build a container image and push it to Google
Artifact Registry using keyless Workload Identity Federation.

## `build-push-image.yml`

```yaml
# .github/workflows/build-push.yml
on:
  push: { branches: [main] }
  workflow_dispatch:
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

`workflow_dispatch:` gives the workflow a **Run workflow** button, so you can build without
a new commit: a path-filtered monorepo workflow otherwise has no event that re-triggers it,
and the first build has to be re-run after the platform PR that onboards the workload merges.

Required inputs: `ar_repo`, `image`, `project`, `wif_provider`. Optional inputs (with
defaults): `region` (`us-west1`), `context` (`.`), `dockerfile` (`Dockerfile`),
`wait_for_repository_minutes` (`10`, see below).

Outputs, for a later job in the calling workflow (`needs.image.outputs.<name>`):

| Output | Value |
|---|---|
| `digest` | The digest the registry serves for the pushed image, `sha256:...`. Empty if the push succeeded but the digest could not be read back; the run then says so with a warning. |
| `image` | The image path without tag or digest, `<region>-docker.pkg.dev/<project>/<ar_repo>/<image>`. |

A successful run's job summary shows the image, the digest, the tags it pushed (and why
`:latest` was or was not one of them), and a ready-to-paste kustomization `images:` entry
that pins the digest.

## Tags: `:latest` follows the default branch only

Every build pushes `<image>:<commit-sha>`. `<image>:latest` is pushed **only** when the
build runs on the calling repository's default branch, that is when `github.ref` is
`refs/heads/<default branch>`. A workload that follows the digest behind `:latest`
(`track: digest` on the hosting platform) therefore follows the default branch, and
nothing else.

| The calling run was started by | `:<sha>` | `:latest` |
|---|---|---|
| A push to the default branch, or **Run workflow** (`workflow_dispatch`) on it | yes | yes |
| A push to, or **Run workflow** on, any other branch | yes | no |
| `pull_request`, a tag push, a merge queue run | yes | no |
| An event whose payload does not carry the repository's default branch (for example `schedule`) | yes | no |

The default branch is read from the caller's event payload
(`github.event.repository.default_branch`). `push`, `workflow_dispatch` and
`pull_request` payloads carry it; where an event's payload does not, the build is still
pushed as `:<sha>`, `:latest` is left where it was, and the run says so with a notice. To
move `:latest`, run the build from the default branch.

A build that did not move `:latest` is not rolled out to a tracked workload, and pinning
its digest by hand does not hold: Image Updater writes the digest behind `:latest` back on
its next check. An untracked workload can be pinned to any `:<sha>` build's digest.

Before v1.3.0 every build moved `:latest`, whatever branch it ran on.

## Before the platform PR merges: `wait_for_repository_minutes`

The Image Repository and your repo's permission to push to it are created by the hosting
platform only after the PR that onboards your workload merges. Before that, a push is
refused with `denied: Permission 'artifactregistry.repositories.uploadArtifacts' denied
... (or it may not exist)`.

So before it builds, the workflow checks that it can reach the Image Repository
(`gcloud artifacts repositories describe`), and retries every 30 seconds for up to
`wait_for_repository_minutes` (default `10`). This covers a run started straight after the
merge, while the repository and its permission are still being created. If the repository
is still unreachable when the time runs out, the run fails **before building**, and its
summary gives the command to re-run it once the platform PR has merged
(`gh run rerun <run-id>`, **Re-run jobs**, or **Run workflow** if you added
`workflow_dispatch:`). Set `wait_for_repository_minutes: 0` to skip the check.

Pin `@v1` (a moving major tag) to get fixes automatically, or an immutable `@vX.Y.Z`
for reproducibility.

## When a push fails, the workload goes stale

A failed build produces no image. If the workload already runs an earlier image, **the
platform keeps serving it**; on a first build there is no image yet, and the workload
cannot start until one is published. Nothing about that is visible from the outside: your
`main` is green, every required check passed, and the running workload silently predates
your commit, or does not exist. A green `main` does not mean the deployed workload
contains it.

Two things guard against this:

- **The flaky steps retry.** `gcloud auth login` (the Workload Identity Federation
  STS exchange) is retried five times with backoff, and `docker buildx build --push`
  three times. The STS exchange in particular fails intermittently with
  `Unable to retrieve Identity Pool subject token` / `reset reason: overflow`, a
  transient fault, not a misconfiguration.
- **A failed run says so.** The run emits an error annotation and a job summary
  naming the commit whose image is missing, and how to recover (re-run the job).

### Getting notified

This workflow **cannot** open an Issue on your behalf. A called workflow's
`GITHUB_TOKEN` permissions [can only be maintained or reduced, not elevated][perms]
relative to the caller's, and callers grant only `contents: read` and
`id-token: write`. If you want an Issue, add a notify job **in your own repo**,
alongside the job that calls this workflow:

```yaml
jobs:
  image:
    uses: elastic/ebc-hosting-actions/.github/workflows/build-push-image.yml@v1
    permissions: { contents: read, id-token: write }
    with: { ar_repo: ..., image: ..., project: ..., wif_provider: ... }

  notify-stale:
    needs: image
    if: failure() && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions: { contents: read, issues: write }
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const sha = context.sha.slice(0, 8);
            const runUrl = `${context.serverUrl}/${context.repo.owner}/` +
              `${context.repo.repo}/actions/runs/${context.runId}`;
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Image not published for ${sha}: the demo is stale`,
              body: `No image was pushed for \`${sha}\`, so the running workload ` +
                    `does not contain it.\n\nRun: ${runUrl}`,
            });
```

Keep untrusted commit text (`github.event.head_commit.message`) out of `run:` blocks
and `${{ }}` expressions (pass it through `env:`), or you have a [script injection][inj].

[perms]: https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows
[inj]: https://securitylab.github.com/resources/github-actions-untrusted-input/

## Security

Authentication is keyless via Workload Identity Federation: there are no secrets in
this repository. The workflow can push only where the calling repository's federated
identity has already been granted `artifactregistry.writer`; the workflow's contents
grant no access on their own.
