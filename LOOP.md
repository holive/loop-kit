# Loop template

A reusable spec for agent loops. Copy it, fill it in, and **specialize it** for the job.
Do not make this file more general to cover a new domain. Fork it and rewrite the body.
Only the gate in section 0 is domain-neutral, and it is the only part that must stay that way.

Provenance: Karpathy's `autoresearch` (92k stars, ~100 unattended experiments per night),
Huang et al. ICLR 2024 (self-correction fails without external feedback), plus one measured
run of this template, 65 claims over 5 rounds, cited inline where it earned a rule.

---

## 0. Gate — do not build a loop unless all four hold

1. The task repeats, at least weekly. A one-off is better served by one good prompt.
2. **You can write down, in advance, the exact result that means FAIL.** Not "a command
   exists." Not "there's a source of truth." The literal failing output, on paper, before
   anything runs. If you cannot, you do not have an oracle and the loop will only spin.
3. The agent can do the work end to end, not hand half back.
4. "Done" is a fact, not taste.

Item 2 is the whole gate. It is deliberately hard to pass, because a loop that blesses bad
work is worse than no loop: you get opinions labelled as verified facts. Failing item 2 and
doing the task by hand costs you nothing you had. Passing it falsely costs you trust in every
row of the output.

Pass only 2-3? Loop the part that passes item 2, keep the rest manual, and say which is which.

---

## 1. Spec — fill this in

```
LOOP:      <name>
GOAL:      <one sentence, objectively checkable>
NOT GOAL:  <what this loop must not drift into>

ORACLE                      # the gate. required. build this FIRST.
  verdict from:    <what produces the verdict>
  FAIL looks like: <the literal output that means fail — write this before running>
  verdict values:  <allowed set. include partial, and require it to name which part>
  evidence path:   <what it may read to decide, independently of the worker>
  PROTECTED:       <files, prompts, commands the worker must never modify>
  pinned at:       <SHA / version / timestamp — a recorded value, never "now">

WORKER
  may edit:      <explicit allowlist>
  may not edit:  the ORACLE, this spec, the LEDGER's history

LEDGER                      # external state. survives session death and any revert.
  path:          <file, outside anything the work itself can roll back>
  records:       round, seen[], accepted[], rejected[+reason], oracle pin, spend
  dedup key:     <what makes two candidates "the same">

ROUND
  1. read LEDGER; do not re-derive anything already in seen[]
  2. rotate the lens from last round: <lens A, lens B, lens C, ...>
  3. produce candidates
  4. dedup against seen[]        # NOT against accepted[]
  5. run ORACLE on each fresh candidate; capture its raw output to a file
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
2. Extract the instructions into a reusable file.
3. Wrap it in the loop: add the oracle gate and the stop conditions.
4. Only then put it on a schedule.

---

## 3. Invariants

Eight rules. Each carries the symptom you get without it and the evidence for it. Nothing
here is a preference.

**1. The worker cannot touch the oracle.**
Symptom: everything passes suspiciously fast. The agent made the test easier, not the work better.
Evidence: `autoresearch` marks `prepare.py` read-only and names `evaluate_bpb` as ground truth.

**2. Make the oracle ungameable by construction, not by instruction.**
Symptom: the test quietly narrows until it goes green.
Evidence: `autoresearch` fixes training at 5 minutes wall clock, so there is no test to shrink,
and picks a vocab-size-independent metric so architectural changes compare fairly.

**3. Prefer an oracle that needs no interpretation.** One number, one exact string, one match.
If a model must read output and form a view, that row is judgment. Label it judgment.
Symptom: confident convergence on a wrong answer.
Evidence: measured run, 51 of 65 verdicts mechanical, 14 needed hedging. The hedged ones are
exactly the ones a stranger cannot re-check.

**4. State lives in a file, outside the model and outside anything a revert can erase.**
Symptom: round N repeats round 1; context regrows every round and cost blows out.
Evidence: `autoresearch` keeps `results.tsv` untracked so the `git reset` that discards a failed
experiment cannot erase the record of it. Measured run carried 66 rows across 5 rounds sharing
no context.

**5. Pin what you judged against, on every row.**
Symptom: two correct rounds disagree and you cannot tell which to believe.
Evidence: measured run, four claims flipped truth value mid-loop because an upstream repo moved.

**6. Rotate the lens on purpose.** Framing variance is the engine of coverage, not noise.
Symptom: high volume, narrow coverage. Every round finds the same class of thing.
Evidence: measured run per lens — premise-truth 16, staleness 14, attribution 18, second-order
16, what-is-missing 5. Each found a class the others missed.

**7. Two stop conditions minimum, success and a hard cap. Then log what you did not cover.**
Symptom: silent truncation, which reads as completeness.
Evidence: measured run ended on the hard cap with the completeness lens still returning fresh
items, and said so.

**8. Never pause to ask whether to continue.**
Symptom: it stops after two rounds for reassurance, which is a conversation with extra steps.
Evidence: `autoresearch` states it flatly — do not ask "should I keep going?", the human may be
asleep, run until interrupted.

Two mechanical notes. Raw oracle output goes to a file and is read back selectively, never
streamed into context (`autoresearch`: redirect to `run.log`, do not `tee`). And a verdict may
be partial; force it into one word and you lose the part that was actually unresolved.

### Not yet exercised

Carried from earlier drafts, untested by any run here. Untested is not wrong, but do not treat
these as load-bearing until something proves them.

- Dedup against `seen[]`, never `accepted[]`, or rejected candidates return every round.
- No model-vs-model debate as the quality mechanism; at matched compute it underperforms plain
  majority voting (Huang et al.).
- If a round evaluates earlier conclusions, write its own findings to disk *first*, then read
  the prior ones, so agreement is costly.
- Re-derive grading criteria as you see outputs rather than freezing them.
- Track cost per accepted change, not tokens or rounds.

---

## 4. Worked instance — verifying review claims against a repo

This is the thing to copy. Rewrite it for your domain. Do not abstract it.

```
LOOP:      pr-claim-verify
GOAL:      every factual claim in the review is confirmed or refuted against repo state
NOT GOAL:  deciding which findings are worth posting (human judgment, stays manual)

ORACLE
  verdict from:    gh api / git show / git grep / file contents
  FAIL looks like: the grep returns no match, or returns a string other than the one the
                   claim asserts. Written down before round 1.
  verdict values:  confirmed | refuted | cannot-establish | partial(name which half)
  evidence path:   the repos themselves, read directly
  PROTECTED:       the claim list, this spec, the ledger's history
  pinned at:       one SHA per repo, in the ledger header AND on every row

WORKER
  may edit:      the candidate-claims file, the working notes
  may not edit:  the claim list under verification, the oracle commands, past ledger rows

LEDGER
  path:          .loop/claims.md          # outside the repo under audit
  records:       claim, verdict, raw output, repo SHAs, round, load-bearing yes/no
  dedup key:     normalized claim text + the file:line it asserts about

ROUND
  1. read ledger; skip claims already resolved at the current pin
  2. rotate lens: [premise-truth, staleness, second-order-effects,
                   attribution, what-is-missing]
  3. produce or re-check claims under that lens
  4. dedup against seen[]
  5. resolve each via the oracle; paste the raw output
  6. append with SHAs

STOP
  success:   every claim resolved, and the what-is-missing lens returns nothing fresh
  dry:       2 consecutive rounds with 0 fresh claims
  hard cap:  5 rounds
  on stop:   confirmed / refuted / cannot-establish / partial, each with evidence and pin;
             list claims that moved because a repo moved; state what was not covered
```

Note the split. The oracle answers "is this true of the repo," which is a fact. Whether a true
finding is worth raising stays with you. That is gate item 4 failing honestly, handled by
scoping the loop rather than pretending.

---

## 5. Minimal version (no tooling)

The only part you must not fake is the oracle.

```
GOAL: <...>
ORACLE: <what produces the verdict>. FAIL looks like: <write it now, before running>.
        Paste raw output. Do not score your own work against a rubric.
LEDGER: append each attempt and the raw output to <file> before continuing.
ROUND: read the file, try something not already in it, run the oracle, append.
STOP: oracle passes, or 5 rounds, or nothing new twice in a row.
      Do not stop to ask whether to continue. Then report what you did not cover.
```
