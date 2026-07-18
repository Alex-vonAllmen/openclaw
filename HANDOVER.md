# Handover: testing upstream PR #108971 (exec-approvals lock fix)

Written by a Claude Code session on the dev machine, for a fresh Claude Code
session starting on the Mac Mini with direct access to the running stack.
This file is a trailing commit on `test-pr-108971` — not part of the PR diff
itself. Don't count it when checking "does this branch touch only the
intended two files" (see Scope section below for the real check).

## Goal

Validate upstream PR #108971 (fixes issue #106777: exec-approvals lock
strands on Docker/VirtioFS because `release` compared `fstatSync` dev/ino
against `lstatSync`, which diverge on VirtioFS, silently skipping the
unlink). The fix replaces inode/device lock ownership with a nonce in the
lock payload, plus a re-entrant depth guard.

Only a **subset** of the PR is wanted:

- WANTED: `src/infra/exec-approvals.ts` (the fix)
- WANTED: `src/infra/exec-approvals-sync-lock.test.ts` (new test)
- EXCLUDED: `src/config/sessions/store-load.ts` — unrelated session-pruning
  behavior change in the same upstream PR. Must never be applied here.

## Repo layout

- This repo (`~/Workdir/openclaw` on the dev machine) = fork of
  `openclaw/openclaw`. `origin` = `https://github.com/Alex-vonAllmen/openclaw`
  (your fork), `upstream` = `https://github.com/openclaw/openclaw.git`.
- Deploy target is a separate repo, `~/repos/personal-assistant`, which
  builds `openclaw-custom:local` via `Dockerfile.openclaw`
  (`FROM openclaw:local`, then layers on gog/whisper/chromium/etc). That
  wrapper is what actually runs as the `openclaw-gateway` container.
- **On the Mac Mini, find wherever the `openclaw` checkout lives** (likely
  `~/repos/openclaw` or similar, sibling to `personal-assistant`) — that's
  where `docker build -t openclaw:local .` gets run from. This dev-machine
  session never had access to the Mac Mini, so the exact path there is
  unconfirmed.

## What's been done (dev machine, this repo)

1. Original local `main` was stale — last synced with upstream on 2026-07-12,
   3359 commits behind true upstream tip (2026-07-18). This is why the first
   deploy attempt failed with:

   ```
   Config health-state write failed: OpenClaw state database ... uses newer
   schema version 3; this OpenClaw build supports 2.
   ```

   (the Mac Mini's `openclaw.sqlite` was already on a schema written by a
   much newer build than our stale `main`).

2. Fixed by merging `upstream/main` into local `main` on the dev machine.
   One conflict, in `extensions/openrouter/stream.ts` — purely a variable
   rename collision between the fork's cherry-picked
   "forward OpenClaw session id as session_id" feature and an unrelated
   upstream refactor (`OPENROUTER_THINKING_STREAM_HOOKS` module constant →
   `openRouterThinkingStreamHooks` local var via
   `buildProviderStreamFamilyHooks(...)`). Resolved by keeping the fork's
   session-id/routing logic and adopting upstream's renamed variable.
   Merge commit: `933982213f7c6689327c5496786137e6b026f30d`.

   **⚠️ This updated `main` was never pushed to `origin`.** It only exists
   as a local branch on the dev machine (original task guardrails said
   "never push main / never modify main" and that wasn't revisited). If the
   Mac Mini's `openclaw` checkout has its own `main` that's stale in the
   same way, `origin/main` on your fork is _also_ still stale — decide
   whether to push the updated main from the dev machine, or independently
   sync `main` to `upstream/main` on the Mac Mini's checkout. Either way,
   `main` needs to be at/past upstream commit `5fe8d6a852adb17f8557a1d860d635cf26333130`
   (2026-07-18) for `OPENCLAW_STATE_SCHEMA_VERSION` to be `4` (see
   `src/state/openclaw-state-db-contract.ts`), matching what's already on
   the Mac Mini's `openclaw.sqlite`.

3. Recreated `test-pr-108971` off the updated `main`. Brought in PR #108971
   via `git fetch upstream pull/108971/head:pr-108971` +
   `git merge --no-commit --no-ff pr-108971`, then
   `git checkout HEAD -- src/config/sessions/store-load.ts` to drop the
   excluded file. One merge conflict, in `src/infra/exec-approvals.ts`
   (main had drifted past the PR's base commit `e1fda44`) — resolved by
   keeping main's surrounding code and applying the PR's three logical
   changes verbatim:
   - `ExecApprovalsSyncLock` type carries `nonce: string` (+ `raw`) instead
     of `device`/`inode`; acquisition stores `nonce: crypto.randomUUID()`
     in the payload and no longer calls `fstatSync`.
   - `removeOwnedExecApprovalsLock` reads the current lock file and removes
     it when the payload's `nonce` matches (falls back to exact raw-string
     match when `requirePayloadMatch` is false); manages a process-global
     held-locks map.
   - Re-entrancy: `HELD_EXEC_APPROVALS_SYNC_LOCKS` (a `resolveGlobalMap`
     keyed by `Symbol.for("openclaw.heldExecApprovalsSyncLocks")`), keyed
     by `lockPath`, with depth counting — nested `acquire` increments depth
     and returns the already-held lock; `release` decrements, and only the
     outermost release closes the descriptor and unlinks. Also exports
     `resetExecApprovalsSyncLockStateForTest()`.

   Commit: `2bd3cc046c0a9329320ea8f2fcfe09eef036ef9a`
   "test: PR #108971 nonce-based exec-approvals lock (store-load hunk excluded)"

   **Scope check** (rerun this on any machine before trusting the branch):

   ```bash
   git diff --stat main       # must show ONLY exec-approvals.ts + exec-approvals-sync-lock.test.ts
   git diff main -- src/config/   # must be empty
   ```

4. Tests, run in Docker (`node:24`, `pnpm@9.15.0` via corepack, `CI=true`
   to avoid an interactive `node_modules` purge prompt when a host-side
   `node_modules` is bind-mounted in):

   ```bash
   docker run --rm -e CI=true -v "$PWD":/repo -w /repo node:24 bash -lc \
     "corepack enable && corepack prepare pnpm@9.15.0 --activate && pnpm install && \
      pnpm test src/infra/exec-approvals-sync-lock.test.ts src/infra/exec-approvals-store.test.ts src/plugin-sdk/file-lock.test.ts"
   ```

   All passed: 84/84 (73 in the two exec-approvals files + 11 in
   `file-lock.test.ts`) after the `main` rebase.

5. `docker build -t openclaw:local .` succeeds on the dev machine
   (this is just a sanity build on this machine — it does **not** prove
   anything about what's built/running on the Mac Mini).

6. Force-pushed the rebased branch: `git push --force-with-lease -u origin
test-pr-108971`.

## The open problem — THIS IS WHERE YOU COME IN

After redeploying on the Mac Mini (`git fetch && checkout test-pr-108971`,
`docker build -t openclaw:local .`, `docker compose build openclaw-gateway`,
`docker compose up -d openclaw-gateway`, `docker compose stop lock-reaper`),
the schema-mismatch error is gone, but **the exec-approvals fix does not
appear to be present in the running container**:

```bash
docker exec openclaw-gateway sh -c 'grep -c "HELD_EXEC_APPROVALS_SYNC_LOCKS" /app/openclaw.mjs'
# → 0

docker exec openclaw-gateway sh -c 'grep -rl "nonce" /app 2>/dev/null'
# → only matches in /app/docs and /app/qa, nothing in code
```

And the lock file keeps getting created in the **old, pre-fix payload
shape** (no `nonce` field) and then getting stranded — i.e. issue #106777
reproducing live, on code that shouldn't have the bug anymore:

```json
{ "pid": 22, "createdAt": "2026-07-18T21:49:38.527Z", "starttime": 9592443 }
```

(compare to what the fix should produce: this object plus a `nonce` field)

One round of testing (4 sequential exec calls through the chat UI) looked
clean — no "file lock timeout" surfaced to the user — but immediately
after, direct inspection showed the lock stuck in the old format again.
**Treat that clean run as inconclusive, not as evidence the fix works** —
it's more likely the old racy code just didn't get unlucky that particular
time.

### What's been ruled out

- Not a `.dockerignore` exclusion — `src/infra` isn't excluded (checked).
- Not a remote git-clone-inside-Dockerfile issue — the `Dockerfile` uses
  `COPY . .` (line 121) from the local build context, no `git clone`. So
  whatever's in the local checkout when `docker build` runs is what should
  end up in the image.
- The runtime bundle lives in `dist/` (built by `pnpm build:docker` +
  `pnpm ui:build` in the `build` stage, then copied into the final image),
  not directly in `openclaw.mjs` — the earlier `grep ... /app/openclaw.mjs`
  check was too narrow. **Never got to check `/app/dist` directly before
  this handover** — that's the next concrete step.

### What was never confirmed (do this first)

```bash
# On the Mac Mini, in the openclaw checkout used for `docker build`:
cd <path-to-openclaw-checkout-on-mac-mini>
git fetch origin
git log -1 --format='%H %s'
# expect: 2bd3cc046c0a9329320ea8f2fcfe09eef036ef9a  test: PR #108971 nonce-based exec-approvals lock (store-load hunk excluded)
git status -sb
grep -n "nonce" src/infra/exec-approvals.ts | head -5
# expect multiple hits, including `nonce: crypto.randomUUID()`
```

If HEAD isn't `2bd3cc046c0...`, that alone explains everything — the branch
was rebased and force-pushed _after_ the Mac Mini's last `docker build`, and
it's simply running stale local commits. Fix with
`git checkout test-pr-108971 && git reset --hard origin/test-pr-108971`,
then rebuild.

If HEAD is correct and the source has the fix, but the running image still
doesn't:

```bash
# force a clean rebuild, no cached layers, to rule out stale layer caching
docker build --no-cache -t openclaw:local .
cd ~/repos/personal-assistant
docker compose build --no-cache openclaw-gateway
docker compose up -d openclaw-gateway

# then check dist/ directly (not openclaw.mjs) inside the container
docker exec openclaw-gateway sh -c 'grep -rl "nonce" /app/dist 2>/dev/null'
docker exec openclaw-gateway sh -c 'grep -rc "HELD_EXEC_APPROVALS_SYNC_LOCKS" /app/dist 2>/dev/null'
```

## Once the fix is confirmed present in the running container

Re-test with multiple exec agent turns and confirm:

- No `file lock timeout` errors.
- No `security=deny` on commands that _are_ on the allowlist (a
  `security=deny` on out-of-workspace paths like `/vault/_raw/...` is
  expected/correct policy behavior, unrelated to this fix — seen already
  and is fine).
- `docker exec openclaw-gateway ls /home/node/.openclaw/exec-approvals.json.lock`
  returns "No such file or directory" between execs (not just after one
  lucky run — check it a few times across different calls).
- If a lock file is ever observed, `cat` it and confirm the payload
  includes a `nonce` field (old format = still broken).

## Rollback (if you decide to abandon this test)

```bash
git checkout main   # ⚠️ see the caveat above — decide first whether this
                     # machine's `main` is the updated one or still stale
docker build -t openclaw:local .
# redeploy via personal-assistant compose as usual
docker compose up -d lock-reaper
```

## Housekeeping / do-not-touch

- Two untracked files sit in this repo's working tree on the dev machine:
  `ADR-0001-openrouter-session-forwarding.md`,
  `PRD-openrouter-session-forwarding.md`. User said: ignore them, do not
  commit them. They're local-machine-only (untracked), so they won't be
  present on the Mac Mini's checkout — mentioned here only for
  completeness in case they show up via some other sync path.
- Never push `main`, only ever push `test-pr-108971`, until/unless the
  user decides otherwise (see the `main` caveat above — that decision is
  now genuinely open, not just a standing rule).
