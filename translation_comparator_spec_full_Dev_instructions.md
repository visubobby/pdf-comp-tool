
# Cursor Development Instructions — Translation Comparator (Enterprise)

> Purpose: Provide concrete, step-by-step development instructions that a developer using **Cursor** can follow to implement the bank-grade Translation Comparison Utility. This document focuses only on developer tasks, repositories/files to create, CI/test requirements, and exact prompts to use for the Gemini stage. It assumes the enterprise constraints discussed previously (on-premise, approved libraries only).

---

## Minimum Viable Deliverables (MVP)
1. `compare_docs.py` CLI that accepts two PDFs and produces `report.json` and `report.html`.
2. `extractor/pdf_extractor.py` that extracts structured blocks (headings, paragraphs, tables, images metadata, footnotes) using `pdfplumber` or `pdfminer.six`.
3. `extractor/structure_parser.py` that maps extracted content into the canonical Block schema.
4. `aligner/align_blocks.py` that aligns source vs target blocks using TF-IDF + cosine similarity and heuristics (section numbering, headings).
5. `aligner/missing_extra.py` to detect missing/extra blocks and annotate with `alignment_status` and zeroed scores.
6. `scorer/meteor_scorer.py` using NLTK + WordNet; `scorer/bleu_scorer.py` using `sacrebleu`; `scorer/comet_proxy.py` implementing a TF-IDF cosine proxy.
7. `reports/report_json.py` and `reports/report_html.py` (Jinja2 template) to produce JSON and HTML outputs.
8. Unit tests for each module in `tests/` and a sample fixture in `tests/fixtures/`.

---

## Repo Layout (create these files in Cursor)
```
translation_comparator/
├── compare_docs.py
├── config.yaml
├── requirements.txt
├── extractor/
│   ├── __init__.py
│   ├── pdf_extractor.py
│   └── structure_parser.py
├── aligner/
│   ├── __init__.py
│   ├── align_blocks.py
│   ├── missing_extra.py
│   └── table_align.py
├── scorer/
│   ├── __init__.py
│   ├── meteor_scorer.py
│   ├── bleu_scorer.py
│   └── comet_proxy.py
├── reports/
│   ├── __init__.py
│   ├── report_json.py
│   └── report_html.py
├── utils/
│   ├── __init__.py
│   ├── text_cleaner.py
│   ├── table_handler.py
│   └── risk_sections.py
└── tests/
    ├── test_extractor.py
    ├── test_aligner.py
    ├── test_scorer.py
    ├── test_reports.py
    └── fixtures/
        ├── sample_source.pdf
        └── sample_translated.pdf
```

---

## Development Steps (explicit, for Cursor to implement)

### Step 0 — Initialize repo & environment
- Create Python venv and install enterprise-safe packages in `requirements.txt`:
  - `pdfplumber`, `pdfminer.six`, `PyPDF2`, `beautifulsoup4`, `lxml`, `jinja2`, `reportlab`, `nltk`, `sacrebleu`, `scikit-learn`, `pandas`, `pyyaml`
- Add a `config.yaml` with thresholds and flags (see template below).
- Ensure `nltk` WordNet data is downloaded during CI only (offline cached wheel or pre-downloaded in the container).

### Step 1 — PDF Extraction (extractor/pdf_extractor.py)
- Implement a function `extract_pdf_blocks(path: str) -> List[Block]`:
  - Use `pdfplumber` to iterate pages, extract:
    - Text blocks with bbox, font-size metadata if available.
    - Tables via `page.extract_tables()` and normalized cell text.
    - Images metadata (`object_type` or image bbox); do NOT OCR by default.
    - Headers/footers heuristics (repeat on multiple pages).
  - Return a list of canonical Blocks (see Block schema below).
- Write unit tests: `test_extractor.py` should verify block count, table extraction, and page-level ordering on `tests/fixtures/sample_source.pdf`.

### Step 2 — Structure Parser (extractor/structure_parser.py)
- Implement `to_blocks(raw_blocks) -> List[Block]` that:
  - Normalizes whitespace, fixes hyphenation across lines, assembles multi-line paragraphs.
  - Detects numbered headings (regex like `^\d+(\.\d+)*`) and assigns `section_id` and `section_path`.
  - Splits lists from paragraphs; preserve nesting level in `list_level`.
  - Convert `pdfplumber` tables into canonical `table` arrays.
- Unit tests: assert `section_id` detection and list nesting on fixture docs.

### Step 3 — Alignment Engine (aligner/align_blocks.py)
- Implement `align_blocks(src_blocks, tgt_blocks, cfg) -> List[SectionComparison]`:
  - Use primary matching by `section_id` if present and identical.
  - If `section_id` missing or mismatched, use heading text similarity (normalize case, punctuation) with TF-IDF cosine.
  - For paragraph-level matching inside a section, compute TF-IDF vectors per paragraph and match by highest cosine similarity.
  - Implement a sliding window search to avoid O(n^2) across entire doc and allow local reordering.
  - Return paired blocks with similarity metadata.
- Provide configuration in `config.yaml` for thresholds: `high: 0.80`, `review: 0.60`.
- Tests: `test_aligner.py` should validate correct alignments and that confidence thresholds are respected.

### Step 4 — Missing & Extra Detection (aligner/missing_extra.py)
- Implement logic to mark blocks as `missing_in_target` or `extra_in_target`:
  - If a source section has no candidate above `review` threshold → mark missing.
  - If a target section cannot be matched to any source section within window → mark extra.
  - For tables, if row/column counts differ significantly, mark as `partial_match` and list cell-level issues.
- Tests: include cases where source contains a clause absent in target and vice versa.

### Step 5 — Table Alignment (aligner/table_align.py)
- Implement `align_tables(src_table, tgt_table) -> dict`:
  - Compare headers (string similarity) to align columns.
  - If headers missing, align by index but flag uncertainty.
  - Return per-cell diff and flag missing/extra cells.
- Tests: test with merged cells and missing rows.

### Step 6 — Scoring (scorer/)
- **meteor_scorer.py**:
  - Implement METEOR using NLTK WordNet stems and synonym mapping; return 0.0–1.0 per section.
- **bleu_scorer.py**:
  - Wrap `sacrebleu` to compute BLEU with smoothing; normalize to 0–1 scale.
- **comet_proxy.py**:
  - Implement TF-IDF vectorization (scikit-learn `TfidfVectorizer`) on paired texts and compute cosine similarity as proxy for COMET. Return 0–1.
- Aggregate with the formula in `config.yaml` to compute `quality_index` with coverage penalty.
- Tests: known sentence pairs with expected score ranges.

### Step 7 — Risk Tagging (utils/risk_sections.py)
- Implement a configurable keyword lexicon for High/Medium/Low severity terms.
- Provide a function `tag_severity(section_text) -> severity` that considers both source and target text and known legal keywords.
- Tests: assert that "Haftung" or "Liability" map to `high` severity.

### Step 8 — Reporter (reports/report_json.py & reports/report_html.py)
- **report_json.py**:
  - Produce `report.json` containing `sections: []` and `document_summary` per schema.
- **report_html.py**:
  - Implement Jinja2 templates for a two-column layout.
  - Each section shows source (left) and target (right), scores, severity tag, and inline issues highlighted.
  - Add a top summary box with overall scores and critical issues.
- Make HTML CSS minimal and print-friendly for `wkhtmltopdf`/`WeasyPrint` if ops chooses that route.
- Tests: generate HTML for fixture and validate presence of key summary elements.

### Step 9 — CLI (compare_docs.py)
- Implement CLI that wires all modules:
  - `python compare_docs.py source.pdf target.pdf --out report.html --json report.json --pdf report.pdf`
  - Config flags: `--enable-comet` (default off), `--ocr` (default off), `--verbose`.
- Ensure exit codes: `0` success, `>0` on fatal errors.
- Tests: basic smoke test running CLI on fixtures and verifying outputs exist.

### Step 10 — Tests & CI
- Provide `tests/` with unit tests and `fixtures/` sample PDFs (redacted or synthetic).
- Add a `Makefile` or `tox` config to run tests and lint.
- CI must run in an environment that simulates offline mode (no network). Include instruction to cache NLTK data in the container image.

### Step 11 — Gemini Prompt Pack (for alignment validation)
Include the exact prompt file `prompts/gemini_alignment_prompt.txt` with the following content (Cursor will use this string when calling Gemini if ops approves):

```
System Prompt:
You are a document alignment and comparison engine for bank-grade legal documents. Keep all structure, numbering, and headings. Do not summarize or omit text.

User Task:
Given two JSON documents (source: German, target: English) representing parsed PDF blocks, align blocks section-by-section and paragraph-by-paragraph. For each alignment produce:
- section_id
- source_text
- target_text (null if missing)
- alignment_status: perfect_match|partial_match|missing_in_target|extra_in_target
- comet_score (0.0-1.0) [use 0.0 if disabled]
- meteor_score (0.0-1.0)
- bleu_score (0.0-1.0)
- issues: list of strings (e.g., 'missing clause', 'terminology mismatch')

Then compute document_summary:
- overall_meteor, overall_bleu, overall_comet_proxy, quality_index, critical_issues

Output only JSON suitable for downstream rendering.
```

---

## Configuration Template (`config.yaml`)
```yaml
thresholds:
  high: 0.80
  review: 0.60
penalties:
  missing_extra_zero: true
weights:
  meteor: 0.5
  comet_proxy: 0.3
  bleu: 0.2
risk_keywords:
  high: ["liability","indemnity","jurisdiction","governing law","termination","interest","collateral"]
  medium: ["confidentiality","data protection","service level","audit"]
render:
  engine: "jinja2"
  pdf_export: "reportlab"   # or "wkhtmltopdf"
security:
  outbound_network: false
  pii_log_truncation: 256
```

---

## Coding Conventions & Notes for Cursor
- Use typed Python (type hints) and docstrings for public functions. Keep functions small (< 200 LOC).
- Fail fast on unexpected input; validate JSON schemas at module boundaries using simple checks (or `pydantic` if approved).
- Keep all IO paths configurable in `config.yaml` and support relative paths for test fixtures.
- Avoid mutable global state; pass `cfg` objects into functions.
- Keep Gemini integration optional and behind a runtime flag; default run-mode is completely offline.

---

## Acceptance Criteria (for PR merge)
- All unit tests pass locally and in CI.
- `compare_docs.py` runs on the sample fixtures and generates `report.json` and `report.html` matching golden snapshots in `tests/fixtures/golden/`.
- Code is linted (PEP8) and type-checked (mypy) if approved.
- README contains run instructions and operations notes (how to enable COMET, how to run in offline mode).

---

## Quickstart: Commands for Cursor to run after scaffolding
```bash
# create virtualenv and install requirements
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# run unit tests
pytest -q

# run sample comparison
python compare_docs.py tests/fixtures/sample_source.pdf tests/fixtures/sample_translated.pdf --out tests/out/report.html --json tests/out/report.json
```

---

## Deliver the following artifacts in the Cursor PR
1. Source code under the repo layout above.
2. `tests/` with fixtures and golden snapshots.
3. `config.yaml` with default thresholds and flags.
4. `prompts/gemini_alignment_prompt.txt` with the exact prompt.
5. `README.md` with setup, run, and ops notes (how to enable COMET and OCR).

---

End of developer-only instructions. Follow this step-by-step checklist in Cursor to implement the first fully functional, enterprise-safe MVP.
