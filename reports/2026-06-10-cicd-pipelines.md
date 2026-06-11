# Elon's Algorithm — CI/CD pipelines of robotframework-chat

Date: 2026-06-10 · Auditor: Claude Code (session goal set by tkarcheski)

## Step 0 — Scope

- **Target:** the CI/CD surface — 8 GitHub workflows, `.gitlab-ci.yml` +
  `ci/common.yml`, 14 `ci/*.sh` scripts, `.pre-commit-config.yaml`, and the
  Makefile targets CI calls. ~2,030 lines.
- **Out of scope:** the test suites themselves, the monitoring system's policy
  content (just merged, audited separately), local dev tooling.
- **Claimed purpose:** gate quality on every change, publish releases
  (PyPI/GHCR), keep the dual-remote topology (dev on GitHub, live on GitLab)
  in sync, and run the multi-hour model-evaluation pipeline on the GitLab
  runner fleet.

## Inventory (from the actual files, 2026-06-10)

GitHub workflows: auto-assign, claude-checkpoints, docker-publish,
mirror-staging-to-gitlab, pypi-publish, robot-tests, sync-from-gitlab,
sync-to-gitlab. GitLab: lint, run-local-models×5 nodes, repo-metrics,
pipeline-summary, deploy-superset, build-check, opencode-review.
Scripts: lint, test, generate, report, pipeline_report, sync, deploy,
ensure_node, send_results, backup_push, release, review, local_review,
audit_markdown (+ AGENTS.md, common.yml).

## Step 1 — Make the requirements less dumb

All requirements trace to one nameable person: **tkarcheski** (solo-owner
repo). That satisfies the named-owner rule but removes the usual excuse —
every "keep" below is his to defend, and several don't survive first
principles.

| Requirement | Named owner | Justification | Verdict |
|---|---|---|---|
| Every PR must pass lint + dryrun before merge | tkarcheski | quality gate | **keep — but it is currently FICTION** (see below) |
| Releases publish to PyPI + GHCR on `v*` tags | tkarcheski | distribution | keep |
| GitHub `claude-code-staging` mirrors to GitLab | tkarcheski | live deployment runs from GitLab | keep |
| `main` syncs **bidirectionally** GitHub↔GitLab every 15 min, force-push both ways | tkarcheski | "keep remotes equal" | **dumb — soften.** Two force-push loops with no lock is a data-loss machine, not a requirement. One direction must own `main`. |
| CI must run the full model fleet (5 node jobs) per pipeline | tkarcheski | continuous model coverage | soften — schedule/manual, not per-pipeline; the fleet is a lab instrument, not a merge gate |
| Pipeline YAML must be machine-generated (`ci-generate`, 3 modes) | tkarcheski | node fleet changes | **dumb — drop.** The fleet is 5 named nodes in a static file; a generator with discover/dynamic/regular modes is automation of a problem that no longer exists (Step-5 work done before Step 2). |
| MR review by external LLM (opencode/Kimi) on label | tkarcheski | second-opinion review | soften — it duplicates Codex (auto-assign) and the new Claude heartbeat; three reviewers is two too many |
| Results rsync to a results server | tkarcheski | off-box archival | **dumb — drop.** `RESULTS_SERVER_*` is unset everywhere; the `results/` LFS submodule already is the archival path. |
| Node ≥18 must be ensurable for Playwright dashboards | tkarcheski | dashboard CI job | **dumb — drop.** The dashboard-playwright job no longer exists. |

**Finding 1 (the headline):** `robot-tests.yml` triggers on `push`/`pull_request`
to **`main`** only — but development happens on `claude-code-staging` and PRs
target it (CLAUDE.md: "Always rebase onto claude-code-staging, not main").
Observed on live PRs #391–#395: the only checks that ran were auto-assign and
checkpoint. **The repo's quality gate does not run on the branch it is supposed
to gate.** Every "CI is green" claim on a staging PR is vacuously true.

## Step 2 — Delete

| Part / step | What breaks if deleted | Delete? | Add-back |
|---|---|---|---|
| `ci/test.sh` (120 ln) | nothing — zero callers | **yes** | none |
| `ci/sync.sh` (36 ln) | nothing — superseded by workflows | **yes** | none |
| `ci/ensure_node.sh` (71 ln) | nothing — job it served is gone | **yes** | none |
| `ci/send_results.sh` + `make send-results` (66 ln) | nothing — env never set, results/ submodule is the archive | **yes** | none |
| `ci/generate.sh` + `scripts/generate_pipeline.py` + discover scripts (75+ ln sh, more py) | regenerating gitlab YAML — which is hand-maintained anyway | **yes** | maybe a 20-line static template comment |
| `sync-from-gitlab.yml` (88 ln, 15-min cron, force-push to GitHub) | GitLab-side commits to `main` stop flowing back automatically | **yes** — make GitHub the single writer of `main`; GitLab becomes read-only mirror. If GitLab-side hotfixes are real, add back a manual-dispatch pull. | partial (manual dispatch) |
| `sync-to-gitlab.yml` `--all --force` default | nothing if narrowed | narrow to `main` + tags (keep workflow) | — |
| `opencode-review` job + `ci/review.sh` + `ci/local_review.sh` (497 ln) | a third LLM reviewer | **yes** — Codex + Claude heartbeat already review every PR | none |
| `repo-metrics` vs `pipeline-summary` jobs | one of two overlapping metric reports | merge into one | — |
| `ci/audit_markdown.sh` (261 ln) | manual-only markdown audit | keep (manual tool, costs nothing in CI) | — |
| auto-assign, claude-checkpoints, docker-publish, pypi-publish, mirror-staging, lint, build-check, deploy-superset, backup_push, lint.sh, report.sh, pipeline_report.sh, deploy.sh, release.sh | active, single-purpose, recently exercised | no | — |

**Deletion accounting:** ~853 lines of shell + 2 workflows + 1 GitLab job + the
pipeline-generator Python proposed for deletion ≈ **40% of the CI surface**.
Expected add-back: a manual-dispatch GitLab→GitHub pull and a static-template
note ≈ 5–8% of the deleted volume — close enough to the ~10% target to say the
deletion is aggressive but not reckless.

## Step 3 — Simplify (survivors only)

1. **Fix the headline:** point `robot-tests.yml` at `claude-code-staging`
   (`push` + `pull_request`) so lint/dryrun actually gate PRs. One-line filter
   change; highest value-per-line in this audit.
2. **One sync direction per branch:** GitHub owns `main` and
   `claude-code-staging`; GitLab receives both (sync-to-gitlab narrowed +
   mirror-staging kept). Delete the 15-min reverse cron (Step 2).
3. Collapse `repo-metrics` + `pipeline-summary` into one job/script.
4. `.gitlab-ci.yml`: with the generator gone, inline the 5 node jobs as 5
   explicit jobs extending one template (they already are — delete the
   generator, keep the YAML).

## Step 4 — Accelerate (survivors only)

- PR feedback latency: lint job currently only exists GitLab-side for MRs;
  after fix #1, GitHub PRs get lint+dryrun in ~2 min on ubuntu-latest. Cache
  `uv` (`astral-sh/setup-uv` with cache) to cut ~40s.
- `robot-tests` (self-hosted, 3 suites) should not block PR feedback — keep it
  push-only (already is) so the PR loop stays at the 2-min lint/dryrun tier.
- Fleet runs: already detachable (`run-local-models` + autopilot); keep them
  out of the merge path entirely.

## Step 5 — Automate (only now)

- Already-automated and earned: tag→PyPI/GHCR publishing, staging mirror,
  checkpoint labelling, heartbeat sweeps. No new automation is justified until
  the Step-2 deletions land — in particular, do NOT automate fleet-run
  scheduling into GitLab CI per-pipeline; the hourly heartbeat + autopilot
  already covers cadence.
- Candidate once stable: auto-file GitHub issues from `run-local-models`
  failures (owner already requested this; it belongs *after* the gate fix so
  issues reflect real regressions, not pipeline rot).

## Verdict

- **Delete (~40% of surface):** `ci/test.sh`, `ci/sync.sh`,
  `ci/ensure_node.sh`, `ci/send_results.sh`+target, the pipeline generator
  (script+python+modes), `sync-from-gitlab.yml` (replace with manual
  dispatch), the opencode review stack (497 ln), one of the two metrics jobs.
- **Fix first:** `robot-tests.yml` branch filter — the quality gate must run
  where development actually happens (`claude-code-staging`).
- **Survives:** publishing workflows, staging mirror, narrowed sync-to-gitlab,
  lint (both sides), build-check, deploy-superset, backup_push, checkpoints,
  auto-assign.
- **Open questions for tkarcheski:**
  1. Does anything ever legitimately commit to `main` on GitLab? (If no →
     delete sync-from-gitlab outright, not even manual dispatch.)
  2. Is opencode/Kimi review still wanted now that Codex + Claude heartbeat
     both review every PR?
  3. Is the pipeline generator's "discover" mode used by any human workflow,
     or was it speculative?

## Deletion accounting (the ~10% check)

Proposed deletions ≈ 853 shell lines + ~400 python (generator+discover) + 2
workflows + 1 job. Expected add-back ≈ 60–90 lines (manual-dispatch pull
workflow + template notes) ≈ **7%** — at target. If the answer to open
question 1 is "no", add-back drops below 5%, meaning we should look for the
next deletion candidate (likeliest: `ci/audit_markdown.sh` if unused by June
audit).
