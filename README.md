# elons-algorithm

A Claude Code skill that runs **Elon Musk's 5-step engineering algorithm** as a
gated, **rigid-order** audit of a target you choose (a feature, module, process,
config, or subsystem).

The order is the point:

1. Make the requirements less dumb (every requirement needs a *named* owner).
2. Delete the part or process (add back ~10% or you didn't delete enough).
3. Simplify / optimize — only what survived deletion.
4. Accelerate cycle time.
5. Automate — last.

It produces a full markdown report under [`reports/`](./reports/).

This repo is consumed as a **git submodule** at `.claude/skills/elons-algorithm/`
inside the parent project so Claude Code can discover it. See [`SKILL.md`](./SKILL.md)
for the full procedure.

# ad

[![Try Claude Code — free guest pass](https://img.shields.io/badge/Claude%20Code-free%20guest%20pass-D97757?logo=claude&logoColor=white)](https://claude.ai/referral/4Q8ajgGjHQ)

> This repo is built with [Claude Code](https://claude.com/claude-code). Grab a guest pass above to try it yourself.
