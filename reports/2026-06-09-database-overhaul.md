# Elon's Algorithm audit — robotframework-chat database overhaul

**Date:** 2026-06-09
**Target:** The full persistence layer — all three databases (Test Results,
Agentic Harness, Agent Workflow), the `models`/`coverage_reports` tables, the
dual SQLite/SQLAlchemy backends, the migration history, and the Superset read
path.
**Boundary (per scope intake):** All 3 DBs, consolidation mandate. Perf-schema
shape to be *decided during the audit*. Deliverable: this report + a redesign
blueprint (no schema code yet).
**Goal context (owner):** "Get back to the basics of Robot Framework; better
alignment with vLLM to see more performance as we integrate AI hardware in the
loop; optimize the Superset database for simple reviews first."

> **Single-owner caveat.** This is a solo repo: virtually every requirement
> traces to **tkarcheski**, sometimes via **Issue #350** (agentic-harness
> linkage). With no second person to triangulate against, the "name the owner"
> rule collapses to one name — so the real filter here is the *laws-of-physics
> test* ("would the project break without this?"), and every "ask why" below is
> a question for tkarcheski. They are listed in § Open questions.

---

## Step 1 — Make the requirements less dumb

Every requirement the persistence layer is currently built to satisfy, challenged.

| # | Requirement (as built) | Owner | Justification (alleged) | Verdict |
|---|---|---|---|---|
| R1 | Record RF outcomes: suite → test → pass/fail/score | tkarcheski | The entire point of the project (benchmark LLMs via Robot Framework). | **KEEP** — this is the laws-of-physics core. |
| R2 | Persist each run's full `output.xml` as a gzip BLOB in the DB (`test_run_artifacts.output_xml_gz`) | tkarcheski | Drill-down / reproducibility. | **SOFTEN** — the `results/` submodule (LFS) *already* stores outputs. A BLOB in Postgres duplicates that and bloats the review DB. Replace with a pointer. |
| R3 | Per-result Q/A archive (question/expected/actual/grading_reason/thinking_text) | tkarcheski | LLM-graded (tier 2–3) review. | **KEEP, audit fill-rate** — real for graded tiers; confirm how many rows are non-empty before sizing. |
| R4 | **Three separate databases** (results, harness, workflow), each with its own connection + backend pair | tkarcheski / #350 | Organic growth; never a stated requirement. | **DUMB** — one engine can hold one schema. Separation buys nothing here and costs cross-DB soft-FKs (no integrity), 3× backend code, and zero Superset visibility for 2 of the 3. |
| R5 | Agentic harness tracking: `agentic_harnesses/plugins/skills/metrics` (4 tables) | #350 / tkarcheski | Know which Claude Code/Codex harness produced a result. | **QUESTION → mostly DROP** — not in Superset, barely populated. `test_runs` *already* carries `session_id` + `model_harness`. 4 tables to record a provenance string is over-built for current reality. |
| R6 | Agent-workflow logging: `agent_workflows/interactions/tool_calls/results` (4 tables) | tkarcheski | Debug agent reasoning/tool use. | **QUESTION → SPLIT OUT** — this is *developer agent-introspection telemetry*, not RF benchmarking. One writer (`agent_workflow_listener.py`), invisible to Superset. Doesn't belong in the metrics DB at all. |
| R7 | `agentic_metrics` EAV (`metric_key` TEXT + `metric_value` REAL) | tkarcheski | "Flexible" KPI capture. | **DUMB** — free-form keys, no validation, typo-prone, not queried by Superset. Classic premature generality. |
| R8 | `models` metadata table (quantization, size_gb, family, …) | tkarcheski | Model leaderboard. | **KEEP THE INTENT, DELETE THE SHAPE** — has an `upsert_model` path but no producer fills the metadata columns. Rebuild as part of the perf layer. |
| R9 | `coverage_reports` table + 4 Superset charts | tkarcheski | Code-coverage dashboard. | **DUMB / zombie** — Superset *reads* it (bootstrap_dashboards.py:449–475) but **nothing writes it**. Permanent empty charts that look like real ones. |
| R10 | ~25 `ALTER TABLE … DROP COLUMN` migrations carried in `_SQLITE_MIGRATIONS` | tkarcheski | Backward-compatible upgrades. | **QUESTION** — this is a *benchmark* DB, rebuildable via `result_importer.py` from the `results/` submodule. Backward-compat migrations may be unnecessary; a clean v2 baseline + re-import could replace the whole chain. |
| R11 | Dual hand-written backends per DB (`_SQLiteBackend` + `_SQLAlchemyBackend`) — 6 classes | tkarcheski | SQLite for local/dev, Postgres for live/Superset. | **SOFTEN** — the *two engines* are justified (dev vs live). Two *hand-written backends per DB* are not — SQLAlchemy already speaks SQLite. Collapse to one backend per DB. |
| R12 | `-1` sentinels for ints, empty-string for text, `INTEGER` booleans, NULL for some timestamps — all mixed | tkarcheski | Per-field convenience. | **SOFTEN** — inconsistency makes queries error-prone (`WHERE score <> -1 AND score IS NOT NULL`). Pick one convention. |

**Dumb requirements called out loudly:** R4 (three databases), R7 (EAV metrics),
R9 (zombie coverage table). None survives a "would the project break without it?"
test. R5 and R6 are *premature*, not wrong — they encode a real future intent at
a cost the present can't justify.

---

## Step 2 — Delete the part or the process

Only requirements that survived Step 1 are eligible to keep; everything else gets
a deletion attempt.

| Part / table / process | What breaks if deleted | Delete? | Add-back |
|---|---|---|---|
| `coverage_reports` table + its 4 Superset charts | Nothing — it's never written; charts are empty today. | **YES, now** | none |
| `agentic_metrics` (EAV table) | Nothing in Superset; no validated consumer. | **YES** | partial → typed perf columns (Step 3), not EAV |
| `agentic_harnesses/plugins/skills` (3 tables) | Lose per-session plugin/skill inventory — not surfaced anywhere today. | **YES, collapse** | partial → keep `session_id`+`model_harness` columns already on `test_runs`; add **one** thin `harness_runs` row if provenance proves needed |
| Agent Workflow DB (4 tables) | Lose agent-reasoning debug logs — a *separate product surface* from benchmarking. | **SPLIT OUT** (not delete) | move to its own optional store outside the metrics DB; out of scope for "RF basics" |
| `models` table (current shape) | Nothing — metadata columns never populated. | **YES, redesign** | full → repopulated `models` table fed at run start (Step 3) |
| 3 of 6 hand-written backends (the bespoke SQLite ones) | Lose hand-tuned SQLite SQL; SQLAlchemy covers it. | **YES** | partial → one SQLAlchemy backend per surviving DB |
| `output_xml_gz` BLOB column | Lose in-DB copy of output.xml — but `results/` submodule holds it. | **YES** | partial → `output_xml_ref` pointer (path/URL into results submodule) |
| ~25 DROP-COLUMN migrations | Lose upgrade path from legacy schemas. | **YES, if** re-import is the accepted reset path | partial → one clean v2 baseline + documented `result_importer` re-import |
| `test_runs`, `test_results`, `*_artifacts`, `test_results_full` view | The entire simple-review read path. | **NO — core** | — |

### Deletion accounting
- **Proposed for deletion/relocation:** 2 whole databases (8 tables), 1 zombie
  table, 1 EAV table, 1 orphan table, 3 backend classes, ~25 migrations, 1 BLOB
  column. Roughly **~13 schema objects + ~28 code/migration artifacts**.
- **Expected add-back (~10% target):** 1 `harness_runs` table (maybe), repopulated
  `models` table, a small set of typed perf columns/table, 1 `output_xml_ref`
  pointer column, 1 v2 baseline. ≈ **3–4 objects added back ≈ 15–20% of what's
  removed** — comfortably over the 10% floor, so the deletion is appropriately
  aggressive without being reckless. If the final design adds back *less* than
  ~10%, revisit — we were too timid on the core.

---

## Step 3 — Simplify / optimize (survivors only)

Applied **only** to what survived Step 2.

1. **One database, one schema.** Fold the surviving tables into a single store
   (SQLite locally, Postgres live — *same schema*). Eliminates cross-DB soft-FKs
   (real `FOREIGN KEY … ON DELETE CASCADE` instead of smell #6's unenforced links).
2. **One backend per DB → one backend, period.** Standardize on SQLAlchemy Core
   for both engines; delete the bespoke SQLite SQL. ~3 classes of code gone.
3. **Core spine stays, trimmed:**
   `test_runs (1) ─< test_results (N) ─< test_result_artifacts (0..1)`
   `test_runs (1) ─< test_run_artifacts (0..1, now a pointer)`.
4. **`test_results_full` view:** keep (it *is* the simple-review path) but project
   only the columns Superset actually selects — drop the rest from the view.
5. **Perf shape — DECISION (the "decide during audit" item):** the delete pass
   shows the leaderboard does **not** need an EAV or a heavy new subsystem. Go
   **typed + lean**:
   - keep `test_results.eval_count`, `thinking_tokens`; **add** `eval_duration_ns`,
     `tokens_per_second` (and `prompt_eval_*` if cheaply available) as **typed
     nullable columns on `test_results`** — not a separate table, until fill-rate
     proves otherwise;
   - **repopulate `models`** (name, family, quantization, size_gb, context_length,
     sha256) from the Ollama/vLLM `/api/show` (or vLLM model endpoint) at run start;
   - **hardware**: promote the existing `hostname` to a real node identity — add
     `gpu`, `vram_gb`, `accelerator` either as columns on `test_runs` or a tiny
     `nodes` lookup keyed by hostname (lookup preferred: hardware is per-node, not
     per-run). This makes "best score-per-second for model@quant on hardware H,
     per suite" a plain JOIN of `test_results × test_runs × models × nodes` — **no
     EAV, no new pipeline.**
6. **Consistent conventions:** NULL (not `-1`) for absent numerics; real
   `BOOLEAN`; document the one convention in `ai/agents.md`.
7. **vLLM alignment:** make `models`/perf columns vLLM-first (vLLM exposes
   `usage`/timing per request) with Ollama as the fallback mapping — so the same
   typed columns serve both as AI hardware comes in-loop.

---

## Step 4 — Accelerate cycle time (survivors only)

1. **`make db-reset` + re-import.** One command drops to the v2 baseline and
   re-imports from the `results/` submodule via `result_importer.py`. Schema
   iteration goes from "write a migration, hope" to "edit baseline, reset,
   re-import" in seconds.
2. **Schema-as-code, single source.** One schema module generates both the SQLite
   and Postgres DDL and the Superset datasets — no hand-syncing two backends and a
   bootstrap script.
3. **Superset views from the schema.** `bootstrap_dashboards.py` derives datasets
   from the canonical schema, so a column add shows up in Superset without manual
   dataset edits.
4. **Faster reviews:** with the leaderboard expressible as one JOIN, the "simple
   review" is a single saved Superset dataset instead of multiple KPI virtual
   datasets stitched together.

---

## Step 5 — Automate (last — only the stable, simplified survivors)

1. **Auto-populate `models` + perf** on each run from the vLLM/Ollama API — *after*
   the typed columns from Step 3 exist and are proven non-empty.
2. **Auto-bootstrap Superset** datasets/views from the canonical schema in CI.
3. **Schema-drift guard:** the issue/PR monitoring heartbeat (built this session)
   can flag PRs that touch the schema without updating the baseline + re-import
   test.
4. **NOT yet:** do **not** automate harness/workflow capture or perf collection
   until (a) the v2 schema is migrated and (b) fill-rate confirms the columns earn
   their place. Automating them now would automate the very waste Steps 1–2 removed.

---

## Verdict

**Delete now (no reason to exist):** `coverage_reports`, `agentic_metrics` (EAV),
the current `models` shape, 3 bespoke SQLite backends, the `output_xml_gz` BLOB,
the 25-migration chain (in favor of a v2 baseline + re-import).

**Collapse:** the 4 `agentic_*` tables → the `session_id`/`model_harness` columns
already on `test_runs` (+ optional single `harness_runs` table).

**Split out of the metrics DB:** the Agent Workflow DB (4 tables) — it's a
separate developer-telemetry concern, not RF benchmarking.

**Survives as the core:** `test_runs → test_results → (run/result artifacts)` +
`test_results_full`, unified into **one database, one SQLAlchemy backend**, with
**typed lean perf columns** + a **repopulated `models`** + a **`nodes` hardware
lookup**. The vLLM/hardware leaderboard becomes a single JOIN — no EAV, no new
pipeline.

**Deletion accounting:** ~13 schema objects + ~28 code/migration artifacts removed;
~3–4 added back (≈15–20%) — over the 10% floor, so the cut is aggressive but not
reckless.

## Open questions / owners to chase (all → tkarcheski)

1. **R4/R6:** Is the Agent Workflow DB actively used for debugging today, or is it
   speculative? If unused → delete rather than split out.
2. **R5/#350:** Is the agentic-harness integration (Issue #350) imminent? If yes,
   keep the 2 provenance columns + 1 `harness_runs` table; if no, drop to columns
   only.
3. **R2:** Is in-DB `output.xml` ever read directly, or always via the `results/`
   submodule? Confirms the BLOB → pointer swap is safe.
4. **R3:** What fraction of `test_result_artifacts` rows are non-empty? Sizes the
   archive and confirms it's worth a table vs columns.
5. **R10:** Is "drop to v2 baseline + re-import from `results/`" an acceptable reset
   path (i.e. is the submodule the source of truth)? If not, we keep a real
   migration path and the cut is smaller.
6. **Perf:** vLLM-first or Ollama-first for the initial perf-field population, given
   AI hardware is coming in-loop?

---

## Blueprint (proposed v2 — for the follow-on implementation)

> Design only. Implementation follows the repo's TDD rule (failing pytest first),
> one atomic migration per idea, on a branch off `claude-code-staging`.

**One database `rfc` (SQLite local / Postgres live), one SQLAlchemy backend.**

```
nodes(hostname PK, gpu, vram_gb, accelerator, cpu, ram_gb)         -- hardware lookup
models(name PK, family, quantization, size_gb, context_length, sha256, source)
test_runs(id PK, ts, model_name→models, hostname→nodes, test_suite,
          total, passed, failed, skipped, duration_s,
          git_commit, git_branch, rfc_version,
          session_id, model_harness)                               -- provenance kept as columns
test_results(id PK, run_id→test_runs, test_name, status, score,
             tags, tier, verify, severity,
             eval_count, thinking_tokens,
             eval_duration_ns, tokens_per_second)                  -- typed lean perf
test_run_artifacts(run_id PK→test_runs, output_xml_ref, output_xml_source)  -- pointer, not BLOB
test_result_artifacts(result_id PK→test_results, question, expected, actual,
                      grading_reason, thinking_text)
-- view: test_results_full  (only Superset-selected columns)
-- OPTIONAL: harness_runs(session_id PK, tool_name, tool_version, model_id, ...)  if #350 lands
-- SPLIT OUT (separate store, not in rfc): agent_workflows/interactions/tool_calls/results
```

**Phased migration (each phase its own PR, green on its own):**
1. **Baseline + delete zombies.** Establish v2 schema module; drop `coverage_reports`
   (+ its Superset charts) and `agentic_metrics`. Failing test → migration → green.
2. **Unify backend.** Replace bespoke SQLite backends with one SQLAlchemy backend;
   prove SQLite + Postgres parity in tests.
3. **Collapse agentic_* → columns;** split Agent Workflow DB into its own module.
4. **Perf + hardware:** add typed perf columns, `models` repopulation at run start,
   `nodes` lookup; wire the single-JOIN leaderboard dataset in Superset.
5. **Pointer-ize artifacts:** `output_xml_gz` BLOB → `output_xml_ref`; backfill via
   `result_importer`.
6. **Automate** (Step 5) only after 1–5 are stable.

This sequence is itself the algorithm: question (1) → delete (1–3) → simplify (2–4)
→ accelerate (`db-reset`/re-import) → automate (6), in order.
