---
name: elons-algorithm
description: >-
  Runs Elon Musk's 5-step engineering algorithm as a gated, rigid-order audit of
  a target the user chooses — a feature, module, process, config, or subsystem.
  Challenges every requirement (and demands a named owner for each), tries hard
  to delete each part, and only then simplifies, accelerates, and automates — in
  that exact order. Produces a full markdown report under the skill's reports/
  directory. The whole point is to stop teams optimizing or automating things
  that should not exist.
when_to_use: >-
  Trigger when the user says "run Elon's algorithm", "apply the 5-step
  algorithm", "make the requirements less dumb", "what can we delete here", "audit
  this for over-engineering", "challenge these requirements", or asks for a
  first-principles / delete-and-simplify pass over a feature, module, pipeline,
  config, process, or subsystem. Also reach for it when a design is accreting
  parts, steps, or requirements that nobody can justify by name.
---

# Elon's Algorithm

A disciplined, **rigid-order** audit based on Elon Musk's 5-step engineering
process. The order is the whole point — the most common and most expensive
mistakes happen when people optimize, accelerate, or automate something that
should have been questioned or deleted first.

> "Possibly the most common error of a smart engineer is to optimize a thing
> that should not exist."

## The five steps (and you MUST run them in this order)

1. **Make the requirements less dumb.** Every requirement is guilty until proven
   innocent — *especially* requirements from smart, senior, or well-liked people,
   because those get questioned the least. The only truly fixed requirements are
   the laws of physics; everything else is a recommendation. **Every requirement
   must carry the name of a real person**, never a department ("legal requires
   it", "the platform team said so" — not acceptable). If you can't name who
   owns a requirement, you can't go ask them *why*, and an unchallengeable
   requirement is exactly the kind that's dumb.

2. **Delete the part or the process step.** Try *hard* to delete each thing that
   survived step 1. The bias everywhere is "leave it in, just in case." Fight it.
   **Accounting rule: if you don't later add back at least ~10% of what you
   deleted, you didn't delete enough.** Deleting too little is the default
   failure; deleting too much (and adding a bit back) is the target.

3. **Simplify or optimize** — but only what survived steps 1 and 2. Do not
   optimize a part you should have deleted. This is where smart engineers burn
   the most effort for the least value, because optimizing feels productive.

4. **Accelerate cycle time** — only after 1–3. "You're moving too slowly, go
   faster" — *but* if you're still digging your own grave, don't dig it faster.
   Speeding up an unquestioned, bloated process just bakes the waste in.

5. **Automate** — last. The classic, expensive failure (Tesla's Nevada line) was
   running this list in reverse: automate → accelerate → simplify → delete. They
   had to rip the automation back out. Automating a process you haven't yet
   questioned, deleted, and simplified means automating waste.

### Why the order is non-negotiable

Each step is cheaper and higher-leverage than the one after it. Deleting a part
costs nothing to maintain, test, optimize, or automate. So you spend your
question-and-delete budget *first*, on everything, before you spend a single hour
optimizing or automating what remains. Running the list out of order is the
single error this skill exists to prevent.

## How to run this skill

### Step 0 — Scope the review (ask the user first)

Before any analysis, ask the user what is under review. Do not assume. Use a
short multiple-choice question covering at least:

- **What is the target?** A feature / a module or file / a process or workflow /
  a config or schema / a whole subsystem.
- **Where are its boundaries?** What's explicitly in, and what's out of scope.
- **Why does it exist / what is "done"?** The claimed purpose — you'll test this
  in step 1.

Pin these down in writing before continuing. A fuzzy scope produces a fuzzy
report.

### Build the inventory

For the scoped target, enumerate two lists by reading the actual code/config/docs
(not from memory):

- **Requirements & constraints** — every "must", "should", limit, invariant,
  field, flag, validation, or assumption the target is built to satisfy.
- **Parts & process steps** — every component, function, file, dependency,
  config knob, manual step, or stage in the pipeline.

### Run the gates in order, writing one report section per step

You may **not** write the Step 3, 4, or 5 sections until Steps 1 and 2 are
complete. You may **not** recommend simplifying, accelerating, or automating a
part that Step 2 marked for deletion. If you catch yourself doing either, stop
and go back.

**Step 1 — Requirements.** For each requirement produce a row:

| Requirement | Named owner | Justification | Verdict |
|---|---|---|---|
| ... | *person, not a dept* | why it (allegedly) exists | keep / soften / **dumb — drop** |

If a requirement has no nameable owner, that is itself a finding. Challenge each
one against first principles and the laws-of-physics test. List the dumb ones
loudly.

**Step 2 — Delete.** For each surviving part/step:

| Part / step | What breaks if deleted | Can we delete? | Add-back? |
|---|---|---|---|
| ... | concrete consequence | yes / no / try it | none / partial |

Track the deletion accounting: total proposed for deletion, and how much you
expect to add back. If add-back is far below ~10%, push harder — you're being
timid.

**Step 3 — Simplify/optimize.** Only survivors of Step 2. Concrete simplifications
and optimizations, each tied to a survivor.

**Step 4 — Accelerate.** Cycle-time wins for what remains: feedback loops, build
times, review/approval latency, iteration speed.

**Step 5 — Automate.** Only now. What is stable, simplified, and worth automating
— and explicitly, what is *not* yet ready to automate and why.

### Write the report

Write the full report to:

```
reports/YYYY-MM-DD-<short-target-slug>.md
```

inside this skill's directory (it's a submodule — committing the report commits
it to the skill repo, not the parent project). Use the date from the session
context, not a guessed one. Structure the file in the rigid step order with a
final summary:

- **Verdict:** what to delete, what survives, what to simplify/accelerate/automate.
- **Deletion accounting:** proposed vs. expected add-back (the ~10% check).
- **Open questions / owners to chase:** the unnamed requirements and unjustified
  parts that need a human to answer "why".

## Red flags — stop if you catch yourself thinking any of these

| Thought | Reality |
|---|---|
| "This requirement is obviously needed." | Especially then. Name its owner and ask why. |
| "Let's keep it in case we need it." | That bias is exactly what Step 2 deletes. |
| "Let me just optimize this part real quick." | Did it survive Step 2? If unsure, you skipped a step. |
| "We should automate this manual step." | Not until it's questioned, deleted-tested, and simplified. |
| "A smart/senior person set this requirement." | Smart people are wrong sometimes too, and get questioned least. |
| "I deleted hardly anything — looks lean already." | If add-back is under ~10%, you didn't delete enough. |
| "The legal/platform/security *team* requires it." | A team is not a name. Find the person. |

## Corollaries Musk attaches to the algorithm

- **Technical managers must have hands-on experience.** Don't trust a
  requirement or design from someone who can't do the work themselves.
- **Comradeship is dangerous** to engineering rigor — it makes people reluctant
  to challenge each other's work. Challenge the work, not the person.
- **The team that owns a thing should be slightly *too eager* to delete it.** If
  deletions are never wrong, deletion isn't aggressive enough.
