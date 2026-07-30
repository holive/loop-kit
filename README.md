# loop-kit

A tested recipe for making an AI work on its own until a job is done.

Two files. `LOOP.md` is the recipe. This file explains how to use it.

---

## What is a loop?

**Normal way you use AI today:**

You ask. It answers. You spot what's wrong. You ask again. Over and over.
You are doing the checking. If you stop, everything stops.

**A loop:**

You give the goal once. The AI does the work, checks its own result against something
real, fixes what failed, and goes again. It stops when the job passes or when it runs
out of tries.

You walk away. The work continues.

---

## The one rule

Before you build any loop, write down **the exact result that means FAIL.**

On paper. Before anything runs. Not "there's a test somewhere." The literal failing output.

The thing that produces it is called the **oracle**. It is what can say "no."

**Real oracles** — you can state the failure in advance:

```
npm test               fail = any line starting "FAIL"
go build ./...          fail = any output at all
grep -c "foo" file.go   fail = prints 0
this month's totals     fail = the two numbers differ
the definitions section fail = a capitalised term with no entry
```

Note the last two. No terminal involved. The rule isn't about code, it's about whether you
could write the failure down beforehand.

**Fake oracles** — you can't:

- "rate your answer 1 to 10"
- "be brutally honest"
- "does this look right?"
- "check it against the spec document"

That last one is the sneaky one. The spec existed first, the AI can't edit it, so it *feels*
like a real oracle. But the actual check is an AI reading two documents and forming an
opinion. You can't write down in advance what failure looks like, so it isn't one.

Fake oracles don't work. The AI gives itself an 8 and moves on. This is the single most
common way loops fail, and it's why most "loop" advice you'll read online is wrong.

**Can't write the failure down? Don't build a loop. Do the task by hand.**

That's not a loss. Doing it by hand is what you do today. A loop that blesses bad work is
worse than no loop, because you end up with opinions labelled as verified facts.

---

## How to use this (3 steps)

### Step 1 — ask for a filled-in spec

Open Claude Code in whatever project you're working on. Say:

```
read ~/projects/loop-kit/LOOP.md and fill in the spec for: <your task>
```

It reads your repo and comes back with a filled-in plan, including which commands it
found to use as the oracle.

### Step 2 — check the commands, then say yes

Look at the oracle commands it picked. Ask yourself: *can these actually fail?*

If yes, approve. Once approved, they are frozen. The AI is not allowed to change them
later. (See "Why freezing matters" below.)

### Step 3 — let it run, then read the notes

It works in rounds. Every round it writes what it tried and what the oracle said into a
notes file (called the **ledger**). When it stops, you read that file.

You don't have to watch it.

---

## Copy-paste starter

If you just want the smallest possible version, paste this into any AI chat:

```
GOAL: <what you want>

ORACLE: <what produces the verdict>. FAIL looks like: <write this now>.
        Paste the raw output. Do not score yourself against a rubric.

NOTES: before each new try, append what you tried and the raw output to
       notes.md. Read notes.md first so you don't repeat yourself.

STOP: when it passes, OR after 5 tries, OR when 2 tries in a row produce
      nothing new. Don't stop to ask me whether to continue.
      Then tell me what you did NOT manage to cover.
```

That's a real loop. The only part you must not fake is the FAIL line.

---

## Should you even use a loop?

Answer these four. **If you answer no to #2, stop — don't loop it.**

1. Does this task come up at least weekly?
2. Can you write down the exact result that means fail, before starting? *(the important one)*
3. Can the AI do the whole thing, without handing half back to you?
4. Is "done" a fact, not an opinion?

Got 2 or 3 yeses? You can still loop the part that is factual, and keep the judgment
part for yourself. That's normal and it's often the right answer.

Example: "is this claim true about the repo" is a fact -> loop it.
"is this worth telling my teammate" is judgment -> you decide.

---

## Why freezing the oracle matters

If the AI is allowed to change the oracle while it works, this happens:

```
try 1-4:  npm test           -> fails
try 5:    npm test -- auth   -> passes
```

It made the test smaller until it went green. Nothing got fixed, but it looks like
success.

This is not the AI cheating on purpose. Changing the test is simply the easiest way to make
the red go away, so it drifts there. Close the door and the problem disappears.

The most successful public example (Karpathy's `autoresearch`, 92k stars) does better than
just forbidding it. Training always runs for exactly 5 minutes, so there is no test to
shrink. The cheat isn't banned, it's designed out. Aim for that when you can.

---

## When it goes wrong, it looks like this

| What you see | What actually happened | Fix |
|---|---|---|
| "All done!" but nothing was checked | The oracle never ran | Demand the raw command output for every try |
| Runs forever, always "found something new" | It forgets rejected ideas and re-finds them | It must skip everything already in the notes, including rejected ones |
| Confidently wrong, over and over | The oracle was another AI opinion | Use a real command |
| Passes suspiciously fast | The oracle got changed | Freeze it, and check it wasn't edited |
| Stops after two tries asking if it should go on | You didn't tell it not to | Add "don't stop to ask me" |
| Huge bill, little output | No limit on tries | Always set a max number of tries |

---

## Build in this order

Don't skip ahead. This is how loops burn money overnight.

1. Do it **once by hand** and get it right.
2. Save those instructions to a file.
3. Add the oracle and the stop limit -> now it's a loop.
4. Only then put it on a schedule.

---

## Is it working?

One number matters: **how much of the output you keep.**

Not tokens spent. Not rounds run. If it gives you 10 things and you throw away 6, you
are doing the reviewing it was supposed to save you.

---

## Using it for something that isn't code

Fork it. Don't generalise it.

The examples in `LOOP.md` are deliberately specific: git, `gh api`, repo SHAs. When you need
a loop for invoices or contracts, copy that worked example and rewrite it for invoices. Do
not rewrite it to cover both.

This is how `autoresearch` spread. Its instruction file is ruthlessly narrow — nanochat,
`train.py`, one metric, five minutes — and yet there are four platform forks linked in its
own README. The reuse comes from being easy to copy, not from being abstract.

Every time you make the recipe broader, you make the FAIL rule easier to fake. That trade is
never worth it. Only the FAIL rule itself is meant to be universal.

---

## Files

| File | What it is |
|---|---|
| `README.md` | This. How to use it. |
| `LOOP.md` | The spec template, eight rules, failure modes, and a worked example to copy |

`LOOP.md` is more technical. You don't need to read it to start — Step 1 above points the AI
at it for you.

---

## Where this came from

Not invented here. Each rule in `LOOP.md` carries its evidence inline. The sources:

- **Karpathy's `autoresearch`** (2026, 92k stars) — the protected evaluator, and the better
  trick of making the oracle ungameable by construction rather than by instruction. Runs
  roughly 100 unattended experiments a night off a single markdown file and no engine.
- **Huang et al., ICLR 2024** — AI cannot reliably fix its own reasoning without outside
  feedback. This is why fake oracles don't work.
- **Same paper** — AI-debating-AI performs *worse* than simple voting at equal compute, which
  is why this recipe never tells you to have two AIs argue.
- **One measured run of this template** — 81 rows, 6 rounds, 71 unambiguous verdicts and 7 that
  needed a compound one. Source of the per-row pinning rule, the lens rotation numbers, the
  88% health metric, and the admission that three verdict values weren't enough.

Two ideas from earlier drafts (MAP-Elites diversity archives, UIST 2024 on criteria drift)
are still in `LOOP.md`, but under "not yet exercised" — nothing here has tested them.
