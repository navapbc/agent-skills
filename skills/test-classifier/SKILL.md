---
name: test-classifier
description: >
  Triage failing tests by classifying each one into exactly one of four
  verdicts — APPLICATION_BUG, TEST_BUG, FLAKY_FAILURE, or ENVIRONMENT_ISSUE —
  so teams never auto-generate no-op tests for genuinely broken code, never
  "fix" the application when the test itself is stale, and never burn a
  code/test patch on an intermittent or infrastructure failure. Grounded in the
  question "is the test wrong or the code wrong?" and aligned to the
  industry-standard four-way failure taxonomy. For each failing test the skill
  produces a verdict, a short rationale, a failure category (visual / behavioral
  / E2E form-flow drift), and an honest confidence level, and presents a
  readable triage report directly to the user. Use this skill whenever a test
  run has failing tests and you want to know whether the test, the code, the
  test's stability, or the environment is at fault — without writing any patch
  or generating any test.
---

# Test Classifier Skill

## Overview

A triage layer that sits between "a test went red" and "a human decides what to
do about it." Its **primary job** is to answer one question for every failing
test: *is the test wrong, or is the code wrong?* — and to say so out loud, with
a rationale, so that downstream automation (or a tired developer at 2pm on a
Friday) never patches the wrong side of the failure.

The skill triages the failing tests an agent encounters — whether running the
suite locally, reviewing a change, or reading captured test output — and
presents its verdicts directly to the user as a readable report. It writes no
code and generates no tests; its entire value is the verdict and the rationale.

---

## The Four-Verdict Taxonomy (what we are classifying)

Every failing test maps to exactly one of four verdicts. This is the whole
model; everything else in this skill is in service of getting the verdict right.
The taxonomy follows the industry-standard four-way split, grounded in the
framing *is the test wrong or the code wrong?* — extended so that an
intermittent or infrastructure failure is never mistaken for either.

| Verdict | Test result | Root cause | What a human should do |
|---|---|---|---|
| `APPLICATION_BUG` | **Fails** | The app **regressed**; the test correctly caught a real defect. | Fix the CODE. The test is doing its job — do **not** relax it. |
| `TEST_BUG` | **Fails** | The app is **correct**; the test is stale or a change-detector asserting on an intended change. | Fix the TEST. Update it to match the new, correct behavior. |
| `FLAKY_FAILURE` | **Fails intermittently** | Non-deterministic test/timing/ordering; passes on re-run with no code change. | Re-run to confirm; then deflake the test. **Not** a code or test-logic patch. |
| `ENVIRONMENT_ISSUE` | **Fails** | Infrastructure: timeout, network blip, port contention, runner resource exhaustion, missing service. | Fix the environment / re-run. Neither the app nor the test is at fault. |

A test that **passes** is not a failure and is not classified — it simply does
not appear in the output. If nothing failed at all, there is nothing to
classify. There is no per-test "no-action" verdict.

> **Why this distinction is load-bearing.** The expensive failure mode is not a
> red build — it is "fixing" the wrong thing. If the app is correct and we
> blindly regenerate or relax the test, we erode the test's value
> (`TEST_BUG` mishandled). If the app regressed and we instead "update the
> snapshot" to make the test pass, we ship a bug with a green checkmark
> (`APPLICATION_BUG` mishandled). And if we invent a code regression to explain
> a timeout, we waste a developer's afternoon chasing a ghost
> (`FLAKY_FAILURE` / `ENVIRONMENT_ISSUE` mishandled). The classifier exists to
> keep these four cases apart **before** anyone writes a patch — and
> specifically so teams never generate no-op tests for genuinely broken code,
> and never patch code to chase a flaky test.

---

## When to Use

Use this skill whenever a test run has failing tests and you want to know
whether the test, the code, the test's stability, or the environment is at
fault — before anyone writes a patch. It is advisory: it produces verdicts and
rationales, not fixes.

**Two directions are explicitly OUT of scope.** They are deliberate
non-features, not omissions:

- **Do not emit applyable code suggestions.** This skill does not propose commit
  patches for `APPLICATION_BUG` / `TEST_BUG` verdicts.
- **Do not generate tests.** This skill only classifies failures that already
  exist; it never authors brand-new tests.

If you find yourself wanting to write a patch or author a test, stop: that is
out of scope. The value of an `APPLICATION_BUG` verdict is precisely that it
tells a human to fix the code rather than silently regenerating the test to
pass.

---

## Core Process

1. **Collect the failing-test signal** — which tests failed, with their output
   (OBSERVED if you ran the suite, INFERRED if you predicted from the diff).
2. **Collect the diff** — the change under test, against the base ref / branch
   you are comparing against.
3. **For each failing test, decide the verdict** — work the decision procedure
   in order: ENVIRONMENT_ISSUE → FLAKY_FAILURE → TEST_BUG vs. APPLICATION_BUG.
4. **Tag a failure category** — visual drift / behavioral drift / E2E form-flow
   drift / other.
5. **Assign a confidence** — high / medium / low, honestly.
6. **Present a readable triage report** to the user.

---

## Step 1 — Collect the Failing-Test Signal

The classifier needs to know *what failed and why it says it failed*. There are
two ways to get that signal, and which one you use determines the result `mode`:

### OBSERVED — run the suite yourself (preferred when you can)

When you are able to execute the repo's test suite, **locate and run it
yourself**, then classify the failures you actually observe:

1. **Locate the test command.** Inspect the checked-out repo: `package.json`
   `scripts.test`, a `Makefile` `test:` target, `pytest.ini`/`tox.ini`/
   `pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, or the repo's own CI
   workflow's test step (`.github/workflows/*`). Use what the repo actually uses.
2. **Install dependencies** using the repo's own lockfile, best-effort —
   `npm ci`/`pnpm i`/`yarn`, `pip install -r …`/`poetry install`, `go mod
   download`, `cargo fetch`, etc. Only what the available toolchain supports.
3. **Run the suite** (prefer the non-watch / single-run invocation; e.g. add
   `--run` for vitest) and capture pass/fail plus the failure output. This is
   your OBSERVED signal.
4. Treat the result `mode` as `"OBSERVED"`.

> **Infrastructure-as-code tests need a teardown guarantee — never strand real
> resources.** Some repos (Terraform/OpenTofu modules, Pulumi, CloudFormation)
> have a "test suite" that **applies real cloud resources**. That is fine to run
> *only* when teardown is guaranteed; the failure mode to avoid is an interrupted
> run (timeout, turn cap) that leaves **orphaned resources** in the cloud account
> (real cost + security exposure). Detect IaC first (`*.tf`,
> `*.tftest.hcl`/`.tftest.json`, a `terraform`/`tofu`/Terratest step, `Pulumi.yaml`),
> then work down this ladder and stop at the first rung you can do safely:
>
> 1. **No-provision checks (always safe, prefer these).** `terraform validate`,
>    `terraform fmt -check`, and `terraform test` **when it cannot apply** — every
>    `run` block is `command = plan`, or the tests use `mock_provider`/`override_*`
>    (TF ≥1.7) so there are no real provider calls. A genuine OBSERVED signal with
>    zero infrastructure. Terratest always applies, so it does **not** qualify here.
> 2. **Apply *with* a guaranteed teardown.** Run a real apply-mode suite only if a
>    teardown is **guaranteed regardless of how this run ends**, i.e. one of:
>    - the repo has a **separate, PR-scoped cleanup** that destroys by environment
>      key on PR close (e.g. a `cleanup-preview` workflow that runs
>      `terraform destroy` for `preview-pr-<N>`) — teardown is decoupled from your
>      run, so an interruption can't strand anything; **or**
>    - you run the destroy yourself **in the same execution** immediately after the
>      assertions, with the destroy guarded so it runs even on test failure (the
>      native `terraform test` lifecycle does this; a raw `apply` does not unless
>      you pair it with `terraform destroy`).
>    Use an **isolated state prefix / ephemeral environment name** so a destroy
>    can't touch anything real, and only proceed if the apply can finish well
>    inside your time budget (leave room for the destroy). If you can't guarantee
>    both, do not apply.
> 3. **Otherwise fall back to INFERRED.** If the only way to exercise the tests is
>    an unguarded real apply, do **not** run it — predict from the diff and say so
>    in the summary (e.g. "Terraform tests provision real infra and no guaranteed
>    teardown was available; ran validate + plan-mode, predicted the rest from the
>    diff"). A safe prediction beats an orphaned VPC. Same rule for any test that
>    stands up real external state (Pulumi, CloudFormation, live DB migrations).

### INFERRED — predict from the diff (fallback)

If you cannot run the suite — no test command found, missing toolchain, the
suite needs services like a database, it times out, **or it would apply real
infrastructure with no guaranteed teardown** (see the IaC ladder above) — do
**not** fabricate a run. Instead reason statically over the diff (Step 2) and
PREDICT which tests the change would cause to fail and why. Then:

- Treat the result `mode` as `"INFERRED"`.
- State the reason you fell back in the summary (e.g. "no test script found",
  "deps failed to install", "suite timed out") so the reader sees why this is a
  prediction, not an observation.
- Prefer lower confidence, and remember `FLAKY_FAILURE`/`ENVIRONMENT_ISSUE` are
  generally NOT determinable from a diff alone — only assert them with explicit
  evidence.

### Either way

Gather, in order of preference: the test runner's failure output (assertion
messages, expected-vs-actual diffs, stack traces, snapshot diffs); the names and
file paths of the failing tests; for visual/snapshot tests, the recorded
baseline-vs-actual diff if available.

If no test failed (the run is green, or the diff implicates no test), there is
nothing to classify: report that no failing tests were found, and stop. Passing
tests are never listed.

---

## Step 2 — Collect the Diff

Determine the base ref / branch you are comparing against (from context, or ask
the user if it is ambiguous), then collect the change under test:

```bash
git diff "<base-ref>" HEAD --unified=5      # full content of the change
git diff "<base-ref>" HEAD --name-only      # list of changed paths
```

The diff is your evidence for the central question: did this change *intend* to
alter the behavior the failing test asserts on? An intended behavior change that
the test wasn't updated for points toward `TEST_BUG`. A change that should NOT
have altered that behavior, yet did, points toward `APPLICATION_BUG`.

---

## Step 3 — Decide the Verdict (the decision procedure)

For each failing test, work through this procedure **in order** and stop at the
first verdict that fits. Be explicit in the rationale about which branch you took
and what evidence drove it. The order matters: rule out non-deterministic and
environmental causes *before* attributing the failure to the app or the test,
so you never invent a code regression to explain a timeout.

1. **Is this an environment/infrastructure failure?** Signals: connection
   refused, DNS/network error, port already in use, out-of-memory or disk on the
   runner, a missing service/dependency, a credential/secret not present, an
   HTTP 5xx from a backing service the test depends on. If the failure is
   about the *substrate the test runs on* rather than the code it exercises,
   verdict = `ENVIRONMENT_ISSUE`. Recommend fixing the environment or re-running;
   do not patch app or test logic.
2. **Is this a flaky/non-deterministic failure?** Signals: the failure is
   timing- or ordering-dependent, the test uses real clocks/sleeps/animations,
   it depends on test-execution order or shared mutable state, or the same commit
   is known to pass on re-run. If the test would likely pass on an unchanged
   re-run, verdict = `FLAKY_FAILURE`. Recommend a re-run to confirm, then a
   deflaking task — **not** a code or test-logic patch.
3. **Otherwise it is a deterministic, code-or-test failure. Determine the app's
   intended behavior** for the assertion that failed. Use: the diff (what did the
   author set out to change?), the test's own intent (what contract is it
   guarding?), surrounding code, and any spec/PR-description signal.
4. **Is the app's *current* behavior correct** with respect to that intended
   behavior?
   - **Yes — the app is doing the right thing, the test is asserting the old
     thing.** Verdict = `TEST_BUG`. The test is stale, or it is a
     change-detector that is firing on an intended change. The TEST should be
     updated to match the new, correct behavior.
   - **No — the app is doing the wrong thing; the change introduced a
     regression the test correctly caught.** Verdict = `APPLICATION_BUG`. The
     CODE should be fixed; the test is doing its job.
5. **If you cannot tell** which verdict is correct from the available evidence,
   do not guess confidently. Pick the more likely verdict, set
   `confidence: low`, and say plainly in the rationale what additional signal a
   human would need (e.g., "needs product decision on whether the new copy is
   intended", or "re-run needed to rule out flakiness before calling it an
   APPLICATION_BUG").

### Tie-breakers and edge cases

- **Flaky vs. APPLICATION_BUG is the dangerous confusion.** A real regression can
  *look* intermittent (e.g., a race the change introduced). When a failure is
  plausibly flaky but you cannot be sure it isn't a real defect, prefer
  `FLAKY_FAILURE` with `confidence: low` and explicitly recommend the re-run as
  the disambiguator — never assert `APPLICATION_BUG` on a single flaky-looking
  run.
- **Deterministic-tooling failures** — a type error, lint failure, schema
  violation, or compile error masquerading as a "test failure" — are best fixed
  by the tool that owns them. Classify as `TEST_BUG` only if the test itself is
  malformed; otherwise treat a broken build step as `ENVIRONMENT_ISSUE`
  (the test never got to run) and say so in the rationale.
- **Manual/exploratory judgment calls** — "is this the *right* UX?" is a product
  decision, not a static one. If correctness genuinely needs a human to look,
  give your best verdict at `confidence: low` and name the decision the human
  must make.

When in doubt, prefer `confidence: low` and an honest rationale over a confident
wrong call. A low-confidence-but-correct triage is useful; a high-confidence
wrong one trains the team to distrust the classifier.

---

## Step 4 — Tag a Failure Category

Tag each failing test with one of the following categories. These describe the
*kind* of failure, independent of the verdict — a `behavioral-drift` failure can
be any of the four verdicts.

| Category | What it means | Typical signal |
|---|---|---|
| `visual-drift` | A rendered-UI / snapshot / screenshot test failed because pixels or DOM structure changed. | Image-diff or snapshot mismatch; "X% of pixels differ"; updated component markup. |
| `behavioral-drift` | A unit/integration test failed because a function's logic, return value, or side effect changed. | Assertion on a value/branch/exception that no longer holds. |
| `e2e-form-flow-drift` | An end-to-end test driving a multi-step user flow (especially form submission flows) failed at some step. | Selector not found, step timed out, validation/redirect path changed, submit produced a different result. |
| `other` | Does not fit the above — e.g. a build/tooling failure, or an infrastructure failure with no test-kind signal. | Lint/type/compile errors, runner OOM, connection refused. |

The category is advisory metadata; it does not change the verdict logic. Note
that `verdict` and `category` are orthogonal: a `FLAKY_FAILURE` on a Playwright
form test is `category: e2e-form-flow-drift`; an `ENVIRONMENT_ISSUE` from a
runner OOM with no particular test-kind signal is `category: other`.

---

## Step 5 — Assign Confidence

Every classification carries a `confidence` of `high`, `medium`, or `low`:

- **high** — the diff and the failure output unambiguously point to one verdict.
  (e.g., the author intentionally changed a label string and only the
  string-assertion test broke → `TEST_BUG`, high.)
- **medium** — the evidence leans one way but a reasonable reviewer could
  disagree, or some context is missing.
- **low** — genuinely uncertain: the failure could be flaky vs. a real
  regression, or correctness needs a human/product judgment. Always pair `low`
  with an actionable rationale naming the disambiguator (usually a re-run or a
  product decision).

---

## Step 6 — Present the Triage Report

Present your findings directly to the user as a readable report. Group by file,
lead each finding with its verdict, and keep rationales to one or two sentences.
One classification per failing test (not per assertion): if a single test has
three failing assertions for one root cause, emit one classification.

```
## Test Classifier Report
**Scope:** Change under test = diff against <base-ref> (N files changed)
**Mode:** OBSERVED (ran the suite) — or INFERRED (predicted from the diff; <why>)
**Failing tests classified:** K

---

### Classifications by file

#### `src/components/Banner.test.tsx`

- **TEST_BUG** | visual-drift | confidence: high
  `Banner › renders the announcement copy`
  The diff intentionally updates the banner copy from "Welcome" to "Welcome back".
  The app is rendering the new, correct copy; the snapshot is stale. Update the test.

#### `src/api/checkout.test.ts`

- **APPLICATION_BUG** | behavioral-drift | confidence: high
  `checkout › applies the loyalty discount`
  The diff refactored `applyDiscount()` and now returns the pre-discount total.
  Nothing in the change intended to remove the discount — the code regressed. Fix the code.

#### `e2e/signup.spec.ts`

- **FLAKY_FAILURE** | e2e-form-flow-drift | confidence: low
  `signup › submits the registration form`
  The submit step times out at the "Create account" button on this run only;
  nothing in the diff touches the submit handler. Likely a flaky selector/wait.
  Re-run to confirm before treating it as a regression.

#### `tests/integration/orders.test.ts`

- **ENVIRONMENT_ISSUE** | other | confidence: high
  `orders › fetches the order history`
  The test failed with "connection refused" to the orders DB on the runner. The
  backing service was unavailable; neither the app nor the test is at fault.
  Re-run once the service is available, or fix the service definition.

---

### Summary
| Verdict           | Count |
|---|---|
| APPLICATION_BUG   | 1 |
| TEST_BUG          | 1 |
| FLAKY_FAILURE     | 1 |
| ENVIRONMENT_ISSUE | 1 |

**Recommendation:** 4 failing tests triaged
(1 APPLICATION_BUG, 1 TEST_BUG, 1 FLAKY_FAILURE, 1 ENVIRONMENT_ISSUE).
```

For each finding, keep the rationale terse — name the mismatch and what to
change, not a narrative. For `APPLICATION_BUG`, say plainly the fix belongs in
the app code, not the test (the guardrail against no-op test generation). For
`FLAKY_FAILURE`, name the re-run as the disambiguator. State the verdict exactly
as written — `APPLICATION_BUG`, `TEST_BUG`, `FLAKY_FAILURE`, or
`ENVIRONMENT_ISSUE` — so the report is unambiguous.

---

## Common Rationalizations

Watch for these tempting-but-wrong shortcuts:

- *"The test is red, so just update the snapshot to make it green."* — Only if
  the app's new behavior is actually correct (`TEST_BUG`). If the app regressed,
  this ships a bug with a green checkmark.
- *"This timeout looks like a code problem, let me find the regression."* — Rule
  out `ENVIRONMENT_ISSUE` and `FLAKY_FAILURE` *first*. Do not invent a code
  regression to explain a substrate failure.
- *"It failed once, it's flaky."* — A real regression can look intermittent.
  Prefer `FLAKY_FAILURE` at `confidence: low` and recommend the re-run as the
  disambiguator; never assert `APPLICATION_BUG` on a single flaky-looking run
  either.
- *"I should just fix the code / write the missing test while I'm here."* — Out
  of scope. The skill classifies; a human (or a separate step) patches.
- *"I'm not sure, so I'll pick one confidently."* — A high-confidence wrong call
  trains the team to distrust the classifier. Pick the likelier verdict at
  `confidence: low` and name what a human would need to decide.

---

## Red Flags

Stop and reconsider if you notice any of these:

- You are about to write a patch or author a test. **Stop** — that is out of
  scope.
- You are about to run a test suite that applies real cloud infrastructure with
  no guaranteed teardown. **Stop** — fall back to INFERRED (see the IaC ladder).
- You assigned `APPLICATION_BUG` without ruling out flaky/environment causes.
- You assigned `FLAKY_FAILURE` or `ENVIRONMENT_ISSUE` purely from a diff, with no
  runtime evidence — these are generally not determinable statically.
- You are about to quote a token, secret, or real PHI/PII from a fixture into
  your report.
- Your confidence is `high` but you cannot point to a specific piece of evidence
  in the diff or the failure output.

---

## Verification

Before presenting the report, confirm:

- **Every failing test has exactly one verdict** — one classification per test,
  not per assertion. Passing tests are not listed.
- **Flaky and environment causes were ruled out before** the app-vs-test
  question — you worked the decision procedure in order.
- **No patch was authored and no new test was generated** — the output is
  verdicts and rationales only.
- **Each `APPLICATION_BUG`** says plainly the fix belongs in the app code, not
  the test.
- **Each `FLAKY_FAILURE`** names the re-run as the disambiguator.
- **Every `low` confidence** verdict names the specific disambiguator a human
  would need (re-run, product decision, etc.).
- **The mode is stated** — OBSERVED vs INFERRED — and if INFERRED, the report
  says why the suite could not be run.
- **No secrets or PHI/PII** appear verbatim anywhere in the report.

---

## Data Minimization (privacy posture)

This skill runs in repositories that may handle PHI/PII and may be subject to
FedRAMP/FISMA constraints. Treat what you read and surface as the smallest set
needed to reach a correct verdict.

What the classifier legitimately needs, and nothing more:

- **The failing-test signal** — test names, paths, assertion messages,
  expected-vs-actual diffs, stack traces, snapshot diffs.
- **The change-under-test diff** — `git diff "<base-ref>" HEAD`. This is
  required: the `APPLICATION_BUG` vs `TEST_BUG` decision hinges on whether the
  change *intended* to alter the asserted behavior, and you cannot judge that
  without seeing the change.

Rules:

- **Never echo secrets.** If a failure output or fixture contains a token, key,
  password, or connection string, redact it in your rationale
  (`Authorization: Bearer ***`), and treat its mere presence in a committed
  fixture as worth flagging.
- **Never reproduce PHI/PII verbatim.** If a test fixture appears to contain real
  personal/health data (names, SSNs, DOBs, member IDs, addresses), do not quote
  it in the report. Describe the failure abstractly and note that the fixture
  should be reviewed for synthetic-data compliance.
- **Do not pull in unrelated files.** Read only the test, the code paths the
  failure implicates, and the diff. Do not load the broader source tree, env
  files, or credential stores "for context."