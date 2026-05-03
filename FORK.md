# OpenClaw Fork — Synthcore-private modifications

This is a **private fork** of [openclaw/openclaw](https://github.com/openclaw/openclaw) maintained for the HumanikOS / Synthcore platform. Upstream is the public OSS repo; this fork carries platform-specific changes that we do not (yet) intend to upstream.

This file is the source of truth for **what we changed, why, and how to merge upstream** without losing those changes. If you are about to pull from upstream, read the [Merge Protocol](#merge-protocol) section first.

---

## Why we fork (vs. extend via plugins / SDK)

OpenClaw exposes a generous public surface — `plugin-sdk`, runtime-helper deps, manifest metadata — and most platform features should live in plugins or in the wrapping `hos-openClaw` host process. **We only fork the OSS engine when the integration point we need does not exist in the public surface and the upstream patch cost is too high to land first.**

Each fork-only change in this file should explain (a) why a plugin/SDK extension wouldn't have worked, and (b) whether we plan to upstream the change.

---

## Fork additions (current as of 2026-05-03)

### 1. Cross-VM cron slot lease — `beforeFireSlot` deps hook

**Files touched:**

- `src/cron/schedule.ts` — added `computeJobSlotMs(schedule, nowMs)` helper
- `src/cron/service/state.ts` — added optional `beforeFireSlot` to `CronServiceDeps`
- `src/cron/service/timer.ts` — wrapped `runDueJob` and `runStartupCatchupCandidate` to consult the hook before firing; on `proceed: false` returns `status: 'skipped'`
- `src/gateway/server-cron.ts` — `buildHosSlotLeaseHook()` reads `HOS_INTERNAL_TOKEN` + `HOS_SERVER_PORT` from env; if both set, wires `beforeFireSlot` to call the parent's `/internal/tools/cron/lease/claim` endpoint

**Why it exists:**
HumanikOS deploys OpenClaw as a Fly.io machine per office. Under certain conditions (cold-start spawn race, manual scaling, retried Fly machine create), two VMs can come up for the same office. Each VM's `runMissedJobs` and `runDueJob` paths fire the same recurring crons independently — there is no in-process coordination point that spans VMs. The `markCronJobActive` Set and `runningAtMs` markers are per-process, so they don't dedupe across VMs.

The hook lets the wrapping host (`hos-openClaw`) coordinate via Firestore, keyed on a deterministic slot identifier. Two VMs evaluating the same recurring cron at the same wall-clock interval compute the same `slotMs`, run a Firestore transaction on `cronJobs/{nexusCronId}`, and only one wins.

**Why not a plugin / SDK extension:**
The fire-time decision happens **inside** `runDueJob` and `runStartupCatchupCandidate`, before `executeJobCore` is called. There is no SDK seam that intercepts at this layer — the closest are `runIsolatedAgentJob` and `enqueueSystemEvent`, which are called _after_ the engine has already committed to firing the slot. Throwing from those would mark the run as an error (with `consecutiveErrors`, `lastError`, failure-alert side effects), which is semantically wrong for "another instance handled this slot."

**Behavior when the hook is not configured:**
Fully back-compat. If `HOS_INTERNAL_TOKEN`/`HOS_SERVER_PORT` are absent, `buildHosSlotLeaseHook()` returns `undefined`, the deps interface stays at its existing shape, and the engine fires every slot exactly as it does upstream.

**Failure modes:**

- Lease endpoint unreachable / Firestore error → engine fails open and fires (logged).
- Lease grants two VMs for the same slot due to write conflict → upstream bug in our Firestore txn, not the engine.
- Lease holder crashes mid-fire → TTL (30 min, configurable in `claimSlot.ts`) lets next attempt re-claim.

**Upstream plan:**
Open question. The `beforeFireSlot` deps shape is generic enough to upstream as a "distributed cron coordination" extension point. Pending: write up an RFC for the OpenClaw repo proposing the hook + a reference Redis-backed implementation. For now, kept private.

**Related changes outside this fork (for context):**

- `hos-openClaw/src/crons/lease/claimSlot.ts` — Firestore transaction implementation
- `hos-openClaw/src/hosClawGateway/tools/controllers/cronLeaseToolController.ts` — HTTP endpoint
- `nexus/nexusApi/src/crons/services/types.ts` — `CronJobDoc` gains `runningInstanceId / runningStartedAtMs / runningSlotMs` fields

---

## Merge Protocol

### When pulling upstream

1. **Read this file first.** Each section under "Fork additions" lists exactly which files we touched and why.
2. **Fetch upstream into a scratch branch:**
   ```sh
   cd hos-openClaw/openclaw
   git remote add upstream https://github.com/openclaw/openclaw.git   # one-time
   git fetch upstream main
   git checkout -b merge/upstream-<date> main
   git merge upstream/main
   ```
3. **Conflicts to expect** — the touched files in each fork addition are conflict candidates. For the cron slot lease specifically:
   - `src/cron/schedule.ts` — additive (new exported function); conflicts only if upstream renames or restructures the file
   - `src/cron/service/state.ts` — additive (new optional dep); conflicts only if upstream changes `CronServiceDeps`
   - `src/cron/service/timer.ts` — wraps existing `runDueJob` and `runStartupCatchupCandidate`; medium conflict risk if upstream restructures the catch-up flow (e.g. they extract a helper or change the outcome shape)
   - `src/gateway/server-cron.ts` — additive (new helper + one new prop on the `CronService` constructor); medium conflict risk if upstream changes `buildGatewayCronService` signature
4. **Resolution rules:**
   - Always preserve the `beforeFireSlot` hook check in `runDueJob` and `runStartupCatchupCandidate` — this is the load-bearing duplicate-fire fix.
   - If upstream changes the `TimedCronRunOutcome` shape, update the `status: 'skipped'` return inside the hook check to match; do not drop the check.
   - If upstream adds its own slot/lease primitive, prefer the upstream API and remove our additions, updating this doc and the hos-openClaw consumers (`buildHosSlotLeaseHook`, `claimSlot.ts`).
5. **After merging,** run the full test suite plus a manual cold-wake test (described in `hos-openClaw/docs/`).
6. **Update this file** if any of the touched-files lists or rationale changed.

### When upstream lands a feature we forked for

Remove the fork addition and adapt to the upstream API in the same PR that bumps the submodule. Update the corresponding "Fork additions" section here to record the migration.

### When adding a new fork-only change

Each new fork addition needs:

- A new section under "Fork additions" with the same shape as the existing entries
- A clear "Why not a plugin / SDK extension" justification — if the answer is "we didn't try," go try
- A concrete upstream plan (RFC, wait-and-see, never-upstream) — don't accumulate forks without intent
- Conflict-risk notes for future merges
