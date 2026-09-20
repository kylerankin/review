---
name: hive-factory-evidence
version: "1.0"
last_updated: 2026-09-20
id: hive-factory-evidence
one_line_purpose: Project only supported Hive factory evidence read-only into OMP Review.
entry_point: docs/skills/hive-factory-evidence.md
category: ci-ops
mcp_compliance_level: partial
optimization_status: draft
status: active
dependencies: []
tags: [hive, omp, read-only, evidence, projection]
description: "Maps which Hive factory facts OMP Review may project read-only, the endpoint/schema/auth/freshness of each, and why project convergence has no supported read mapping. Use before adding any Hive-sourced field to the queue card, rail, or status segment."
metadata:
  type: policy
---

# Hive Factory Evidence Projection

## When to Use

Load this before adding any Hive-sourced field to the OMP Review queue card,
rail, status segment, or a structured consumer. It decides whether a fact may
be projected at all, and if so from which endpoint, with what auth, and how to
report it when stale. It is the read-only half of
[#314](https://github.com/projectbluefin/review/issues/314), which keeps Review
useful without Hive and forbids every mutation.

The governing rule: Review makes human judgment fast; Hive makes work happen.
A refresh may read evidence and display its provenance, age, freshness,
confidence, and uncertainty. It may **not** plan, admit, rank, schedule,
assign, continue, complete, merge, or write to GitHub or Hive.

## When Not to Use

Do not use this to justify projecting a fact whose upstream mapping is
unverified. A fact with no confirmed read contract shows as unavailable, never
as a derived number. Do not use this to build a scheduler, admission engine,
convergence counter, or attention metric. Those are non-goals of #314.

## The Projection Boundary

Every projected field must carry three things the maintainer needs to trust it:

1. **Source** — the endpoint that produced it, retained on the model.
2. **Freshness** — when it was fetched (`fetchedAt`, already on
   `HiveSnapshot`), surfaced through `queueAge` as text only once it is old.
3. **Confidence** — `known`, `unknown` (endpoint answered but the field is not
   mapped here), or `unavailable` (endpoint did not answer). These are three
   distinct states and must never render the same.

The existing read model in `image/extension/bluefin-review/hive.ts` already
carries all of this: `hub`, `configured`, `online`, `actionableItems`,
`items`, `triage`, `ranks`, `claims`, `error`, `fetchedAt`. A new field attaches
to that snapshot; it never invents a second.

## Supported Evidence Sources (verified against `hivecommons/hive` @ `d0fc9cc`)

The route table and the top-level response shape below were read from the
pinned SHA. Field-level schemas for `ReadyQueueItem` and the event stream live
in sibling package files and are marked **confirm** — read them against the
live endpoint before projecting the nested fields.

| Endpoint | Handler | Confirmed top-level fields | Auth | Freshness |
|---|---|---|---|---|
| `GET /api/contribute/status` | `handleContributeStatus` | `hub`, `active_contributors`, `total_registered`, `actionable_items`, `candidate_items`, `surface`, `api_version`, `served_sha` | anonymous read; `X-Hive-Contribute-Protocol` header identifies the answering surface | per fetch; `served_sha` + `api_version` prove which hub answered |
| `GET /api/contribute/queue` | `handleContributeQueue` | `queue_total`, `held_total`, `queue[]`, and — only when convergence shadow-mode (#4246, default **off**) is on — `withheld`, `admission_coverage` | contributor token | per fetch |
| `GET /api/contribute/activity` | `handleContributeActivity` | `activity[]` of `RepoActivity`: `repo`, `issues`, `prs`, `merges`, `claims`, `reviews`, `advisory`, `reconciled`, each `{count, newest_at}`; `collected_at`, `window_hours` | contributor token | `collected_at` + `window_hours` |
| `GET /api/contribute/fleet` | `handleContributeFleet` | fleet/work snapshot + `ContributeAdmissionPolicy` (`suspended`, modes, deny/allow lists) | contributor token | per fetch |
| `GET /api/contribute/events` | `handleContributeEvents` | **confirm** event schema | contributor token | per fetch |
| `GET /api/v1/{status,queue,contributors,activity,knowledge,me}` | `handleAPIv1` | authenticated, caller-scoped where noted | GitHub token / authorized user | per fetch |

`status` already flows to `HiveSnapshot.actionableItems`; `activity` already
flows to the CI tally and the rail age badge. Nothing here is new.

## The One Missing Decision: Project Convergence

The maintainer decision this maps: **"Is this project's Hive-queued backlog
shrinking or growing, and how much of it is done?"** That is a convergence
signal. It has **no supported read-only mapping**, and this is the finding, not
a gap to paper over.

Two independent reasons, both read from the pinned source:

1. **The only convergence endpoint is an owner-gated write.**
   `handleConvergenceConfigGet` / `handleConvergenceConfigPut`
   (`api_config_convergence.go`) begin with `requireOwnerRole`, and the PUT
   mutates `convergence.mode` through `saveConfig`. It lets an owner *set* the
   rollout mode; it does not *report* a convergence stage or percentage. A write
   control is not evidence, and projecting it would be a mutation in disguise.

2. **The read endpoints report depth, not convergence.** `handleContributeQueue`
   returns `queue_total` / `held_total` and — only under the default-off
   shadow-mode toggle — `withheld` / `admission_coverage`. `handleContributeActivity`
   returns per-repo action counts over a window. None is a convergence stage,
   percentage, or authoritative frontier. Deriving one from queue counts,
   labels, local ranking, or the activity deltas is exactly what #314 forbids
   ("never derive convergence from queue counts").

Therefore a projected convergence field is `unknown` (the endpoint exists but
does not return this fact) until a supported read contract is judged and
verified. It is never fabricated, and never recomputed from the queue.

## Stale, Unknown, and Unavailable Behaviour

- **`unknown`** — a mapped endpoint answered, but the fact is not among its
  fields (e.g. convergence on any hub). Show the source and `unknown`, not a
  number.
- **`unavailable`** — the endpoint did not answer, or the hub is
  configured-but-broken. `HiveSnapshot.error` already carries the reason and
  `orderSourceLabel` in `rail.ts` renders it as `hive ▸ hive unreachable
  (…)`. A configured-but-broken hub must stay an error, never an empty queue.
- **stale** — `queueAge(fetchedAt, now)` in `rail.ts` shows the age as text
  only after `STALE_AFTER_MS` (90 s), so a fresh read is silent and an old one
  is explicit.

## Evidence It Is Not Already Displayed

The current OMP projection shows, and only these:

- `hive.ts` — hub identity, online/configured, `actionableItems`, the queued
  `items`, `triage` groups, `ranks`, `claims`, `error`, `fetchedAt`.
- `priority.ts` — a category (`hive`/`review`/`fix-ci`/…), Hive's rank, and a
  human reason.
- `rail.ts` — `orderSourceLabel` (`present/total queued`, actionable count, or
  unreachable), the per-item priority chip, and `queueAge`.

None projects a convergence stage, percentage, frontier, attention count,
Continuity claim, authority level, or control posture. Adding one here is the
whole of #314's remaining work, and this skill says it stays `unknown` until a
read mapping is verified.

## Red Flags

- Projecting a number an endpoint does not return, or deriving convergence from
  queue counts, labels, ranking, or activity deltas.
- Projecting through a write endpoint (any `requireOwnerRole`, any `PUT`/`POST`).
- Rendering `unknown` and `unavailable` identically.
- Turning a configured-but-broken hub into an empty or "converged" queue.
- Building a scheduler, admission/assignment engine, or attention metric.
- Citing a branch path instead of the pinned SHA this maps against.

## Verification

```bash
bash scripts/check-skill-frontmatter.sh
bash tests/generate-skills.sh
git diff --check
just --list
pre-commit run --all-files
```

To verify a nested field before projecting it, read the handler at the pinned
SHA and confirm the JSON tag in the live response, then cite the SHA with line
anchors — never a branch path.
