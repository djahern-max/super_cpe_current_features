# Feature 014a — Make the versioning control deployable

## Why this feature exists

013 shipped a `prod` boot refusal (`ensure_bucket_versioning`) whose only
remedy was `deploy/bucket-setup.py` — and on 2026-08-30 that script proved
unrunnable from the environment the runbook assumes:

1. The droplet host has no `python` and no boto3; the script was written as
   if it runs on the host.
2. It is not in the api image either — the Dockerfile copies only
   `backend/`, so `docker compose run api python deploy/bucket-setup.py`
   cannot see it.
3. Bind-mounting it to `/tmp` broke its `sys.path` bootstrap
   (`ModuleNotFoundError: No module named 'app'`), because the path insert
   is computed relative to `__file__` for the repo layout.
4. When the operator finally reached the S3 API by hand, the key produced
   `AccessDenied` on `PutBucketVersioning` — the runbook says "temporary
   Full Access key" but never says that a **bucket-scoped** key, even with
   full object permissions, cannot call bucket-configuration operations.
   Only an **All Permissions** (all-buckets) key can.

Meanwhile `ad00797` had already been deployed, so the boot refusal took
production down: containers up, uvicorn workers dead, Caddy returning 502.
`deploy.sh` fails safe on a bad migration but has no equivalent gate for a
boot refusal — the old version was already stopped by the time the guard
fired.

014a fixes the operability, not the control. The guard stays. Versioning is
the bucket-layer 9.02 control 013 was built for: with it on, an accidental
overwrite or delete of a certificate or audit bundle is recoverable by
`VersionId` instead of merely being forbidden by application discipline.
The refusal-to-boot design (a control someone can switch off in a panel
must not be silently absent) also stays. What changes is that a failed
guard must become a **failed deploy with the old version still serving**,
never an outage — and the setup step must be runnable from the documented
environment with no mounts, no host Python, and no path tricks.

## Scope

### 1. `python -m app.cli bucket-setup`

Move the logic of `deploy/bucket-setup.py` into an `app.cli` subcommand and
delete the standalone script (git history retains it; the changelog entry
records the move).

Behavior is identical to 013's spec:

- Reads the Spaces endpoint, region, and bucket from the normal settings
  (the container's `env_file` provides them).
- Reads credentials **only** from `SETUP_SPACES_KEY` / `SETUP_SPACES_SECRET`
  environment variables — never from `.env`, never from the runtime key
  settings. Exits non-zero with a clear message if either is unset.
- Enables object versioning; puts the lifecycle configuration with exactly
  one rule (expire noncurrent versions under `BACKUPS_PREFIX` after
  `BACKUP_NONCURRENT_DAYS`).
- Prints both read-backs; exits non-zero if either does not read back as
  set. Idempotent.
- New: on `AccessDenied`, the error message must say in plain words that a
  bucket-scoped key cannot perform bucket-configuration calls and that an
  All Permissions Spaces key is required. This exact failure cost real
  debugging time; the tool should name it.

Because it runs inside the api image, boto3 and the `app` package are
simply present. The documented invocation becomes one line with no mounts:

```bash
cd /srv/supercpe/repo && export GIT_SHA=$(git rev-parse HEAD)
docker compose -f deploy/docker-compose.yml run --rm \
  -e SETUP_SPACES_KEY -e SETUP_SPACES_SECRET \
  api python -m app.cli bucket-setup
```

Port 013's moto tests for the script (idempotency, read-back failure) to
the CLI subcommand. Add one test asserting the unset-credentials refusal
and one asserting the AccessDenied message names the key-scope cause.

### 2. `python -m app.cli preflight` and the deploy gate

A new CLI subcommand that runs, without starting the server, exactly the
checks that would refuse boot in `prod`:

- the 012 production-config validations (listing every violation at once,
  same code path — do not duplicate the rules), and
- `ensure_bucket_versioning` when `STORAGE_BACKEND=spaces`.

Exit 0 when the app would boot; non-zero with the same messages the boot
refusal would print.

`deploy.sh` gains a step: after building the new images and **before**
running migrations or touching the running containers, run `preflight` as a
one-off container from the newly built image against production's
`env_file`. Non-zero aborts the deploy; the running API is never stopped.
This is the same fail-safe shape the migration step already has, extended
to cover boot refusals. `rollback.sh` gets the identical gate (a rollback
target that cannot boot should also fail before touching anything).

Ordering note: preflight runs before migrations on purpose. It validates
config and bucket state, which do not depend on schema; putting it first
means a guard failure aborts before the schema has moved, leaving nothing
to reconcile.

Tests: preflight exit codes under (a) clean prod config with versioning
enabled (moto), (b) versioning suspended/absent → non-zero naming
`bucket_versioning`, (c) a config violation → non-zero listing it. And one
test asserting `deploy.sh` contains the preflight invocation before the
migration step (a grep-level assertion is fine; the script itself is not
unit-testable).

### 3. `docs/OPERATIONS.md` — rewrite "Bucket versioning" setup

- The one-line invocation above, verbatim, with the `GIT_SHA` export
  prefix per the standing runbook rule.
- Key requirements stated bluntly: DigitalOcean console → Spaces Object
  Storage → Access Keys → Create Access Key → **All Permissions** (do not
  scope to the bucket — a bucket-scoped key gets `AccessDenied` on
  `PutBucketVersioning` even with full object rights). Use `read -rs` to
  keep the secret out of shell history. Delete the key in the console
  immediately after the read-backs print; `unset` the variables.
- A dated record line for the real run's two read-backs (this is the
  evidence for whether DigitalOcean honors `NoncurrentVersionExpiration`
  with a prefix filter — 013's open question).
- The recovery-drill record line stays where 013 put it.
- A new paragraph under the deploy procedure: what a preflight abort looks
  like and that it means the old version is still serving — read the
  listed violations, fix, redeploy; do not treat it as an outage.

### 4. Changelog

Per CLAUDE.md, append-only — two entries when this ships:

1. A correction/incident entry for 2026-08-30: `ad00797` was deployed
   before `bucket-setup.py` had run (013's own known-gaps note said it
   must not be); the boot refusal fired in production; the site served
   502 for the duration; the script proved unrunnable as documented for
   the four reasons above; recovery path taken. Plain statement of fact,
   no editing of the 013 entry.
2. The normal 014a entry (comes back after the build, as usual).

`COMPLIANCE.md`: no locator changes expected; if the 9.02 row for 013
references the script by path, add the correction row pointing at the CLI
subcommand.

## Explicitly out of scope

- Removing or weakening `ensure_bucket_versioning`. Considered and
  rejected: the alternatives are `ENV=dev` in production (disables every
  guard — cookies, CORS, sslmode — to dodge one) or deleting the control
  013 exists to enforce.
- The off-site mirror (still dormant by the 2026-08-30 operator decision).
- 014 proper (ASC842-PCX re-ingest and real review) — unblocked by this,
  not part of it.

## Acceptance (on the droplet, in order)

0. **Stabilize first if still down.** If https://supercpe.com is serving
   502, `./deploy/rollback.sh 62de030` before anything else — with the
   note that if `ad00797`'s migrations added schema, `62de030` must be
   confirmed to run against it (rollback never downgrades migrations).
   Record the outage window for the changelog entry.
1. Build 014a, tag, and run `bucket-setup` via the one-liner with a fresh
   All Permissions key. Both read-backs print; copy them, dated, into
   OPERATIONS.md. Delete the key in the console; `unset` the variables.
2. Recovery drill: write `health/sentinel` twice, list its versions,
   recover the older by `VersionId`, record the dated result in
   OPERATIONS.md (013's empty line).
3. Deploy the 014a sha via `deploy.sh`. Observe preflight pass in the
   output. `/health` reports the new sha, `bucket_versioning: ok`, all
   components ok.
4. Negative test of the gate, from the repo checkout: run `preflight` as a
   one-off container with `STORAGE_BACKEND` overridden to a bucket that
   does not exist (or versioning check otherwise failing) and confirm
   non-zero with the violation named — proving the gate would have caught
   today's failure before the restart.
5. Full test suite passes locally (report the count).
