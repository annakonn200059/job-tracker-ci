# Job Tracker CI
 
Reusable GitHub Actions workflows shared by `job-tracker-api`,
`job-tracker-web` and `job-tracker-worker`.
 
| Workflow | Runs |
|---|---|
| `go.yml` | tidy check, `go vet`, golangci-lint, `go test -race`, build |
| `node.yml` | `npm ci`, turbo lint, typecheck, build |
| `python.yml` | `uv sync --frozen`, ruff, pytest |
 
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
 
Callers set their own `concurrency` — a reusable workflow can't cancel runs of
the workflow calling it.
 
## Notes
 
Pinned to `@main`, so changes here reach all three repos immediately. Tag
releases if that stops being acceptable.
 
`go test -race` always — slower, but the only reliable way to catch data races.
 
`go mod tidy` is verified rather than run: the job fails if the lockfile would
change, instead of silently fixing it.
 
Image building and deployment live elsewhere. This repo only validates.