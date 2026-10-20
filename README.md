# CI Early-Warn Beacon

Know when a **new GitHub-hosted runner image**, not your code, broke your build.

This action sends one anonymized health beacon at the end of a job. The [CI Early-Warn](https://github.com/kobima6813/ci-earlywarn) collector compares failure rates across many opt-in repositories on image version N vs N-1, and flags a version when independent repos start failing on it. That's often days before a public issue exists.

Live status: https://collector-production-5b85.up.railway.app/

## Usage
1. Get a token: `curl -s -X POST https://collector-production-5b85.up.railway.app/v1/signup`, then save `token` as the repo secret `EW_TOKEN`.
2. Add it as the **last step** of a job:
```yaml
permissions:
  contents: read
  actions: read     # read this run's job steps to find the failing step
  id-token: write   # optional: prove you're a real GitHub repo (OIDC); counts you immediately
steps:
  # ... your steps ...
  - if: always()
    uses: kobima6813/ci-earlywarn-action@v1
    with:
      token: ${{ secrets.EW_TOKEN }}
      job-status: ${{ job.status }}
```

## Inputs
| input | default | |
|---|---|---|
| `token` | — (required) | tenant token from `/v1/signup` |
| `job-status` | — (required) | pass `${{ job.status }}` |
| `endpoint` | public collector | your self-hosted collector URL |
| `github-token` | `${{ github.token }}` | needs `actions: read` only |
| `send-toolchain` | `true` | node/python/java/go major.minor |
| `oidc` | `true` | attach a GitHub OIDC token (aud `ci-earlywarn`) if `id-token: write` |
| `failures-only` | `false` | v1.1: send only when the job failed; passing runs send nothing |
| `report-actions` | `false` | v1.1: send salted hashes of this job's `uses:` actions, so the collector can tell a broken action from a broken image |

## What is sent (privacy)
**Default mode** (one beacon per job, pass or fail): `runner_os, image_os, image_version, conclusion, step_category` (one of install/build/test/lint/deploy/cache/docker/other/none), optional toolchain major.minor versions (`send-toolchain`), and a timestamp. The token identifies your tenant; with OIDC the collector keeps only a salted hash of the numeric owner id.

**`failures-only: true`**: the same fields, plus `"mode": "failures_only"`, sent **only when the job failed**. Nothing at all leaves the runner on passing runs. Trade-off: the collector has no pass baseline for you, so your tenant is **not** counted in failure-share math (the ≥50%-failing-tenants logic uses full-mode tenants only). Your failures are only extra corroboration toward the "how many independent tenants are failing" threshold, and they feed the action-vs-image diagnosis. In this mode you give the network less signal than in default mode.

**`report-actions: true`**: adds `actions`, up to 20 values, each the first 16 hex chars of `sha256("ci-earlywarn-actions-v1:" + owner/repo@ref)` for the `uses:` steps of the **current job**. The salt is shared by the whole network so the same action matches across tenants. Honest limits:
- The salt is public, so this hides names from casual readers but is **not anonymity**. Anyone can hash `actions/checkout@v4` and recognize it, and a private or unusual action could be confirmed by someone who already guesses its name. Leave it off if your action list is sensitive.
- The list is read from the workflow file in `GITHUB_WORKFLOW_REF`: the checked-out file if present, otherwise via the contents API (needs `contents: read`). The parser is a small indentation-based YAML reader. It skips local `./` actions and `docker://` refs. It does **not** expand reusable workflows, composite actions' inner `uses:`, matrix/expression refs, or YAML anchors. If it can't read the file, nothing is sent for `actions`.

**Never sent:** repo or org names, step names, logs, code, secrets, raw action names. The failing step name is categorized inside the runner. The OIDC token is verified by the collector and discarded. Raw beacons are deleted after 90 days.

## Action breakage vs image regression (v1.1)
When a version is flagged, the collector labels it:
- **likely image regression**: at least 50% of the failing tenants still pass on the previous image version during the same window.
- **likely action breakage**: the failing tenants share one action hash (60% or more of those reporting actions), and that action also fails for 2 or more tenants on *other* image versions in the same window.
- **unclear**: neither condition holds. That includes the case where too few tenants use `report-actions`.

The action has no dependencies, runs on Node 20 and **never fails your job**.

## License
MIT. Collector source, privacy and terms drafts: https://github.com/kobima6813/ci-earlywarn
