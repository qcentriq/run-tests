# QCentriQ Test Run — GitHub Action

Run your QCentriQ test suites from GitHub Actions. The action triggers a run,
waits for the verdict, and fails the step when tests fail — so a failing suite
blocks the merge.

```yaml
- uses: qcentriq/run-tests@v1
  with:
    api-url: ${{ vars.QC_API_URL }}
    api-token: ${{ secrets.QC_API_TOKEN }}
    project-id: ${{ vars.QC_PROJECT_ID }}
    test-group-id: ${{ vars.QC_TEST_GROUP_ID }}
```

No `actions/checkout` is needed. QCentriQ never reads your repository — your
test definitions live in QCentriQ and execute on its runners, so this job only
triggers a run and polls for the result. Nothing is installed or downloaded:
the action uses `curl` and `jq`, both preinstalled on GitHub-hosted runners.

---

## 1. Create an API key

A project admin creates a project-scoped key. **The plaintext key is returned
only once** — copy it immediately.

Either use **Project Settings → API Keys** in the QCentriQ UI, or:

```bash
curl -X POST https://<qcentriq>/api/v1/api-keys/ \
  -H "Authorization: Bearer <your-jwt>" \
  -H "Content-Type: application/json" \
  -d '{"name": "github-actions", "project_id": "<project-uuid>", "scopes": ["ci"]}'
```

The response includes `api_key` (save it) and `key_prefix` (safe to record for
identification). Keys are revocable at any time and each is scoped to exactly
one project — create separate keys for separate projects.

## 2. Find your IDs

One request returns everything you need:

```bash
curl -s https://<qcentriq>/api/v1/integration/info \
  -H "X-API-Key: <api-key>" | jq
```

It returns the project (`project.id`) plus every test group with its cases, so
you can pick a `test_groups[].id` for `test-group-id` or a
`test_groups[].test_cases[].id` for `test-case-id`.

## 3. Configure the repository

**Settings → Secrets and variables → Actions**

| Name | Kind | Value |
|---|---|---|
| `QC_API_TOKEN` | Secret | the API key from step 1 |
| `QC_API_URL` | Variable | your QCentriQ base URL |
| `QC_PROJECT_ID` | Variable | project UUID |
| `QC_TEST_GROUP_ID` | Variable | test group UUID |

The token must be a **secret**, not a variable — secrets are encrypted and
masked in logs.

## 4. Gate merges on the result

Mark the job a required status check: **Settings → Branches → branch protection
rule → Require status checks to pass**, then select your job (e.g.
`QCentriQ tests`). GitHub then refuses to merge while the suite fails.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `api-url` | yes | — | QCentriQ base URL |
| `api-token` | yes | — | Project-scoped API key; pass via `secrets` |
| `project-id` | yes | — | Project UUID |
| `test-group-id` | one of | `''` | Test group to run |
| `test-case-id` | one of | `''` | Single test case to run |
| `environment-id` | no | project default | QCentriQ environment to run against |
| `browser-id` | no | project default | Browser to run on |
| `env-vars` | no | `''` | `KEY=VALUE` per line, overriding environment values |
| `poll-interval` | no | `5` | Seconds between status polls |
| `timeout` | no | `3600` | Max seconds to wait for a terminal status |
| `fail-on-test-failure` | no | `true` | Set `false` to report without gating |
| `upload-log` | no | `true` | Upload `run.log` as an artifact |
| `comment-on-pr` | no | `true` | Post a sticky comment on the PR |

Supply exactly one of `test-group-id` or `test-case-id`.

## Outputs

| Output | Description |
|---|---|
| `run-id` | UUID of the QCentriQ run |
| `status` | `passed`, `failed` or `error` |
| `total` | Test cases executed |
| `passed` | Test cases that passed |
| `failed` | Test cases that failed |

```yaml
- uses: qcentriq/run-tests@v1
  id: qa
  with: { ... }
- run: echo "${{ steps.qa.outputs.passed }}/${{ steps.qa.outputs.total }} passed"
```

## Exit codes

| Code | Meaning |
|---|---|
| 0 | All tests passed |
| 1 | One or more tests failed |
| 2 | Run error — infrastructure or execution failure |
| 3 | Action error — bad input, auth failure, unreachable API, or timeout |

## Environment overrides

Point the suite at a different target without touching QCentriQ config:

```yaml
with:
  env-vars: |
    BASE_URL=https://staging.example.com
    API_KEY=${{ secrets.STAGING_API_KEY }}
```

- One `KEY=VALUE` per line; blank lines ignored.
- Only the **first** `=` separates key from value, so base64 and JWT values
  containing `=` survive intact.
- CI values win over the QCentriQ-configured environment on collision. The
  selected environment still supplies the browser, secrets and base config.
- Server limits: at most **50 keys** and **4096 bytes** serialized.

### Testing a deployment's URL

Any tool that reports a GitHub deployment — Vercel, Netlify, your own deploy
job — exposes its URL on the `deployment_status` event, so preview
environments need no extra plumbing:

```yaml
on: [deployment_status]

jobs:
  qa:
    if: github.event.deployment_status.state == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: qcentriq/run-tests@v1
        with:
          api-url: ${{ vars.QC_API_URL }}
          api-token: ${{ secrets.QC_API_TOKEN }}
          project-id: ${{ vars.QC_PROJECT_ID }}
          test-group-id: ${{ vars.QC_TEST_GROUP_ID }}
          env-vars: BASE_URL=${{ github.event.deployment_status.target_url }}
```

## What you get in the PR

- **Annotations** — one inline error per failed test case, with its message.
- **Job summary** — a pass/fail table plus a collapsible list of failures.
- **A sticky comment** — updated in place on each push rather than stacking.
  Requires `permissions: pull-requests: write`.
- **`run.log`** — full per-case logs, uploaded as an artifact even on failure.

## Troubleshooting

**`Trigger failed (HTTP 401)`**
The API key is invalid, revoked, expired, or belongs to a different project.
Regenerate it from Project Settings → API Keys.

**`Trigger failed (HTTP 422)`**
Usually a malformed UUID, or both `test-group-id` and `test-case-id` supplied.

**`Set exactly one of test-group-id or test-case-id`**
Neither was provided, or both were. Supply exactly one.

**`Could not reach <url>`**
Wrong `api-url`, or your QCentriQ instance is not reachable from GitHub's
runners. Self-hosted instances behind a VPN need a self-hosted runner.

**`Timed out after 3600s`**
Raise `timeout`, and keep the job's own `timeout-minutes` above it.

**Job failed but no annotations**
The run ended `error` rather than `failed` — an infrastructure problem rather
than a test assertion. Check `run.log` in the workflow artifacts.

**No PR comment**
The job needs `permissions: pull-requests: write`. The action warns rather than
failing when it cannot comment.

## Security

- Always pass `api-token` from `secrets`, never a variable or a literal. The
  action masks it in logs and passes it via the environment, never on a command
  line where `ps` could expose it on a shared runner.
- Each key is scoped to one project and is revocable.
- Test values sent through `env-vars` appear in the API request. Pass anything
  sensitive from `secrets` so GitHub masks it in logs too.
