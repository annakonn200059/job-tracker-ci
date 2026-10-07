# Job Tracker CI
 
Reusable GitHub Actions workflows shared by `job-tracker-api`,
`job-tracker-web` and `job-tracker-worker`.
 
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
    uses: annakonn200059/job-tracker-ci/.github/workflows/go.yml@main
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
    uses: annakonn200059/job-tracker-ci/.github/workflows/image.yml@main
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
 
## Notes
 
Pinned to `@main`, so changes here reach all three repos immediately. Tag
releases if that stops being acceptable.
 
`go test -race` always — slower, but the only reliable way to catch data races.
 
`go mod tidy` is verified rather than run: the job fails if the lockfile would
change, instead of silently fixing it.
 
Deployment lives elsewhere. This repo validates and publishes images.