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

## What is sent
`runner_os, image_os, image_version, conclusion, step_category` (one of install/build/test/lint/deploy/cache/docker/other/none), optional toolchain versions, and a timestamp. **Not sent:** repo or org names, step names, logs, code, secrets. The failing step name is categorized inside the runner. The OIDC token is verified by the collector and discarded; only a salted hash of the numeric owner id is kept. Raw beacons are deleted after 90 days.

The action has no dependencies, runs on Node 20 and **never fails your job**.

## License
MIT. Collector source, privacy and terms drafts: https://github.com/kobima6813/ci-earlywarn
