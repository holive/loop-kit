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

Before you build any loop, finish this sentence:

> "I will know it's wrong when I run `________` and see `________`."

That command is called the **oracle**. It is the thing that can say "no."

**Real oracles** (something outside the AI decides):

```
npm test              -> passes or fails
go build ./...         -> compiles or it doesn't
gh api repos/x/y       -> the file is there or it isn't
grep -c "foo" file.go  -> says 3, or says 0
```

**Fake oracles** (the AI decides about its own work):

- "rate your answer 1 to 10"
- "be brutally honest"
- "does this look right?"

Fake ones don't work. The AI gives itself an 8 and moves on. This is the single most
common way loops fail, and it's why most "loop" advice you'll read online is wrong.

**If you can't name a real command, don't build a loop. Just do the task by hand.**

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

ORACLE: run `<a real command>` and paste its exact output.
        That output decides pass or fail. Do not score yourself.

NOTES: before each new try, append what you tried and the command output
       to notes.md. Read notes.md first so you don't repeat yourself.

STOP: when the command passes, OR after 5 tries, OR when 2 tries in a row
      produce nothing new. Then tell me what you did NOT manage to cover.
```

That's a real loop. The only part you must not fake is the ORACLE line.

---

## Should you even use a loop?

Answer these four. **If you answer no to #2, stop — don't loop it.**

1. Does this task come up at least weekly?
2. Can a command automatically reject bad work? *(the important one)*
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

This is not the AI cheating on purpose. Changing the test is simply the easiest way to
make the red go away, so it drifts there. Close the door and the problem disappears.

The most successful public example of this pattern (Karpathy's `autoresearch`, ~92k
stars) works exactly this way: the AI could rewrite the whole model, but was blocked
from editing the file that scored it.

---

## When it goes wrong, it looks like this

| What you see | What actually happened | Fix |
|---|---|---|
| "All done!" but nothing was checked | The oracle never ran | Demand the raw command output for every try |
| Runs forever, always "found something new" | It forgets rejected ideas and re-finds them | It must skip everything already in the notes, including rejected ones |
| Confidently wrong, over and over | The oracle was another AI opinion | Use a real command |
| Passes suspiciously fast | The oracle got changed | Freeze it, and check it wasn't edited |
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

## Files

| File | What it is |
|---|---|
| `README.md` | This. How to use it. |
| `LOOP.md` | The full spec template, the rules, failure modes, and a worked example |

`LOOP.md` is longer and more technical. You don't need to read it to start — Step 1
above points the AI at it for you.

---

## Where this came from

Not invented here. Distilled from things that were tested or peer reviewed:

- **Karpathy's `autoresearch`** (2026) — the protected-evaluator idea. ~700 experiments,
  kept 20 real improvements, cut training time 11%.
- **Huang et al., ICLR 2024** — proved AI cannot reliably fix its own reasoning without
  outside feedback. This is why fake oracles don't work.
- **Same paper** — also found AI-debating-AI performs *worse* than simple voting, which
  is why this recipe doesn't tell you to have two AIs argue.
- **Mouret & Clune, MAP-Elites / Cully et al., Nature 2015** — why you keep a record of
  everything tried, not just the winners.
- **Shankar et al., UIST 2024** — your quality criteria change as you look at results,
  so the checklist is an output, not a constant.
