# Contributing to MetaExtractor

Thanks for helping. This document covers the parts of this repository that are
**load-bearing and easy to break by accident** — rules that otherwise live only
in test docstrings and `pyproject.toml` comments. Installation, the CLI, the
Python API, and the evaluation metrics are documented in the [README](README.md)
(see §Install and §Developing for the dev setup: `uv sync --dev`,
`uv run pytest`, `uv run ruff check .`); this file links to them rather than
restating them.

## The gold firewall — read this before you touch imports

MetaExtractor is scored against human-curated curatedMetagenomicData (cMD)
tables: one row per sample, columns = field names, curated by people who read
the papers. Every precision/recall number in the manuscript means "how close
does the extractor get to the curators". An extractor that could read those
tables — directly, transitively, or through a helper that happens to be shared
with the scorer — would make every metric meaningless. And it would do so
*silently*: routing around the firewall doesn't fail loudly, it just inflates
every number until nothing about the evaluation means anything.

So the boundary between extraction and scoring is guarded by
[`tests/test_gold_firewall.py`](tests/test_gold_firewall.py), which pins four
invariants plus a fifth on the schema:

1. **Import graph** — importing the CLI must not pull in the scoring modules
   (`metaextractor.evaluation`, `metaextractor.alignment`). Checked in a
   subprocess, because the test session imports them itself.
2. **Static source scan** — no extraction-path module may name a curated path,
   even in code no test exercises. The forbidden literals (`CMD_CURATION_DIR`,
   `_sample.tsv`, `_ontology_term_id`, `crosswalk_`, `load_gold`,
   `inst/curated`) are listed in the test.
3. **One-way dependency** — the evaluation side may import the library; the
   library may never import the evaluation side or the benchmark tree.
4. **The frozen-run argv** — the benchmark runner may know where the curated
   checkout is (it records its git HEAD for provenance) but must never pass a
   path from it to the extractor.
5. **Schema disclosure** — what the schema may tell the model is the answer
   space (enum labels, ontology CURIEs), never the answers.

What this means in practice:

- The extraction-path module list in the test (`EXTRACTION_PATH`) is spelled
  out by hand on purpose, so that adding a module to the extraction path is a
  conscious edit. If your change adds one, updating that list is part of the
  change — and it will be reviewed as such.
- If a helper is needed on both sides, put it in a third module that reads no
  gold and have both sides import it. Don't import `evaluation` from the CLI
  "just for this one constant".
- When the firewall test fails, the fix is almost never to relax the test. If
  you believe an invariant is wrong, open an issue and explain which reported
  metric survives it.

Two known exposures are deliberately **not** guarded, because they are eval-side
and measured rather than asserted: `bench.py run-tables` feeding a curated file
to the extractor as a supplementary table (ad hoc, not on the frozen path), and
the harmonization crosswalks unioning the gold side when canonicalizing values.
Both are documented at their source; see
[`evaluation/extensions/`](evaluation/extensions/).

## The `anthropic==0.104.1` constraint

`[tool.uv]` pins `anthropic==0.104.1` for the **development and CI environment**
(`[project.dependencies]` stays permissive at `anthropic>=0.40.0` for the
published package). This is not a stale pin to tidy up: both frozen benchmark
trees (`benchmarks/cmd_sonnet5/out_v3`, `out_v4`) and every number the
manuscript reports were produced with SDK 0.104.1, recorded per attempt as
`anthropic_sdk` in each `<study>.meta.json`. A fresh resolve picks 1.12.x — a
major version jump.

The point is that the next benchmark re-run must differ from `out_v4` by the
code under test and nothing else. **Lifting the pin is a deliberate, separately
measured step with its own benchmark run** — not a routine dependency bump, and
not something to slip into a feature PR. If your change genuinely needs a newer
SDK, say so in an issue first.

## The frozen benchmark trees

Everything under `benchmarks/` that is an *output* tree (`out_*`, `haiku_r*`,
`sonnet_r*`, `REPORT.md`, `summary.json`, `ledger.jsonl`, …) is a frozen run
artifact backing numbers in the manuscript ([`MANUSCRIPT.md`](MANUSCRIPT.md)).
Don't regenerate, reformat, or "clean up" those files, and don't commit new
attempts into an existing tree. The one-off research scripts beside them are
excluded from lint on purpose (`extend-exclude` in `pyproject.toml`); their
terse style is deliberate and their outputs are frozen artifacts.

## Tests are hermetic by design

The default suite needs no network and no `ANTHROPIC_API_KEY` — that is a
guarantee, not an accident:

- every HTTP path is monkeypatched at a module-level `_http_get` choke point;
- every model call goes through the injected seam (`metaextractor.model.Model` —
  `MockModel` in tests; see README §"Choosing or faking the model").

Consequences for new code: a new network call must route through the same
choke point (or a new, equally patchable one), and every model call must go
through `metaextractor.model`. An inline `requests.get` or an `Anthropic(...)`
client will pass locally and then silently break the guarantee the whole suite
is built on. Tests that genuinely need a live service get the `network` marker
and stay deselected by default (`addopts = "-m 'not network'"`).

## Orientation

Three console scripts (`[project.scripts]`):

| command | entry point | what it is |
|---|---|---|
| `metaextract` | `metaextractor.cli:main` | the extractor CLI |
| `metaextract-app` | `metaextractor.launch:main` | Streamlit web-app launcher |
| `metaextract-review` | `metaextractor.review.cli:main` | human-review packets & verdicts |

Module map of `src/metaextractor/`, one line each:

- `extractor.py` — core pipeline: schema in, structured records out.
- `model.py` — the model seam; the only place an LLM call happens.
- `prompts.py` — the system prompt (the extractor's contract; kept stable so prompt caching hits).
- `fetcher.py` — NCBI PMID/PMCID text retrieval (PMC full text, else abstract).
- `supplementary.py` — supplementary-material fetch; table/text extraction.
- `sra_manifest.py` — SRA/ENA run manifest → per-sample table.
- `table_select.py` — deterministic scoring of which tables carry the schema.
- `table_plan.py` — structural reading of a table (row-0 parse or an LLM structure plan).
- `column_mapper.py` — deterministic column → schema-field mapping.
- `sample_rules.py` — deterministic post-extraction rules (age bounds).
- `decision.py` — cheap bounded judgments behind a seam, with an audit trail.
- `schema.py` / `schema_loader.py` — input schema model; JSON/YAML/LinkML loading.
- `adapters/linkml.py` — LinkML → internal schema adapter.
- `output.py` — the result envelope (Pydantic).
- `writers.py` — JSON (primary) and flat CSV output with per-field provenance.
- `keys.py` — API-key resolution shared by CLI and web app.
- `cli.py`, `launch.py`, `webapp.py` — CLI, Streamlit launcher, Streamlit app.
- `evaluation.py` — gold scoring (**eval side**; must stay off the extraction path).
- `alignment.py` — identity alignment of extracted samples to gold rows (**eval side**).
- `review/` — self-contained HTML review packets and verdict tooling.

## Pull requests

- Say up front whether the PR is docs-only, behavior, or benchmark-related. A
  behavior change ships with tests; a benchmark change needs its own
  justification.
- Keep the diff inside the module that owns the change; no unrelated refactors.
- `uv run pytest` and `uv run ruff check .` must pass before you push.
- If something was ambiguous, surprising, or undocumented while getting set up,
  put it in the PR description (or a comment) — that friction log is valuable.
  Things that are bugs go in separate issues, not into this PR.