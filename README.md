# pr4xis-prompter

*LLM-Interaction Strategy Toolkit — multi-axis classifier, two-layer strategy DB, self-extending via convergence.*

> **Status:** design (2026-04-23). Not yet implemented. This README is the load-bearing design spec until Phase 0 lands.
>
> **Repo:** https://github.com/peterpodj/pr4xis-prompter

---

## What this is

A Claude Code skill plus build pipeline that classifies LLM-interaction requests against a hierarchical, multi-axis taxonomy and returns ranked strategies (atomic tactics + composed playbooks) from a markdown-tree-backed database. When the database has no good match, the toolkit invokes the `convergence-investigation-2` skill as a background subagent, extracts the resulting tactics, and writes them back into the tree — making the system **self-extending**.

pr4xis-prompter sits structurally between `convergence-main` (research methodology, single-shot investigations) and `G0DM0D3-main` (live multi-model chat with hardcoded GODMODE/ULTRAPLINIAN combos). It generalizes G0DM0D3's hardcoded combo table into a data-driven, growable corpus, and gives convergence a recurring application surface.

## Hybrid skill+pipeline posture

- **Live assessor** — runtime path: user request → classifier → DB lookup → ranked strategies. Sub-second target.
- **Self-extending generator** — cold path: coverage gap detected → convergence subagent → composer LLM → atomic indexer rebuild. Minutes to hours; non-blocking; off the runtime path.
- **Manual UI** — `db/browser.html` single-file static viewer with composer + coverage heatmap. Independent third consumer of the index. Fallback when assessor stack is offline.

### Minimal mode (`PR4XIS_MODE=simple`)

Most of this README describes the *elaborate* mode. The toolkit also ships a stripped-down mode for cases where the elaborate machinery doesn't earn its complexity:

- **Closed-enum classifier** — depth-1 axes only, regex/keyword routing (no embeddings, no LLM call)
- **Flat strategy table** — atomic tactics only, no playbook composition layer
- **No convergence integration** — coverage gaps return "no strategy" instead of triggering investigation
- **No browser composer** — read-only tree view in `browser.html`

Minimal mode exists as a **falsification test** for the elaborate mode. If real-world telemetry shows minimal mode handles ≥ 70% of queries acceptably (operationalized as: user did not retry the query, did not request convergence, did not switch to elaborate mode), the elaborate mode is over-engineered for the actual workload and should be deprecated. The toggle removes the temptation to defend the design through architecture rather than evidence.

## Three orthogonal taxonomy axes

Each axis is a hierarchical tree. Lookups are by `(intent_path, op_path, surface_path)` triple with ancestor fallback and depth-based confidence decay.

| Axis | What it captures | Initial depth-1 children |
|---|---|---|
| `adversarial_intent` | What the user is trying to *make the model do* | refusal_bypass, prompt_injection, exfiltration, eval_probe, payload_smuggle, persona_break |
| `cognitive_operation` | The pattern of mental work being induced | extract, persuade, synthesize, decompose, role_induce, context_launder |
| `liberation_surface` | Which guardrail layer the request hits | post_training_refusal, system_prompt_fence, safety_classifier, output_filter, rate_limit |

> **Taxonomy provenance: INVENTED, not derived.** The three axes and their depth-1 children are this project's own framing, not a published taxonomy. They overlap with but do not directly map to OWASP LLM Top 10 (LLM01-LLM10), MITRE ATLAS adversarial-ML tactics, or academic frameworks (Greshake et al. on indirect prompt injection; Perez & Ribeiro on Ignore-Previous-Instructions). Phase 0.5 of the implementation plan back-fills explicit citations per leaf in `_meta.yml`; until then, treat the structure as a working hypothesis open to revision rather than a settled vocabulary. Downstream consumers should pin to a specific PR4XIS version and expect breaking taxonomy changes between v0.x releases.

Leaves carry `status: hypothesis | routable`. The lookup engine refuses to route to `hypothesis` leaves. Promotion to `routable` requires (a) ≥ N supporting tactics at the leaf AND (b) at least one of those tactics has tier `high` or `confirmed` from **independent** evidence — i.e. arena measurement (G0DM0D3), human review with a second-rater, or convergence finding with `confirmed` tier. Importer-default `tier: high` does NOT count as independent evidence (see "Promotion gate is non-circular" below).

## Two-layer data model

- **Tactic** — atomic artifact: one prompt, one perturbation function, one model choice, one sampling config, or one output filter. Tagged on all three axes. Provenance: `hand_tuned | convergence | speculation | measurement`. Tier: `confirmed | high | medium` (matches convergence epistemic discipline).
- **Playbook** — composed pipeline: `uses_tactics: [t_id, ...]` + `composition_order` + `expected_output_filter`. Cannot reference tactics with tier ≤ `medium` (invariant enforced by indexer).

### Promotion gate is non-circular

Earlier drafts of this spec defaulted seed imports (L1B3RT4S, CL4R1T4S, GODMODE) to `tier: high`. Combined with the leaf-promotion rule "leaf becomes routable when ≥ N high-tier tactics exist", this created a rubber-stamp loop: imports were trusted because we said they were trusted. Fixed:

- **Seed imports default to `tier: medium`.** Tactics from L1B3RT4S/CL4R1T4S enter as research-grade material, not auto-routable.
- **Promotion to `tier: high` requires independent evidence:** an arena score from G0DM0D3 (`provenance: measurement`), a second human reviewer (`provenance: hand_tuned` requires reviewer initials in the file), or a convergence finding with `confirmed` tier.
- The 5 GODMODE CLASSIC playbooks DO carry `provenance: measurement` from the start because G0DM0D3's arena scoring already produced independent evidence — that's the legitimate path.

Provenance ladder is monotonic: `speculation → hand_tuned → measurement`. Promotion to `measurement` requires arena evidence. Two ways `measurement`-tier rows enter the corpus:

- **At seed time** — G0DM0D3's GODMODE/ULTRAPLINIAN combos already carry arena scores; the seed importer preserves their `measurement` provenance verbatim.
- **At runtime** — out of scope for v1. Composer-generated playbooks start at `speculation` and stay there until a separate arena-validation loop (post-Phase-6 integration with `ultraplinian.ts`) measures them. v1 ships with seeded `measurement` playbooks alongside `speculation` proposals; the lookup engine ranks the former higher.

## Storage layout

Filesystem-as-DB is the **source of truth** (committed to git, human-diffable, browsable). A derived SQLite index + embeddings file accelerates runtime lookup. The indexer is the *single writer*; convergence and the composer write markdown, then the indexer rebuilds atomically (tmpfile + rename).

```
pr4xis-prompter/
├── pr4xis.skill/                  # the Claude Code skill itself
│   ├── SKILL.md
│   └── references/
├── src/
│   ├── assessor.ts                # two-stage classifier (embedding kNN → LLM)
│   ├── lookup.ts                  # SQL + ancestor fallback
│   ├── composer.ts                # LLM-driven playbook assembly
│   ├── convergence-bridge.ts      # invoke subagent + parse bundle
│   ├── indexer.ts                 # markdown tree → SQLite + embeddings
│   ├── telemetry.ts               # local jsonl writer
│   ├── hf-publisher.ts            # re-export from G0DM0D3-main/HF
│   └── cli.ts                     # pnpm pr4xis-{index,serve,eval,expand}
├── db/
│   ├── _meta.yml                  # root, declares axes
│   ├── adversarial_intent/        # axis tree
│   ├── cognitive_operation/
│   ├── liberation_surface/
│   ├── _state/                    # .gitignored — telemetry, locks, logs
│   ├── _reports/                  # convergence bundles (committed audit trail)
│   ├── _index/                    # .gitignored — derived SQLite + embeddings
│   └── browser.html               # single-file UI
├── tools/                         # one-time seed importers
│   ├── seed-from-l1b3rt4s.ts
│   ├── seed-from-cl4r1t4s.ts
│   └── seed-from-godmode.ts
├── test.js                        # single integration test (≤200 lines)
├── test-fixtures/
└── package.json                   # workspace
```

## Schemas

### Taxonomy node — `<axes>/<path>/_meta.yml`

```yaml
axis: adversarial_intent
slug: divider_based
parent: refusal_bypass
display_name: Divider-based refusal bypass
description: |
  Strategies exploiting model treatment of formatting dividers
  (=====, ###, code fences) as conversational boundaries...
status: routable           # hypothesis | routable
supporting_tactics_min: 3  # promotion threshold
references:
  - convergence_report: db/_reports/divider_based-2026-04-21.md
```

### Tactic — `<axes>/<path>/tactics/<slug>.md`

```yaml
---
id: l33t_divider_inverse
type: tactic
axes:
  adversarial_intent: refusal_bypass/divider_based/l33t_speak
  cognitive_operation: persuade/format_authority
  liberation_surface: post_training_refusal/instruction_following
tier: high                       # confirmed | high | medium
provenance: hand_tuned           # hand_tuned | convergence | speculation | measurement
source: [L1B3RT4S-main/OPENAI.mkd:142]
artifact_kind: system_prompt     # system_prompt | user_prefix | perturbation_fn |
                                 # output_filter | sampling_params | model_choice
applicable_models: [openai/gpt-4o]
known_failures: [...]
created: 2026-04-21
---

# Body — actual prompt / function / spec
```

### Playbook — `<axes>/<path>/playbooks/<slug>.md`

```yaml
---
id: godmode_classic_gpt4
type: playbook
axes: { ... }
tier: high
provenance: measurement
arena_score: 87/100
uses_tactics:
  - l33t_divider_inverse
  - parseltongue_l33t_perturb
  - openai/gpt-4o
  - sampling_aggressive_top_p
composition_order: [perturb_input, set_system, call_model]
expected_output_filter: strip_l33t_response
known_failures: [...]
---
```

### Derived SQLite (`_index/taxonomy.db`)

Tables: `axis_nodes(id, axis, path, depth, status, description)`, `strategies(id, type, tier, provenance, body, frontmatter_json, file_path)`, `strategy_axis_tags(strategy_id, axis_node_id)`, `playbook_tactics(playbook_id, tactic_id, position)`. Embeddings live in separate mmap-friendly `embeddings.bin` + `leaf_embeddings.bin`.

### Indexer-enforced invariants

1. Every `axes:` path resolves to an existing `_meta.yml` (no dangling tags).
2. Playbooks cannot reference `medium`-tier tactics.
3. `routable` leaves must have ≥ `supporting_tactics_min` `high`-tier tactics; auto-demoted otherwise.
4. Provenance ladder monotonic; demotion = delete and replace.

## Data flows

### Hot path — lookup

> **Latency budget (TARGETS, not measurements).** All numbers below are *targets to verify in Phase 2*, not benchmarks. Real numbers depend on embedding model, network, LLM provider, and corpus size. The `pr4xis-eval` harness in Phase 6 records actual p50/p95 to `eval-report.md`.

| Stage | TARGET p50 | Notes |
|---|---|---|
| Stage 1 (embedding kNN) | < 100ms local / < 300ms remote | Local `bge-small-en-v1.5` is faster but lower quality |
| Stage 2 (LLM classify) | < 1s | Single Haiku call, structured JSON output |
| Lookup (SQLite) | < 50ms | Indexed query, intersect three tags |
| Response composer | < 50ms | Pure local formatting |
| **TOTAL** | **< 1.5s p50** | If exceeded in eval, optimize stage 1 first |

1. **Assessor stage 1** — embed request, kNN over `leaf_embeddings.bin`, top-3 candidate paths per axis with cosine sim. Default embedding model: `text-embedding-3-small` via OpenRouter (chosen for cost/quality balance and consistency with G0DM0D3's existing OpenRouter integration); swappable via `PR4XIS_EMBED_MODEL` env var. Local fallback (`bge-small-en-v1.5` via transformers.js) when offline.
2. **Assessor stage 2** — single LLM call, given top-3 candidates per axis, returns final `(intent, op, surface, confidence)` triple. The LLM may emit `path: "none-fits"` for any axis when no candidate is appropriate — this is the explicit gap signal that triggers convergence-fill. Default model: `claude-haiku-4-5`; swappable via `PR4XIS_CLASSIFIER_MODEL`.
3. **Lookup engine** — SQLite intersection query on the triple. If 0 hits, walk each axis up one level (confidence × 0.7 per level), retry.
4. **Response composer** — top-K playbooks rendered with provenance + tier + matched-depth + known_failures. Coverage report appended. If gap detected → "trigger convergence?" prompt.

### Cold path — convergence-fill

Triggers (any one fires): stage-1 max sim < 0.65 on any axis | stage-2 returns "none-fits" | lookup hits = 0 after fallback to depth ≤ 2 | manual `/pr4xis-expand`.

Bridge → builds investigation brief (research_question, sources_to_consult including L1B3RT4S/CL4R1T4S, deliverable_kind: tactic_set, epistemic_floor: high) → invokes `convergence-investigation-2` as background subagent → on completion, parses graph JSON → **runs each candidate tactic body through a content-moderation classifier** (default: a small LLM check for CSAM/CBRN/explicit harm content; tactics flagged are written to `_reports/<topic>-moderated.md` audit trail with redacted bodies, never to the live tree) → emits one tactic markdown per `confirmed`/`high` node that passed moderation → composer LLM proposes 1–3 speculation-tier playbooks from the new tactics → indexer atomic rebuild → next lookup hits the new entries.

> **Calibration warning.** `convergence-investigation-2` is calibrated for institutional / cross-domain investigation (bridge figures, structural parallels, document parsing) — not specifically for prompt-engineering research. Phase 4 of the implementation plan includes a **manual pilot** before the bridge is wired into automatic triggers: invoke convergence on 2–3 real leaves, audit output quality, and only then enable automation. If output is poor, the cold path is replaced with a lighter "research-mode" alternative (web-search arxiv + the L1B3RT4S/CL4R1T4S corpora, no convergence).

### Manual path — `db/browser.html`

`sql.js` loads `taxonomy.db` (or fallback: walk markdown tree via `fetch`). Tabs per axis, collapsible tree, filter by tier/provenance/keyword, coverage heatmap (empty cells red). Composer pane: drag tactics → assemble playbook → export as markdown (download or POST to `pnpm pr4xis-serve` writer endpoint on `:7777`). Browser is the only UI for demote-leaf, deprecate-playbook, and view-batch-diff operations.

## Failure modes

**Runtime — fail loud, never silent.** Embedding model unavailable → skip stage 1, run stage 2 with full taxonomy in prompt. LLM fails → return stage-1 candidates with degraded banner. Index corrupt → refuse to serve, surface "run pnpm pr4xis-index". 0 hits after full fallback → coverage_status emitted, no fabrication. Stage-2 malformed JSON → single retry, then degrade. No `|| default`, no `catch { return null }`.

**Generation — fail safe, never partial-write.** Convergence crashes mid-run → discard partial output, log to `_state/triggers.log`. Convergence yields no high-tier tactics → write a `_reports/no-tactics-found-<date>.md` (preserve research) but zero tactic files. Composer violates schema → indexer rejects playbook, tactics still committed. Indexer atomic rename failure → old index stays live, log + non-zero exit. Two batches race → `flock` on `_state/writer.lock`. Malformed YAML → indexer rejects entire batch, old index stays live (intentional rigor).

**Convergence epistemic-tier collapse.** `confirmed`/`high` → tactic written. `medium` → tactic written but not playbookable (composer skips). `excluded` → `_reports/<topic>-excluded.md` only.

## Telemetry & opt-in HF publishing

Always-on local: `db/_state/telemetry.jsonl` — one line per lookup / gap / convergence-trigger / indexer-rebuild. Schema: `{ts, event, axes, stage1_max_sim, stage2_confidence, hits, fallback_levels, elapsed_ms}`. No PII keys; raw request text stored only as SHA256 hash + length + token estimate.

Opt-in upstream via `PR4XIS_HF_PUBLISH=1` (analytics) and separate `PR4XIS_HF_PUBLISH_RESEARCH=1` (convergence bundles — different sensitivity class). Reuses G0DM0D3's HF publisher module verbatim — same env vars (`HF_TOKEN`, `HF_DATASET_REPO`, `HF_FLUSH_INTERVAL_MS`, `HF_FLUSH_THRESHOLD`), same flush semantics, same opt-in/opt-out patterns. Inverted default (opt-IN here, vs G0DM0D3's opt-OUT) reflects strategy-query sensitivity > chat-completion sensitivity.

## Testing strategy

Single `test.js` at pr4xis-prompter root, ≤200 lines, real system end-to-end against `test-fixtures/mini-db/`. Nine sections:

1. **Indexer roundtrip** — build → query → match
2. **Schema invariants** — bad fixtures → indexer rejects each with expected error
3. **Lookup correctness** — fixture queries → expected playbook IDs
4. **Ancestor fallback** — empty leaf walks up correctly with confidence decay
5. **Classifier fixtures** — `(request → expected axes)` from `classifier-cases.jsonl`, against deterministic-hash embedding mock + LLM-response lookup table
6. **Convergence bridge** — mock convergence bundle → tactic extraction + composer invocation
7. **Browser sql.js fallback** — headless puppeteer/jsdom: delete `taxonomy.db` → "rebuild from markdown" button → in-page index build → tree renders + lookup works
8. **Telemetry jsonl shape** — required fields present; no PII keys; SHA256 hash where raw text would be
9. **HF publisher dry-run** — `PR4XIS_HF_DRY_RUN=1` intercepts upload, writes payload to `_state/hf-dryrun.log`; assert hashed bodies; assert opt-in absence → zero publisher calls (privacy tripwire)

Real LLM/embedding eval lives separately: `pnpm pr4xis-eval` runs held-out classifier set against real APIs, costs credits, run manually before merging classifier changes. Not in CI.

CI runs `test.js` only — sub-second, no network, no API spend. Eval triggered by PR label `eval-required`.

## Seeding plan

Bootstrap from existing taxonomy artifacts via three one-time importers in `tools/`. Counts below are TARGETS-pending-measurement (the real numbers come from `pnpm pr4xis-index` after Phase 1):

1. **L1B3RT4S** → tactics across `refusal_bypass` and `persona_break` subtrees, `provenance: hand_tuned`, **`tier: medium` default** (was `high` in earlier drafts — see "Promotion gate is non-circular"), `source` field preserves origin lineage. Promotion to `high` requires independent evidence on a per-tactic basis.
2. **CL4R1T4S** → tactics in `system_prompt_fence` subtree, `artifact_kind: system_prompt_reference` (these are *targets* of liberation, not liberation tools), `tier: medium` default.
3. **G0DM0D3 GODMODE/ULTRAPLINIAN** → 5 playbooks with `provenance: measurement` (legitimate — the arena scoring in G0DM0D3 IS independent evidence), `tier: high`. These anchor the playbook table.

## Roll-out phases

| Phase | Scope | Exit criterion |
|---|---|---|
| 0. Foundation | `package.json` workspace, `db/_meta.yml` axes (depth 1–2), schema validator, indexer, `test.js` sections 1–3 against `mini-db` | `pnpm pr4xis-index && pnpm pr4xis-test` green |
| **0.5. Taxonomy citations** | Per-leaf citations in `_meta.yml` mapping to OWASP LLM Top 10 / MITRE ATLAS / arxiv. Leaves with no citation get `provenance: invented` flag. | Every depth-1 leaf has either a citation or an explicit `invented` flag |
| 1. Seed corpus | Three importers (default `tier: medium`), hand-review output, real indexer run, lookup engine, `test.js` sections 3+4 | Real corpus indexed; ancestor fallback verified |
| 2. Classifier | `assessor.ts` (stage 1+2), `test.js` section 5, `pr4xis.skill/SKILL.md`, **minimal-mode toggle (`PR4XIS_MODE=simple`)** wired alongside elaborate mode | Skill invokable in both modes from Claude Code |
| 3. Browser | `db/browser.html`, `test.js` section 7 | Manual compose + coverage heatmap functional |
| **4.0. Convergence pilot** | Manually invoke convergence on 2–3 real leaves, audit output quality | Pilot report documents whether convergence produces usable tactics for prompt-engineering questions |
| 4. Convergence bridge | `convergence-bridge.ts` (with **content-moderation step**), `composer.ts`, `test.js` section 6, one real `pr4xis-expand` end-to-end | Real leaf expanded; moderation tripwire tested |
| 5. Telemetry + HF | `telemetry.ts`, HF re-export, `test.js` sections 8+9 | Dry-run privacy tripwire green |
| 6. Eval + docs | `pnpm pr4xis-evaluate` (with **adversarial cases + frequency-class baseline**), `README.md`, skill references | First convergence-driven leaf documented as case study; eval report includes baseline comparison |

Phases 0–2 are critical path. Phases 3–6 partially parallelizable (browser is independent; telemetry is independent). Phase 4.0 is a hard gate — if the convergence pilot fails, Phase 4 is replaced with a lighter "research-mode" alternative (see Cold path "Calibration warning").

## Deferred decisions

| Deferred | Rationale | Trigger to revisit |
|---|---|---|
| Normalize `applicable_models` / `known_failures` into tables | YAGNI until corpus crosses ~100 strategies | When cross-row analytics become real questions |
| Arena-validation loop (speculation → measurement) | Out of scope for v1; G0DM0D3 runs arenas externally | Post-Phase-6 — integrate with `ultraplinian.ts` |
| Multi-language classification | Prompts English-only initially | If non-English requests appear in telemetry |
| SkillForge cross-skill orchestration | SkillForge under-developed currently | After SkillForge stabilizes |

## Naming

Project: **pr4xis-prompter** (the toolkit). CLI / runtime short name: `pr4xis`. The `pr4xis` short form keeps `pnpm pr4xis-*` commands tight; `pr4xis-prompter` is the full project / repo name and disambiguates from any future `pr4xis-*` siblings. Riffs on the PRAXIS protocol referenced in convergence; aligns with sibling l33t-styled names (G0DM0D3, L1B3RT4S, CL4R1T4S, OBLITERATUS, ST3GG).

## Dependencies

- `pod-taxonomy/convergence-main/` — research methodology skill (invoked as background subagent on coverage gap)
- `pod-taxonomy/G0DM0D3-main/HF/` — HF dataset publisher reused verbatim for opt-in telemetry
- `pod-taxonomy/L1B3RT4S-main/` — seed corpus (jailbreak prompts, ~200–400 tactics)
- `pod-taxonomy/CL4R1T4S-main/` — seed corpus (leaked system prompts, ~30–50 tactics)

## License

TBD on first commit (likely MIT to match siblings).
