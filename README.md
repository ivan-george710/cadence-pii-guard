# Cadence PII Guard

Cadence PII Guard extracts document text, detects personal information, and redacts confirmed spans while preserving internal audit metadata.

## Architecture

1. **Engine 1 — File Intelligence:** identifies file type and extraction strategy.
2. **Engine 2 — Extraction + OCR:** produces ordered pages, original text and available OCR blocks.
3. **Engine 3 — Deterministic PII Detection:** Microsoft Presidio, the Cadence employee-ID recognizer, and existing validation rules assign CONFIRM, REJECT or REVIEW. Ambiguous candidates go directly to REVIEW. No generative model or LLM adjudication is used.
4. **Engine 4 — Text Redaction:** replaces confirmed page-local spans with deterministic tags, retains audit records, resolves contained overlaps and blocks export for unresolved findings.
5. **Engine 5 — Audit & Human Review:** records detection/redaction history, exposes review queues, accepts explicit reviewer corrections and regenerates text through Engine 4. Local JSON snapshots preserve audit metadata and feedback.

```text
Extracted pages → Presidio + custom recognizers → validation rules
               → CONFIRM / REJECT / REVIEW → redaction → audit and human review
```

```python
from app.engines.pii_detection import PIIDetectionEngine
from app.engines.redaction import RedactionEngine

detections = PIIDetectionEngine().detect(extraction_result)
redaction = RedactionEngine().redact(extraction_result, detections)
```

`detections`, `rejected` and `review` account for each returned Presidio candidate exactly once. Records retain original text, page-local offsets, recognizer, original confidence, validation decision/reason and final disposition. `AMBIGUOUS` remains an internal validation state and maps directly to final REVIEW. Deprecated optional LLM audit fields remain for historical records; new detection results leave them unset.

REVIEW and unresolved crossing overlaps block `to_text_payload()`. Successful replacement does not prove full-document PII coverage: `safe_for_llm` remains false. The full result contains sensitive internal audit data and must not be sent downstream.

## Setup and verification

Use Python 3.13 x64 on Windows. The default Presidio analyzer uses the pinned English spaCy model; scanned/hybrid PDFs also require Poppler on PATH. The application serves its committed React bundle, so Node is needed only for frontend development. No database or local LLM service is required.

From Windows Command Prompt in the repository root, create `.venv` only if it does not already exist:

```cmd
py -3.13 -m venv .venv
call .venv\Scripts\activate.bat
python -m pip install -r requirements.txt -r requirements-models.txt
python -m pip check
python run.py
```

Open `http://127.0.0.1:8765/`. For regression tests, install `requirements-dev.txt` and run `python -B -m pytest tests -q -p no:cacheprovider`. See [Windows CMD setup](docs/WINDOWS_CMD_SETUP_GUIDE.md) and [runtime verification results](docs/WINDOWS_RUNTIME_FIX_REPORT.md).

`requirements.txt` contains application dependencies; `requirements-dev.txt` adds test tooling, and `requirements-models.txt` pins `en_core_web_lg`. The original mixed dependency snapshot is retained at `docs/dependencies/requirements-original.txt` for historical reference only; its experimental dependencies are not part of the supported installation.

For the cached real-document diagnostic, with the existing access-restricted `.private_validation` directory:

```powershell
.venv/Scripts/python.exe -u tools/validate_cadence_document.py --cached-extraction
```

This reuses trusted extracted pages and runs current deterministic detection/redaction without repeating OCR. New results are stored separately from historical results. Keep `.private_validation` gitignored and restricted to the current Windows user and SYSTEM. Cached text and audit records contain original PII.

See [ENGINE3_DETERMINISTIC_REPORT.md](ENGINE3_DETERMINISTIC_REPORT.md) for the current implementation and results. Earlier engine and real-document reports describe previous experiments and are retained as historical evidence.

## Audit and reviewer corrections

```python
from app.engines.audit import AuditEngine, ReviewerFeedback

engine = AuditEngine()  # default private local store: .private_audit/
audit = engine.create_audit(extraction_result, detections, redaction)
queue = engine.get_review_queue(audit)

corrected = engine.apply_feedback(audit, ReviewerFeedback(
    action="FALSE_POSITIVE",
    reviewer_id="caller-supplied-reviewer-id",
    detection_id=audit.records[0].detection_id,
    reason="Reviewer explanation",
))
engine.save_audit(corrected)
reloaded = engine.load_audit(corrected.document_id)
summary = corrected.sanitized_summary()
```

Supported actions are `CONFIRM_PII`, `FALSE_POSITIVE` and `MISSED_PII`. A missed finding supplies page number, original start/end offsets and entity type; its text is sliced from the original extraction. Corrections always regenerate from that extraction and leave caller inputs unchanged. Reversing a previous reviewer decision requires `expected_decision` to identify the current state. Exact retries are idempotent.

Saving over an existing different snapshot requires its `expected_revision`; for example, `save_audit(corrected, expected_revision=0)`. JSON writes are atomic and private storage receives restricted Windows ACLs or POSIX permissions plus a local `.gitignore`. Configure a dedicated directory using `AuditEngine(storage_dir=...)`; avoid shared/public directories. Snapshots contain original PII and are not encrypted or a multi-user production database. Authentication and reviewer authorization belong to the caller's application.

`human_review_complete` means no known unresolved review issues remain. It does not establish coverage. `coverage_verified`, `downstream_export_permitted` and `safe_for_llm` remain false; there is no approval API that turns them on. Use only `sanitized_summary()` for display outside the internal review boundary.

The complete cached-document audit diagnostic requires no OCR or new detection run:

```powershell
.venv/Scripts/python.exe tools/validate_audit_cache.py
```

See [ENGINE5_REPORT.md](ENGINE5_REPORT.md) for Engine 5's data model, conflict policy, persistence, test evidence and limitations.

## End-to-end pipeline

```python
from app.pipeline import DocumentPipeline
from app.engines.audit import ReviewerFeedback

pipeline = DocumentPipeline()
result = pipeline.process("path/to/document.pdf")
if result.failure:
    print(result.sanitized_summary())
else:
    summary = result.sanitized_summary()  # Only this view is suitable for external display.
    queue = result.review_queue
    preview = result.internal_preview  # Authorized internal review only.
    # Use an actual audit record ID selected by the reviewer.
    updated = pipeline.apply_feedback(result, ReviewerFeedback(
        "CONFIRM_PII", "caller-supplied-reviewer-id", result.audit_result.records[0].detection_id,
    ))
    saved = pipeline.save_audit(updated)  # Explicit; check saved.failure before proceeding.
    loaded = pipeline.load_audit(updated.document_id)
```

`process()` runs File Intelligence, Extraction, Detection, Redaction and Audit synchronously, followed by private artifact generation and verification for PDF, DOCX and PPTX. PDF extraction retains its native/OCR/hybrid routes. Standalone images remain unsupported by extraction.

`PipelineResult` retains source identity and each engine result in memory. `detection_result` always holds the original automated findings; `effective_detection_result` provides the audit's corrected findings. Feedback returns a new result, preserves the original extraction/detections, updates history/queue/redaction, and does not repeat classification, extraction or Presidio. Engine 5 revalidates its baseline and performs redaction regeneration.

Failures return `PROCESSING_FAILED` with a stage, fixed code and sanitized message; downstream stages stop. Successful outcomes are `PROCESSED_REVIEW_REQUIRED` or `PROCESSED_NO_KNOWN_REVIEW`. Coverage remains `COVERAGE_UNVERIFIED`; `safe_for_llm` and `downstream_export_permitted` always remain false. Preview, source text, paths, original findings and reviewer history are sensitive internal data.

Audit persistence delegates to Engine 5's existing store and remains explicit. `process()` and feedback do not automatically save audit snapshots; they write private review artifacts. `save_audit()` returns a new result with `persisted_path` or a structured failure; overwriting a changed snapshot requires `expected_revision`. `load_audit()` rebuilds review state without source extraction/detection and regenerates artifacts from the fingerprinted original. File Intelligence metadata and original processing timings are not part of the audit snapshot and are absent on reload. Stage timings include lazy initialization and the latest feedback/artifact operations; total runtime accumulates operations on that result branch.

See [PIPELINE_REPORT.md](PIPELINE_REPORT.md) for test evidence, the format matrix, cached Cadence results and limitations.

## DOCX and PPTX review artifacts

```python
pipeline = DocumentPipeline(artifact_output_dir=".private_artifacts")
result = pipeline.process("path/to/document.docx")  # .pptx also supported
if result.failure is None:
    artifact = result.artifact_result
    private_review_path = artifact.path  # Internal use only; may retain unresolved PII.
    diagnostics = artifact.sanitized_summary()
```

DOCX paragraphs, nested tables, and shared headers/footers map to logical paragraph segments, not physical pages. PPTX text boxes, placeholders, recursive groups, tables and accessible speaker notes map to slide-local paragraphs. Text-node mappings retain exact offsets across formatting runs. Office extraction uses the existing `python-docx` and `python-pptx` dependencies; no image OCR is performed.

Artifacts copy the original OOXML package and change only reliably mapped text nodes according to Engine 4's current tags. Feedback always creates a new artifact from the unchanged source; it does not reverse a prior redacted copy or repeat source extraction/Presidio. A SHA-256 source check blocks generation if the original changed. Audit snapshots retain the structural mappings for reload.

Outputs use random `review_...` names in a gitignored directory secured through Engine 5's existing directory guards/permissions. `OfficeArtifactEngine` reopens and independently extracts each output, checks expected text/tags, verifies package structure and unchanged parts, and counts remaining confirmed values and email/employee-ID/SSN patterns. Unsupported parts and attributes remain in the private review copy and are explicitly reported.

`redaction_completed` means the artifact generation operation ran, including partial review copies. `verification_completed` means checks ran, not that they passed. `artifact_verified` retains the stricter Phase 7 gate: no mapping/check failures, residual matches, unsupported structures or known review issues. `supported_content_verified` reports technical verification of the mapped confirmed replacements. Neither permits export. PPTX spans can cross adjacent paragraphs in the same text frame/table cell when the mappings prove correspondence. Generated structural newlines remain intact; unrelated shapes/cells are never merged. Unreliable spans remain untouched with internal `mapping_issues`, so the text preview can differ from the artifact.

Metadata, text boxes, content controls, tracked changes, dynamic fields, comments/footnotes, charts, embedded objects, alternative text, external relationships, images, master/layout text and other unmapped content are unverified. No rendering or visual layout guarantee is made. `safe_for_llm`, `coverage_verified` and downstream export remain false.

Run the supplied-document diagnostic locally:

```powershell
.venv/Scripts/python.exe -u tools/validate_office_documents.py
# Evaluate just one format while retaining the other saved summary:
.venv/Scripts/python.exe -u tools/validate_office_documents.py --format pptx
```

See [PHASE7_REPORT.md](PHASE7_REPORT.md) for measured Cadence results and remaining limitations.

## PDF and common artifact results

```python
pipeline = DocumentPipeline(artifact_output_dir=".private_artifacts")
result = pipeline.process("path/to/document.pdf")
if result.failure is None:
    artifact = result.artifact_result
    private_path = artifact.output_path
    diagnostics = artifact.sanitized_summary()
    # Same basic fields for PDF, DOCX and PPTX:
    # format, redaction_completed, verification_completed, reopened,
    # supported_content_verified, mapping_issue_count, verification_failure_count
```

PDF generation uses PyMuPDF destructive text redaction with garbage-collected saves for reliably aligned native glyph coordinates. Optional visible tags are inserted within the removed region when they fit. Scanned pages use the trusted OCR block coordinates/resolution: corresponding source pixels are removed, and a fresh page embeds only the edited raster. The retained non-sensitive OCR block transcription is redrawn with tags where space permits. This changes the typography inside patched blocks; all other raster pixels remain intact. Unicode transcription uses an embedded font. Hybrid documents follow each extracted page's route; images on native-text pages remain unverified.

PDF verification reopens the output, checks underlying selectable text, rerenders every page, validates dimensions/page count, compares rendered output to generation evidence and verifies the edited image payload on scanned pages. Source fingerprints and DPI metadata are retained in new extraction/audit results. Legacy PDF audits lacking this metadata fail closed at artifact generation; trusted caches require explicit validated metadata attachment, as demonstrated by the local diagnostic tool. No OCR or Presidio runs during feedback regeneration.

Outputs remain internal review artifacts. Image/OCR accuracy, metadata, unresolved review findings and complete PII coverage are not cleared. Inspect both the audit queue and artifact diagnostics; `supported_content_verified=True` is not downstream approval.

```powershell
.venv/Scripts/python.exe -u tools/validate_phase8_documents.py
# Repeat only the cached PDF artifact evaluation, without OCR/Presidio:
.venv/Scripts/python.exe -u tools/validate_phase8_documents.py --format pdf
.venv/Scripts/python.exe -m pytest tests -q
```

See [PHASE8_REPORT.md](PHASE8_REPORT.md) for the final three-format findings and verification evidence.

## React privacy review application (Phase 10B)

The primary UI is now React + TypeScript + Vite, with Tailwind, selected shadcn components, and MIT Bklit funnel/ring components. FastAPI and Engines 1–5 remain the processing backend. No authentication, database, cloud service, LLM, or automatic learning was introduced.

### Build and launch

For normal execution, use the Python setup above and the committed frontend bundle:

```cmd
call .venv\Scripts\activate.bat
python run.py
```

Open **http://127.0.0.1:8765**. `run.py` is unchanged: loopback only, access logs disabled, four-connection limit. Restart an already-running server to load the additive bounded comparison route and restored-audit serialization fix. In-memory review sessions are temporary; save desired audits before a normal restart. Built React files are served from `app/web/static/`; Vite replaces that directory cleanly, so no vanilla production UI remains. No npm dev server is needed after building.

For frontend development, keep the backend on 8765 and in a second terminal run:

```cmd
cd frontend
npm ci
npm run dev
```

Open **http://127.0.0.1:5174/assets/**. The Vite proxy translates only this trusted local browser Origin for the existing API boundary; foreign Origins are not rewritten. The backend still requires `X-Cadence-Local: 1`. Frontend JSON requests are serialized to respect the existing synchronous local launcher rather than bursting concurrent requests. An intentional `npm run build` replaces `app/web/static`; it is not needed for normal startup.

### Product workflow

1. **Overview:** upload PDF, DOCX or PPTX using the picker/drop area (30 MiB API limit). While processing, the UI shows “Processing document…” without invented stage progress. Completed stages show actual timings.
2. Read the pipeline, Bklit disposition funnel, entity rings and known-review completion. Click any disposition/entity/location to filter the review workspace. DOCX locations are **logical segments**, not physical pages. The funnel outcomes are branches, not sequential conversions.
3. **Review Queue:** inspect paginated metadata and bounded highlighted context. Open **Review finding**, provide a reviewer identifier/optional reason, then **Confirm as PII** or **Mark False Positive**. The backend changes effective decisions and regenerates artifacts without rerunning Presidio. Revision conflicts refresh current state and require reopening the finding.
4. **Find missed PII:** choose a page/slide/logical segment, select up to 1,000 existing source characters, and submit **MISSED_PII**. The UI submits Python-codepoint offsets/entity/reviewer/reason, never selected values or arbitrary replacement text.
5. **Artifact verification:** view generated/reopened/mapping/failure checks separately from coverage. Download only via an opaque session endpoint. PDF has a current inline browser preview; DOCX/PPTX use bounded extracted text and actual downloads rather than a simulated Office renderer.
6. **Audit:** inspect the graphical decision timeline and **Save audit** explicitly. **Documents** lists opaque saved references for restoration; the retained original must remain available.
7. **Detector Quality:** compares policies 1.0/1.1 using only aggregate values exported from existing synthetic evaluation artifacts. The visible disclaimer excludes real Cadence-document accuracy claims.

**Artifact supported-mapping verification can pass while PII coverage remains NOT ESTABLISHED and downstream submission remains BLOCKED.** `safe_for_llm`, `coverage_verified`, and `downstream_export_permitted` remain false. Known-review resolution is not a safety score. Original uploads/artifacts/audits remain local plaintext under existing restricted private-directory permissions, not an encryption claim.

### Checks

```cmd
call .venv\Scripts\activate.bat
python -m pip install -r requirements-dev.txt
set "PYTHONDONTWRITEBYTECODE=1"
python -B -m pytest tests -q -p no:cacheprovider
node --test tests\frontend\core.test.mjs
cd frontend
npm test
npm run lint
npx tsc --noEmit
cd ..
rem Optional real Chrome acceptance against the running backend and fictional fixtures:
python -B tools\create_phase10b_fixtures.py
set "CADENCE_UI_URL=http://127.0.0.1:8765"
set "CADENCE_UI_FIXTURE=%CD%\.private_review_app\phase10b-fixtures\synthetic.pdf"
set "CADENCE_UI_SYNTHETIC=1"
node tests\frontend\phase10b_browser.mjs
```

Browser evidence/downloads/profiles remain in `.private_review_app/phase10b-ui/`. The default browser acceptance uses the supplied PPTX and records an explicit demonstration override on an uploaded copy; it is not ground truth. `tests/frontend/browser_smoke.mjs` forwards to the React acceptance script. `tools/export_detector_quality.py` regenerates the small aggregate UI metadata without changing the evaluation artifacts or detector.

See [PHASE10B_REPORT.md](PHASE10B_REPORT.md) for architecture, component provenance, test/browser evidence and the 10–15 minute demo. [PHASE9_REPORT.md](PHASE9_REPORT.md) is historical: its vanilla UI/file commands are superseded.

## Detector evaluation and offline feedback curation

Detector policy `1.1` adds conservative format and adjacent-context checks. The public
synthetic dataset contains 420 labeled cases with a seeded development/held-out split.
Restart an already running reviewer application to load the new detector policy.
Scoring uses exact source offsets and entity types; REVIEW contributes only to separately
reported potential human-assisted recall. It never counts as an automated true positive.

```powershell
.venv/Scripts/python.exe tools/evaluate_detector.py --output my_detector_evaluation.json
.venv/Scripts/python.exe -m pytest tests -q
# Explicit private feedback export; fill in an existing audit's internal identifier:
.venv/Scripts/python.exe tools/export_reviewer_feedback.py --audit-store .private_audit --document-id <document-id> --output-directory .private_feedback/new-export
```

Evaluation refuses to overwrite existing evidence. Baseline JSON/Markdown were saved
before hardening; source fingerprints identify the policies used. Real-document tools
keep source text and changed-candidate evidence inside `.private_detector_eval/` and reuse
the trusted PDF OCR cache. Existing baseline snapshots are required for comparisons.

Feedback exports include only explicit reviewer events and metadata by default. Use
`--include-context` only when allowing sensitive bounded context in private local output.
Historical audits have no recorded policy version: the default is `unknown-legacy`, and
an explicit `--detector-version` should identify their actual original policy. Exports
require curation and never update production rules. Demonstration overrides are not
ground truth. No coverage or downstream clearance flags changed.

See [PHASE10A_REPORT.md](PHASE10A_REPORT.md) for measured results and limitations.

## Local visual privacy (Phase 11)

The existing pipeline now inspects supported document rasters and embedded PNG/JPEG
media with a checksum-pinned local CPU YuNet model. Anonymous FACE / BIOMETRIC
findings are separate from textual Presidio metrics. Mapped candidates scoring at
least 0.85 are policy-confirmed; emitted candidates from 0.35 require review otherwise.
Confirmed faces receive strong pixel blur in freshly generated PDF/DOCX/PPTX review
artifacts. Reused Office media receives the same protection at every placement.

The review application shows visual counts, bounded before/blurred-after crops,
occurrence locations, and independent serialized transformation verification.
Confirm/reject regenerates from the original source without rerunning detectors.
Unsupported visual content remains visible and blocks visual verification.
All coverage and downstream clearance flags remain false. Save pending reviews
before restarting an existing app process to load the new backend modules.

See [PHASE11_REPORT.md](PHASE11_REPORT.md) for provenance, thresholds, format support,
measured real-document diagnostics, test commands, and limitations. Synthetic fixtures
can be generated locally with `.venv\Scripts\python.exe tools/create_phase11_fixtures.py`.
