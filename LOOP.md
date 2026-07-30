# Loop template

A reusable spec for agent loops. Fill in the blanks per context. The invariants in section 3
do not change — each one is there because skipping it has a known failure mode.

Provenance: distilled 2026-07-30 from Karpathy's `autoresearch` (protected evaluator),
Huang et al. ICLR 2024 (self-correction fails without external feedback), Mouret & Clune
MAP-Elites / Cully et al. Nature 2015 (diversity archive), Shankar et al. UIST 2024
(criteria drift), and one live case where ground truth moved mid-review.

---

## 0. Gate — do not build a loop unless all four hold

1. The task repeats, at least weekly. A one-off is better served by one good prompt.
2. Something can **automatically reject** bad output: a test, build, type check, linter,
   query, or hard rule. If nothing can fail the work for you, the loop only spins.
3. The agent can do the work end to end, not hand half back.
4. "Done" is objective, not taste.

Miss one and keep it manual. Pass only 2-3? Loop the objective part, keep the judgment part
manual, and say which is which in the spec.

---

## 1. Spec — fill this in

```
LOOP:      <name>
GOAL:      <one sentence, objectively checkable>
NOT GOAL:  <what this loop must not drift into>

ORACLE                      # the gate. required. build this FIRST.
  verdict from:  <command(s) or query that produce pass/fail or a score>
  evidence path: <what it may read to decide, independently of the worker>
  PROTECTED:     <files, prompts, commands the worker must never modify>
  pinned at:     <SHA / version / timestamp — a recorded value, never "now">

WORKER
  may edit:      <explicit allowlist>
  may not edit:  the ORACLE, this spec, the LEDGER's history

LEDGER                      # external state. survives session death.
  path:          <file>
  records:       round, seen[], accepted[], rejected[+reason], oracle pin, spend
  dedup key:     <what makes two candidates "the same">

ROUND
  1. read LEDGER; do not re-derive anything already in seen[]
  2. rotate the lens from last round: <lens A, lens B, lens C, ...>
  3. produce candidates
  4. dedup against seen[]        # NOT against accepted[]
  5. run ORACLE on each fresh candidate; capture its raw output
  6. append everything to LEDGER, including rejections and why

STOP  (need at least success + hard cap)
  success:   <objective condition>
  dry:       K consecutive rounds with 0 fresh candidates (K = 2)
  hard cap:  <max rounds> or <max spend>, whichever hits first
  on stop:   report accepted[], rejected[] with reasons, oracle pin,
             and explicitly what was NOT covered
```

---

## 2. Build order

Do not skip ahead. Scheduling something unproven by hand is how loops burn money overnight.

1. Get **one manual run** reliable.
2. Extract the instructions into a reusable file (skill/prompt).
3. Wrap it in the loop: add the oracle gate and stop conditions.
4. Only then put it on a schedule.

---

## 3. Invariants

**The worker cannot touch the oracle.** Enforce by file permission, allowlist, or separate
process. Without this the agent makes the test easier instead of the work better. This is the
single highest-value rule; `autoresearch` enforces it by forbidding edits to the eval file.

**Prefer a non-LLM oracle.** A command, a query, a diff, a build. If the oracle must be a
model, give it an **independent evidence path** — its own tool calls — never the worker's
summary of its own work. Self-grading against a rubric the grader also interprets is not a
gate.

**Do not use peer debate as the quality mechanism.** At matched compute it underperforms
plain majority voting over independent attempts. Maker/checker separation is worth it only
when the checker has its own evidence path.

**Dedup against `seen[]`, never against `accepted[]`.** Dedup against accepted and every
rejected candidate returns each round; the loop runs forever looking busy.

**Two stop conditions minimum: success and a hard cap.** Dry-round detection is an
optimization on top, not a replacement. The loop must terminate even when not converging.

**Pin the environment and record the pin.** Every round writes the SHA/version it judged
against. Otherwise two correct rounds can disagree and you cannot tell why.

**Rotate the lens on purpose.** Framing variance is the engine of coverage, not noise to
suppress. Keep the diversity; make it accumulate in the ledger instead of scattering.

**Commit before comparing.** If a round evaluates earlier conclusions, it writes its own
findings to disk *first*, then reads the prior ones. Agreement must be costly or you get
anchoring dressed as verification.

**Re-derive criteria; do not freeze them.** Grading criteria shift as you see outputs. Treat
the rubric as an output of the loop, not a constant.

**Log what you did not cover.** Any cap, sample, or top-N must be stated. Silent truncation
reads as completeness.

**Track cost per accepted change.** Not tokens, not rounds. If you discard most output you
are doing the review the loop was meant to save.

---

## 4. Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Worker declares done, nothing verified | Oracle never ran ("Ralph Wiggum" early exit) | Require raw oracle output per candidate; no artifact means not done |
| Runs forever, always "new" findings | Dedup against `accepted[]` | Dedup against `seen[]` |
| Converges confidently on wrong answer | Oracle is a model with no independent evidence | Give the oracle its own tools, or replace with a command |
| Everything passes suspiciously fast | Oracle was mutated, or pin drifted | Verify PROTECTED integrity each round; diff the pin |
| Cost blowout with little output | No hard cap; context regrown every round | Hard cap; carry a compact ledger, not full history |
| Round N repeats round 1 | Ledger not read, or lens not rotated | Read `seen[]` first; rotate lens explicitly |

---

## 5. Worked instantiation — verifying review claims against a repo

```
LOOP:      pr-claim-verify
GOAL:      every factual claim in the review is confirmed or refuted against repo state
NOT GOAL:  deciding which findings are worth posting (human judgment, stays manual)

ORACLE
  verdict from:  gh api / git show / git grep / file contents
  evidence path: the repos themselves, read directly
  PROTECTED:     the claim list, this spec, the ledger's history
  pinned at:     one SHA per repo, recorded in the ledger header

WORKER
  may edit:      the candidate-claims file, the working notes
  may not edit:  the claim list under verification, the oracle commands, past ledger rows

LEDGER
  path:          .loop/claims.md
  records:       claim, verdict (confirmed|refuted|cannot-establish), evidence,
                 repo SHAs, round, whether load-bearing
  dedup key:     normalized claim text + file:line it asserts about

ROUND
  1. read ledger; skip claims already resolved at the current pin
  2. rotate lens: [premise-truth, staleness, second-order effects,
                   attribution, what-is-missing]
  3. produce or re-check claims under that lens
  4. dedup against seen[]
  5. resolve each via the oracle; paste the raw command output
  6. append with SHAs

STOP
  success:   every claim resolved, and the completeness lens returns nothing new
  dry:       2 consecutive rounds with 0 fresh claims
  hard cap:  5 rounds
  on stop:   confirmed / refuted / cannot-establish, each with evidence and the pin;
             list claims that moved because the repo moved
```

Note the split: the oracle answers "is this true of the repo," which is objective. Whether a
true finding is worth raising stays with you — that is gate item 4 failing honestly, handled
by scoping the loop rather than pretending.

---

## 6. Minimal version (no tooling)

If you want the shape without infrastructure, the only part you must not fake is the oracle.

```
GOAL: <...>
ORACLE: run `<real command>` and paste its raw output. That output is the verdict.
        Do not score your own work against a rubric.
LEDGER: append each attempt and the command output to <file> before continuing.
ROUND: read the file, try something not already in it, run the oracle, append.
STOP: oracle passes, or 5 rounds, or nothing new twice in a row.
      Then report what you did not cover.
```
