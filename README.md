# Job Tracker CI
 
Reusable GitHub Actions workflows shared by `job-tracker-api`,
`job-tracker-web` and `job-tracker-worker`, and the shared Renovate preset
(`default.json`) that every job-tracker repo extends.
 
| Workflow | Runs |
|---|---|
| `go.yml` | tidy check, `go vet`, golangci-lint, `go test -race`, build |
| `node.yml` | `npm ci`, turbo lint, typecheck, build |
| `python.yml` | `uv sync --frozen`, ruff, pytest |
| `image.yml` | Docker build, Trivy scan; on `main`: push to GHCR, cosign sign, SBOM attestation |
 
## Usage
 
```yaml
jobs:
  ci:
    uses: annakonn200059/job-tracker-ci/.github/workflows/go.yml@v1.0.0
    with:
      go-version: "1.24"
```
 
Inputs: `go-version` (1.24), `working-directory` (.) · `node-version` (24),
`turbo-filter` · `python-version` (3.12).
 
### image.yml
 
```yaml
jobs:
  image:
    permissions:
      contents: read
      packages: write
      id-token: write   # required for keyless cosign signing
    uses: annakonn200059/job-tracker-ci/.github/workflows/image.yml@v1.0.0
    with:
      image-name: job-tracker-api
```
 
Inputs: `image-name` (required, pushed as `ghcr.io/<owner>/<image-name>`),
`dockerfile` (Dockerfile), `context` (.), `trivyignores` (path, optional).
 
The caller job must grant all three permissions above — a reusable workflow
can't have more than its caller, and the run fails at startup otherwise.
 
Images are tagged with the short commit SHA. PRs and non-`main` branches only
build and scan; push, signing and the CycloneDX SBOM attestation run on pushes
to `main`. The build fails on fixable HIGH/CRITICAL CVEs.
 
Callers set their own `concurrency` — a reusable workflow can't cancel runs of
the workflow calling it.
 
## Releases
 
Callers reference a release tag (`@v1.0.0`), never `@main`, so a change here
reaches a service only through a Renovate PR in that repo, after its own CI
passes. To release, merge to `main` and push a semver tag:
 
```sh
git tag -a v1.1.0 -m "v1.1.0" && git push origin v1.1.0
```
 
Don't move or delete a published tag: callers and image signatures refer to it.
 
## Renovate
 
`default.json` is the shared preset. Each repo's `renovate.json` only extends it:
 
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>annakonn200059/job-tracker-ci"]
}
```
 
Repo-specific rules go in that file and override the preset. What the preset does:
 
- Runs before 6am on Mondays (Europe/Zurich); security fixes don't wait.
- Skips releases younger than 3 days (`minimumReleaseAge`).
- Groups minor/patch/digest updates into one PR and automerges it when CI
  passes. Majors, 0.x minors and Go/Node/Python minor upgrades get their own
  PR and need a review.
- Pins GitHub Actions and Docker images to digests, except these shared
  workflows, which stay on tags (see below).
- Weekly lockfile refresh; `go mod tidy` after Go updates.
- Picks up versions Renovate can't detect on its own from a comment on the
  line above:
 
  ```yaml
  # renovate: datasource=golang-version depName=go
  go-version: "1.26.7"
  ```
 
`renovate-config.yml` validates the preset on every PR. A broken preset stops
Renovate in every repo, so make it a required check.
 
Automerge only waits for checks GitHub knows about, so each repo needs a
branch protection rule (or ruleset) on `main` that requires its CI jobs.
 
## Verifying an image
 
`image.yml` signs with keyless cosign. The certificate names the workflow that
signed (this repo's `image.yml` at the tag the caller used) and the repo it
ran for:
 
```sh
IMAGE=ghcr.io/annakonn200059/job-tracker-api:<short-sha>
 
cosign verify "$IMAGE" \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity-regexp='^https://github\.com/annakonn200059/job-tracker-ci/\.github/workflows/image\.yml@refs/tags/v[0-9]+\.[0-9]+\.[0-9]+$' \
  --certificate-github-workflow-repository=annakonn200059/job-tracker-api \
  | jq .
```
 
Without `--certificate-github-workflow-repository`, any repo calling the
shared workflow would pass. Images built before the switch to tags were signed
as `image.yml@refs/heads/main`; verify those with that ref in the regexp.
 
The tag in the identity is why Renovate is told not to pin these workflows to
SHAs: a SHA-pinned call would be signed as `image.yml@<sha>`, which this
check rejects.
 
## Notes
 
`go test -race` always — slower, but the only reliable way to catch data races.
 
`go mod tidy` is verified rather than run: the job fails if the lockfile would
change, instead of silently fixing it.
 
Deployment lives elsewhere. This repo validates and publishes images.