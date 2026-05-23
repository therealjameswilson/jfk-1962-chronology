# AGENTS.md

## Project: JFK 1962 Chronology

Build a day-by-day chronology of January 1 – December 31, 1962 (the second calendar year of the Kennedy presidency), surfacing every passage in the 2025 NARA JFK assassination records release that either (a) was authored on that day, or (b) references that day retrospectively from a later document (most often mid-1970s Church Committee, Rockefeller Commission, or HSCA-era materials).

Key 1962 events heavily represented in the release include Operation Mongoose, the Cuban Missile Crisis (October 16–28), and ongoing Berlin tensions.

The end product is a set of per-day markdown files plus a master index, suitable for scholarly citation.

## Repository Layout

- `../jfk/` — READ ONLY. Cloned `doctly/jfk` corpus (~60,000 pages of 2025 release converted to markdown). Never write here.
- `src/` — Python pipeline modules.
- `data/` — Intermediate artifacts (JSON indices, extraction caches). Git-ignored except for schemas.
- `output/` — Final chronology markdown. One file per day: `1962-01-01.md` ... `1962-12-31.md`, plus `index.md` and per-month rollups `output/by-month/1962-MM.md`.
- `tests/` — Pytest suite. Fixtures use a 100-document sample in `tests/fixtures/`.
- `notebooks/` — Exploratory Jupyter work. Not part of the production pipeline.

## Core Design: Dual-Axis Chronology

Every excerpt has two dates:

1. **Referenced date** — the 1962 day the passage describes (the chronology key).
2. **Document date** — when the source document was authored.

Each per-day markdown file MUST have two sections:

- `## Contemporaneous (1962)` — document date within Jan–Dec 1962.
- `## Retrospective` — document date after 1962. Sub-grouped by originating agency (CIA, FBI, Church Committee, HSCA, etc.) and sorted by document date ascending.

## Date Extraction Requirements

Recognize these formats for the **referenced date** scan over Jan 1 – Dec 31, 1962:

- `January 1, 1962` / `Jan. 1, 1962` / `Jan 1 1962`
- `1 January 1962` / `1 Jan 62`
- `1/1/62` / `01/01/1962` / `1-1-62`
- `1.1.62` (rare, European-style in some cables)
- Bare `January 1` or `1/1` ONLY when the surrounding paragraph independently establishes the year as 1962 (require a second-pass year anchor).

Edge cases to handle explicitly:

- Two-digit years: `62` resolves to 1962 only; reject `1862`, `2062` matches.
- Ranges: `October 16–28, 1962` expands to thirteen day-keys.
- "Early/mid/late October 1962" goes to a separate `month-level/1962-MM.md` bucket, NOT to specific days.
- Quarter and season references ("spring 1962", "Q3 1962", "fall 1962") go to `quarter-level/1962-QN.md`.
- OCR artifacts: `l962`, `196l`, `19G2`, missing commas. Maintain an OCR-variant table in `src/date_patterns.py`.

For each hit, capture a ±300-word context window, the source file path, the source document date, and the originating agency.

## Document Metadata Extraction

Each markdown file in `../jfk/` typically begins with a header derived from the NARA RIF (Record Identification Form). Parse:

- RIF number / record number
- Originating agency
- Document date (this is the **document date**, not the referenced date)
- Document type (cable, memo, report, transcript, etc.)
- Classification level (at time of release)

If the document date is illegible or absent, mark the document as `doc_date: unknown` and route it to a `review_queue.jsonl` for manual triage. Do NOT guess.

## Output Format

Per-day file template:

```
# 1962-MM-DD — [Day name]

> Brief one-line context (e.g., "Cuban Missile Crisis, Day 1", "Operation Mongoose planning meeting"). Drawn from a curated `key_events.yaml`, not invented.

## Contemporaneous (1962)
### [Source filename or RIF] — [Doc date] — [Agency]
> Excerpt with ±300 words of context.
[Link to source markdown in ../jfk/]

## Retrospective
### Church Committee (1975–76)
...

### HSCA (1976–79)
...
```

The master `output/index.md` lists all 365 days with hit counts per axis. Per-month rollups in `output/by-month/` provide month-at-a-glance summaries with hit-count tables and the top 5 most-referenced days that month.

## Scope and Filtering

- **Window**: January 1 – December 31, 1962. Full calendar year. 365 day-files total.
- **Topic filter**: NONE by default. Surface everything the files say about each day.
- **Deduplication**: If the same passage appears in multiple released versions (different redaction states), prefer the least-redacted version and note alternates in a footnote.
- **Empty days**: Days with zero hits still get a stub file noting "No references in 2025 release." Do not skip them — the absence is itself a research finding.

## Coding Conventions

- Python 3.11+. Use `uv` for environment management.
- Type hints required. `mypy --strict` must pass.
- Format with `ruff format`; lint with `ruff check`.
- Tests with `pytest`. Aim for ≥85% coverage on `src/`.
- No network calls in the pipeline. The corpus is local; results must be reproducible offline.
- Logging via `structlog`, JSON output to `data/logs/`.

## Performance Targets

- Full corpus scan should complete in under 15 minutes on an M-series Mac.
- Use multiprocessing for the file-walk and regex passes.
- Cache parsed metadata in `data/metadata.parquet`; rebuild only on `--refresh`.

## Quality Bar (Scholarly Use)

This output will be cited in academic work. Therefore:

- Never paraphrase excerpts. Quote verbatim from the markdown, preserving OCR artifacts, with `[sic]` annotations only where strictly necessary.
- Every excerpt must link back to its source file and, where possible, the NARA record number.
- A `--audit` mode emits a sample of 50 random hits with full context for manual spot-checking before publication.
- Maintain a `KNOWN_ISSUES.md` for systematic OCR or date-parsing failures discovered during review.

## What NOT to Do

- Do not summarize or interpret the documents. The chronology is a primary-source compilation, not analysis.
- Do not modify files in `../jfk/`.
- Do not infer dates that are not explicitly in the text.
- Do not silently drop ambiguous hits; route them to `review_queue.jsonl`.
- Do not add topic filters without an explicit user instruction.

## First Tasks for the Agent

1. Scaffold the project (`pyproject.toml`, `src/`, `tests/`, `output/`).
2. Implement `src/metadata.py`: walk `../jfk/`, extract per-document metadata, write `data/metadata.parquet`.
3. Implement `src/date_patterns.py` with full regex coverage and unit tests against `tests/fixtures/date_samples.txt`.
4. Implement `src/extract.py`: scan corpus, emit `data/hits.jsonl` with referenced-date, document-date, agency, file path, and context window.
5. Implement `src/render.py`: build per-day markdown plus `index.md` and per-month rollups.
6. Run end-to-end on the 100-doc fixture set; show the user 3 sample per-day files spanning different periods (e.g., Mongoose planning, Cuban Missile Crisis week, post-crisis) for review before scaling to the full corpus.
