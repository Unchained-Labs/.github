# Do the graph tools help the Kymatics stack?

Written August 2026, after building the five graph tools and then pointing them at
our own product. The question was whether they actually help Kymatics or whether
they are a separate hobby that happens to live in the same org.

**Short answer: three of the five help today, two of those found real bugs on
first contact, and two do not apply at all yet.** The negatives are the useful
part of this document — a report where all five tools turned out to help would
tell you nothing except that the author wanted them to.

The third only counts because the exercise itself produced the missing piece:
authsweep could not read Rust, so it was written to. That is the honest shape of
dogfooding — the gap was not "Otter needs work", it was "our tool cannot see
our own control plane".

## The stack, for orientation

| Component | What it is | Language |
| :--- | :--- | :--- |
| [Otter](https://github.com/Unchained-Labs/otter) | Orchestration engine: queues prompts, runs them in isolated workspaces, streams output, persists history | Rust (axum, Postgres, Redis) |
| [Lavoix](https://github.com/Unchained-Labs/lavoix) | STT/TTS service behind the voice input | Python (FastAPI) |
| [Seal](https://github.com/Unchained-Labs/seal) | Voice-first Kanban frontend | React / TypeScript |
| [Kymatics](https://github.com/Unchained-Labs/kymatics) | The umbrella: all three plus a landing page | — |

---

## 1. authsweep → Lavoix — works today, and found two real gaps

**Status: proven.** Run it yourself:

```sh
git clone https://github.com/Unchained-Labs/lavoix
npx authsweep scan lavoix/src
```

```
  authsweep  12 files · 3 routes · fastapi

  prefilter  1 of 3 routes dropped before any agent ran

  ! POST /v1/stt/transcribe  src/lavoix/api.py:29
     no authorization check found on or above this handler · POST changes state

  ! POST /v1/tts/synthesize  src/lavoix/api.py:54
     no authorization check found on or above this handler · POST changes state

  2 medium
```

Both endpoints accept input and do **paid provider work** — an upload goes
straight to a speech provider that bills per second of audio. Neither has an
authorization check. `/healthz` is correctly prefiltered as conventionally public.

This is rated `medium` rather than `high` because the deployed topology matters:
inside the Compose stack Lavoix is reached by Otter over the internal network, not
published. The finding is still real — the service has no opinion about who is
calling it, so the moment it is exposed, or the moment anything else inside the
network is compromised, it is an open billing endpoint. **The fix is either an
auth dependency or an explicit `# public` marker** so the next scan stays quiet
for a stated reason rather than by accident.

### Pointing the tool at our own code found a bug in the tool

The first run reported **"every route has an authorization check"** — a clean
bill of health. That was wrong, and wrong in the worst available direction.

The guard pattern treated any FastAPI `Depends(...)` as an auth check.
`Depends(get_service)` is a service locator, not a guard. So authsweep read
dependency injection as authorization and went quiet on two open endpoints.

`SECURITY.md` in that repo calls a false clean the most serious defect the tool
can have, because silence reads as safety. Fixed in
[`7141abe`](https://github.com/Unchained-Labs/authsweep/commit/7141abe):
`Depends()` and `Security()` now count only when the dependency is *named* like an
auth check, with three regression tests and a fixture taken from Lavoix's exact
shape.

Worth stating plainly: **the tool would still be reporting a false clean if it had
not been run against real code.** Fixtures written by the person writing the rule
share that person's blind spots.

---

## 2. preflight ↔ Otter — the strongest fit, and it closes preflight's biggest weakness

**Status: half shipped.** Otter already has the other half of preflight's cost
model, built independently:

| | preflight | Otter (`otter-core/src/usage.rs`) |
| :--- | :--- | :--- |
| When | **Predicts** before the run, from a spec | **Measures** after the run, from the transcript |
| Prices | `PRICING`, USD per MTok, CI-checked for staleness | `PricingTable`, USD per MTok, from `OTTER_MODEL_PRICING` |
| Output | agents, tokens, USD as a range | `otter_tokens_total`, `otter_estimated_cost_usd_total` |
| On no data | states its assumptions | records tokens, reports cost as `None` |

Both were written with the same instinct, which is why they fit: *cost is derived,
never reported by the provider, so keep it explicitly separate and say when you do
not know.* Otter's comment on an empty price list — "a bad price string should cost
you cost-reporting, not your control plane" — is the same judgment as preflight
refusing to quote an expired intro rate.

### What is shipped

`preflight models --format otter-env` emits Otter's price list directly:

```sh
$ preflight models --format otter-env
claude-fable-5=10:50,claude-opus-5=5:25,claude-sonnet-5=2:10,claude-haiku-4-5=1:5,…
```

```sh
# in Otter's .env
OTTER_MODEL_PRICING=$(preflight models --format otter-env)
```

One price list instead of two. The exported one is the CI-checked one —
preflight's build **fails** when its table is more than 120 days old, whereas a
hand-set environment variable rots silently. It also respects intro-rate expiry:
after 2026-08-31 the exporter emits Sonnet 5 at `3:15` rather than the
promotional `2:10`, because exporting an expired rate would make Otter's cost
reporting quietly under-report.

### What is not built yet: the calibration loop

preflight's most honest documented weakness is that its token profiles are
generic — its README tells you to run one real workflow, read the actual counts
out of your spans, and put them in `preflight.json`.

**Otter is a machine that produces exactly those spans.** `otter_tokens_total`
and the per-job `TokenUsage` parsed out of the agent transcript are the
calibration data preflight asks for and cannot generate on its own. An exporter
in the other direction — Otter's measured usage → a `preflight.json` — would turn
preflight from "generic defaults" into "calibrated against our own runs".

That is the highest-value unbuilt thing in this document, and it is small.

---

## 3. decorrelate → Otter's evals — nothing to add, and that is a compliment

**Status: does not apply, correctly.**

Otter's eval suite scores scenarios on **delivery rate**, not success rate, and its
README explains why: `status == "succeeded"` only means the agent process exited
zero, so the gap between success and delivery is "exactly the population of runs
that looked fine and shipped nothing."

That check — *did a preview URL actually publish and respond?* — is an
**executable oracle**. Deterministic, zero tokens, no shared priors.

decorrelate's fix list is, in order of return: vary the lens, vary the model,
**prefer an executable oracle**, use asymmetric thresholds. Otter's evals already
sit at step three. There is no verifier panel to measure the independence of,
because there is no model verifier in the loop at all.

decorrelate becomes relevant to Otter the moment a model judge enters eval
scoring — grading code quality, or judging whether a built app matches the prompt's
intent, where no oracle exists. Until then the correct answer is that the tool has
nothing to offer, and Otter is already doing the thing the tool would have told it
to do.

---

## 4. graphlint → Otter — does not apply today

**Status: does not apply.**

An Otter job is a single prompt executed in one workspace. graphlint analyses
multi-node graphs — fan-out width, barriers, verifier panels, cycles — and none of
those concepts exist in a single-prompt job. There is nothing for it to read.

It would apply if Otter grew **declarative multi-step jobs** (plan → build →
verify → deploy as separate queued nodes with typed edges). At that point Otter
jobs become graph specs and every rule lands: is that barrier justified, is a
schema attached to the edge between build and verify, does the retry loop have a
round cap. Speculative, but it is the natural direction for an orchestration
engine, and the spec format already exists.

---

## 5. authsweep → Otter — was blocked, so we unblocked it

**Status: works now, and it is the most serious finding in this document.**

Otter's HTTP surface *is* the control plane: `/v1/jobs`, `/v1/projects`,
`/v1/workspaces/{id}/command`, `/metrics`, plus workspace filesystem endpoints
that read files out of a build workspace.

The first run found nothing, because authsweep read Express, Fastify, Koa,
FastAPI and Flask — and Otter is axum. It printed **"every route has an
authorization check"** and exited `0`. A CI job would have gone green on a
completely unscanned control plane. Where the `Depends()` bug mis-read a guard,
this one read nothing at all; both make silence look like safety. The immediate
fix was to stop lying about it
([`2b58024`](https://github.com/Unchained-Labs/authsweep/commit/2b58024)): a
zero-route scan now says plainly that nothing was examined and sets
`summary.examinedNothing`.

The real fix was to read Rust
([`b3f9c76`](https://github.com/Unchained-Labs/authsweep/commit/b3f9c76)). axum,
actix-web and rocket, token-structural rather than parser-based, resolving
chained method routers, `nest`/`mount`/`scope` prefixes, and auth-looking
extractor types in handler signatures. On Otter:

```
  authsweep  17 files · 38 routes · axum

  prefilter  2 of 38 routes dropped before any agent ran

  ✗ POST /v1/workspaces/{id}/command  otter-server/src/main.rs:244
     .route("/v1/workspaces/{id}/command", post(run_workspace_command))
     no authorization check found on or above this handler · POST changes state ·
     executes code or shell commands · takes an id or wildcard

  ✗ POST /v1/workspaces/command  otter-server/src/main.rs:245
  ✗ GET  /v1/runtime/workspaces/{id}/shell/ws  otter-server/src/main.rs:271

  3 high · 21 medium · 12 low
```

38 routes, exactly the 36 `.route()` calls plus the two that chain a second
method. The two prefiltered are `/healthz` and `/metrics`.

**Otter's control plane has no authorization at all.** That is a defensible
choice for a localhost dev orchestrator and an indefensible one the moment it
binds to anything else, and the three `high` findings say why: two of them accept
a command and run it in a workspace, and the third is a shell over a websocket.
This is the single most valuable thing any of the five tools has produced, and it
was invisible until the tool could read the language the service is written in.

### Writing it taught the tool three things the fixtures could not

- **`web::scope("/admin").wrap(auth)` chains *after* its own arguments.** Scoping
  the guard to the enclosing function marked every sibling route as covered — a
  false clean over `/v1/secrets/{id}`. Prefixing constructs now own a region, and
  where that region ends depends on the framework.
- **Resolving `.mount("/api/v1", …)` broke the public-path list.** `/ping` became
  `/api/v1/ping`, which a head-anchored pattern stopped recognising, so the tool
  began reporting health checks *because it had got better at reading the router*.
- **The severity table had no notion of code execution.** It was written against
  CRUD apps, so unauthenticated `POST /workspaces/{id}/command` ranked level with
  `POST /projects`. Rating remote code execution the same as "create a record" is
  exactly the "teaches you to ignore both" failure the module's own docstring
  warns about.

None of those three is reachable from a fixture, because a fixture only contains
the shapes its author already thought of.

---

## Summary

| Tool | Kymatics fit | Status |
| :--- | :--- | :--- |
| **authsweep** | Lavoix (FastAPI) | ✅ Works. Found 2 unauthenticated paid endpoints, and a false-clean bug in itself. |
| **authsweep** | Otter control plane | ✅ Works now — the Rust front-end was written for this. 38 routes, 3 high, no authorization anywhere. Found the *second* false clean on the way. |
| **preflight** | Otter cost model | ✅ Pricing export shipped. ⏳ Calibration loop unbuilt — highest-value next step. |
| **decorrelate** | Otter evals | ➖ Nothing to add; Otter already uses an executable oracle. |
| **graphlint** | Otter jobs | ➖ Does not apply — single-prompt jobs are not graphs. |

Three of five help. But the most useful thing that came out of the exercise was
not an integration at all: **pointing a security tool at our own stack found two
bugs in the security tool**, both of the single class its own threat model calls
the worst one — a false clean.

- Against Lavoix it mis-read dependency injection as an auth guard and reported a
  clean bill of health over two open billing endpoints.
- Against Otter it found no routes at all, because it cannot read Rust, and
  reported that as a pass.

Neither was reachable from the fixtures, because the fixtures were written by the
same person who wrote the rules and inherited the same blind spots. Real code had
shapes the author did not think to invent. And fixing the second one surfaced
three more defects — a guard scoped to the wrong region, a public-path list that
broke as soon as prefixes resolved, and a severity model that could not tell
remote code execution from a record insert.

That is the argument for dogfooding, stated more sharply than any README could
put it. It is also the argument for writing the negatives down: the two rows in
the table above that say "does not apply" are the reason you can believe the
three that say it does.

## What is still unbuilt

- **preflight ← Otter calibration.** Otter measures the token counts preflight
  guesses at. An exporter from measured usage to `preflight.json` is small and is
  the highest-value item here.
- **Otter has no authorization.** The scan above is the argument, not the fix.
  Whether it needs one depends on whether it stays a localhost tool.
- **decorrelate stays on the shelf** until a model judge enters Otter's eval
  scoring. Until then the executable oracle is the better instrument and the tool
  has nothing to add.
