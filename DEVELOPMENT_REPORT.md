# Cadence PII Guard — Development and Engineering Report

Consolidated 15 October 2026 from the working checkout of commit `1a377f8` (`V1`), branch `main`. This report and [README.md](README.md) are the two authoritative first-party project documents. Required third-party attribution exceptions remain separately at their original paths.

Evidence labels used here: **current implementation** means inspected source/configuration; **historical measured** means a dated report or retained JSON result, not a newly repeated experiment; **current verification** means executed during this documentation-consolidation task. A historical phase's “complete” describes its stated scope, not complete PII coverage or production certification.

## 1. Executive summary

Cadence PII Guard, developed in the Optiv project context, is a trusted local document-review application. It identifies and extracts PDF/DOCX/PPTX content, routes text through Presidio plus deterministic validation, inspects supported raster media with YuNet, regenerates redacted review artifacts, and records explicit human corrections and saved JSON audits.

The runtime is one Python/FastAPI/Uvicorn process serving a committed React bundle. An optional Vite process supports frontend development. Processing engines run in-process; there is no database service, queue, worker, cloud inference integration or downstream submission feature. Persistent original uploads are necessary for saved-audit artifact regeneration.

The 15 October stabilization demonstrated actual Windows engine, HTTP, Chrome and restart/reload workflows, with 302 Python, 40 React and four Node helper tests passing. Current consolidation regressions are separately recorded in section 15. **Local runtime readiness does not establish production security or universal document sanitization.** All coverage/downstream clearance flags remain false.

## 2. Problem statement and requirements

The repository's case-study narrative concerns business document-analysis agents and a policy restricting PII from AI processing. Relevant artifacts include scanned policy PDFs, training/process DOCX files and organizational presentations. Intended users are local authorized reviewers and developers who need evidence, correction and traceability before any separate downstream decision.

Implemented requirements are content-based identification, supported-format extraction, local OCR/NLP/face processing, exact-source redaction, explicit uncertainty handling, human correction, artifact verification, evidence/history, and explicit save/reload. Privacy requirements include keeping original values inside the review boundary, restricting filesystem access, rejecting stale/invalid operations, and separating technical transformations from coverage assurance.

The source implements an inspection/review tool; a connected AI-agent gateway, organization-wide policy enforcement, authenticated multi-user deployment, complete PII discovery and automatic clearance are absent. Completing the known queue is an operational milestone, not a safety attestation. No invented percentage of project completion or real-document accuracy is used.

## 3. System architecture

### Components and communication

`run.py` invokes Uvicorn with `app.main:app`, loopback `127.0.0.1:8765`, disabled access logs, warning log level, disabled proxy-header trust and a four-connection concurrency limit. `app/main.py` calls `app.web.server.create_app()`. The factory prepares private storage, attaches the local HTTP boundary, mounts `/assets`, serves `/`, and creates a lazy reusable `DocumentPipeline`.

Normal browser requests are same-origin. React's `lib/api.ts` attaches `X-Cadence-Local: 1`, uses same-origin credentials and serializes JSON requests to respect the launcher limit. Multipart uploads use FormData without a JSON Content-Type. Optional Vite at `127.0.0.1:5174/assets/` proxies `/api` to 8765 and translates only the exact trusted Vite Origin.

```text
Browser / committed React bundle
    -> LocalBoundary -> typed FastAPI routes and revision checks
        -> DocumentPipeline (synchronous; reusable lazy engines)
           1 file intelligence
           2 extraction (native / OCR / hybrid PDF, DOCX, PPTX)
           3 face inspection
           4 deterministic text detection
           5 text redaction
           6 audit construction
           7 artifact generation
           8 artifact verification
        -> in-memory sessions; explicit private JSON snapshots
        -> bounded evidence endpoints and private review downloads
```

The API display-stage list places text detection before face inspection, but actual execution in `DocumentPipeline.process()` runs face inspection first. The visual module supplements the historical five-engine organization; it is not a separate daemon.

### Orchestration and state contracts

`DocumentPipeline.process(path)` returns `PipelineResult`. It retains source identity and each engine result, original automated detections, effective corrected detections, internal preview, review queue, stage timings, accumulated runtime, optional artifact, optional persistence path and failure. Full results contain sensitive material; `sanitized_summary()` is the allowlisted aggregate interface.

The classification adapter supplies the existing File Intelligence result to default extraction, avoiding a redundant classification. Engines initialize on first use and are reused. Creating `ExtractionEngine` initializes its OCR component even when the selected format needs no OCR inference.

Stages stop after failure or an unexpected result type. Codes distinguish `SOURCE_NOT_FOUND`, `UNSUPPORTED_EXTRACTION` and generic `STAGE_FAILED`; the orchestrator avoids exposing vendor exception text. An HTTP 200 upload response can contain a failed processing state, so inspect `summary.failure` and stage states.

Feedback returns a copied result, validates/replays audit decisions, regenerates redaction/artifacts from original evidence, and does not rerun source extraction or either detector. Timings include lazy initialization; later operations replace their own stage timing and add to branch runtime. Reload records load/regeneration timing rather than reconstructing original processing times or File Intelligence metadata.

| State/property | Meaning |
| --- | --- |
| `PROCESSING_FAILED` | A processing, feedback, persistence or artifact operation failed. |
| `PROCESSED_REVIEW_REQUIRED` | Processing returned with known queue items or a non-verified artifact. |
| `PROCESSED_NO_KNOWN_REVIEW` | No known queue item/artifact issue requires review under current checks. |
| `human_review_complete` | No known unresolved audit queue items, not independent coverage review. |
| `supported_content_verified` | Required supported mapping/text/structure checks passed. |
| `artifact_verified` | Stricter technical gate also requiring no mapping/check failures, residual diagnostics, unsupported structures or known review issues, with applicable visual verification. |
| `COVERAGE_UNVERIFIED` | Independent coverage state. |
| `coverage_verified`, `safe_for_llm`, `downstream_export_permitted` | Always false; no enabling action or approval API. |

### Persistence and trust boundaries

The web factory owns `.private_review_app/uploads`, `artifacts` and `audits`. Sessions and saved-reference dictionaries are process-local. File operations use restricted directories and generated identifiers; the browser does not supply arbitrary local paths. An RLock serializes shared operations. Multiple workers would have independent sessions/locks and are not a supported deployment configuration.

Original uploads, extraction/audit snapshots, source windows, review reasons, image crops and partial artifacts remain sensitive. Opaque IDs are locators, not authenticated identities. There is no external submit action, user database, broker or migration path.

## 4. Repository structure

| Path | Responsibility |
| --- | --- |
| [run.py](run.py), [app/main.py](app/main.py) | Launcher and ASGI entry point. |
| [app/web](app/web) | Factory, HTTP boundary/routes, Pydantic schemas, committed static UI. |
| [app/pipeline](app/pipeline) | Synchronous orchestration, typed results/statuses/failures. |
| [app/engines/file_intelligence](app/engines/file_intelligence) | libmagic/OOXML identification and PDF classification. |
| [app/engines/extraction](app/engines/extraction) | Native/OCR/Office extractors and source mappings. |
| [app/engines/pii_detection](app/engines/pii_detection) | Presidio, employee recognizer, deterministic policy 1.1. |
| [app/engines/visual_detection](app/engines/visual_detection) | Image/media inspection, YuNet, face models and attribution. |
| [app/engines/redaction](app/engines/redaction) | Tags, overlap resolution, Office/PDF/visual artifacts and proof checks. |
| [app/engines/audit](app/engines/audit) | Immutable baseline/effective review state, events, storage, summary and feedback export. |
| [frontend/src](frontend/src) | React product, hooks, API/types/review helpers, selected reusable UI/chart components and Vitest tests. |
| [tests](tests) | Python regressions, synthetic datasets/fixtures and Chrome/Node harnesses. `test_pipeline.py` remains an empty historical placeholder; orchestration tests are in `test_document_pipeline.py`. |
| [tools](tools) | Synthetic fixture generation, runtime checks, evaluation and optional historical diagnostics/report generation. |
| [requirements.txt](requirements.txt), [requirements-dev.txt](requirements-dev.txt), [requirements-models.txt](requirements-models.txt) | Runtime, test and pinned model installation inputs. |
| [constraints-windows-py313.txt](constraints-windows-py313.txt), [.python-version](.python-version) | Exact resolved versions and selected Python minor. |
| [docs/dependencies/requirements-original.txt](docs/dependencies/requirements-original.txt) | Original mixed 161-package snapshot; provenance only, not an installation input. |
| Root evaluation/diagnostic JSON | Retained detailed machine-readable historical evidence, not runtime configuration or current accuracy estimates. |
| `scratch_*.py`, `createfile.py` | Experiments with old paths/dependencies; not launchers. `createfile.py` overwrites a tracked fixture and is not a harmless smoke test. |

`app/config/settings.py` is an empty placeholder; there is no dotenv settings layer. Several empty historical engine placeholders remain, including `pii_detection/presidio.py` and redaction `text.py`/`document.py`; the implementations are in the substantive modules. Tracked `.pyc` files exist and must not be mistaken for dependencies or fresh verification. This consolidation intentionally does not remove non-Markdown files.

### Documentation inventory and consolidation decisions

Inventory was captured **before rewriting/deleting documentation**: **24 Markdown documents, 381,434 bytes**. Scope excluded `.git`, virtual environments, node_modules, build outputs, generated caches/private application data and vendor directories. All 24 documents were read; the three generated evaluation bodies were parsed and every embedded JSON key/value was verified identical to its retained `.json` counterpart. Private inventory records retain original paths, hashes, content, purpose, unique/duplicated/outdated information, destination and removal decision under `.venv/documentation-consolidation`.

The table is also the deletion manifest. “Remove” means relevant content was consolidated here or in README before deletion. Historical names are provenance records, not links to missing files.

| Original path | Purpose / unique content preserved | Duplicate or outdated information reconciled | Disposition and destination |
| --- | --- | --- | --- |
| `README.md` | Product, public APIs, installation and review semantics | Repeated engine/phase material and obsolete phase links | Rewritten; current practical guide. |
| `CADENCE_REAL_DOCUMENT_VALIDATION.md` | Initial 35-page Qwen diagnostic, residuals and timings | Retired model workflow; no current private-input availability claim | Remove; sections 5, 15, 18. |
| `ENGINE3_REPORT.md` | Candidate-loop/empty-page repair, strict resolver handling, 141 tests | Qwen and resolver files are absent now | Remove; sections 5, 8, 16. |
| `ENGINE3_DETERMINISTIC_REPORT.md` | Model removal, explicit REVIEW, 117 tests, cached dispositions | Engine 5 was future work then; dependency counts superseded | Remove; sections 5, 8, 14, 15. |
| `ENGINE4_REPORT.md` | Exact tags, overlap/export semantics and upstream defect observation | Reported upstream failures and absent visual artifacts were historical | Remove; sections 5, 10, 16. |
| `ENGINE5_REPORT.md` | IDs, feedback conflict/idempotency, ACL privilege repair, 164 tests | No web/visual workflow at that phase | Remove; sections 5, 11, 17. |
| `PIPELINE_REPORT.md` | Phase 6 orchestration, timing/failure contracts, 192 tests | DOCX/PPTX unsupported and no upload boundary were historical | Remove; sections 3, 5, 15. |
| `PHASE7_REPORT.md` | Office maps, virtual placeholders, partial copies, old deck failures | Two PPTX failures resolved later; unsupported-content caveats retained | Remove; sections 7, 10, 15, 16. |
| `PHASE8_REPORT.md` | Destructive/raster PDF proof, PPTX boundary fixes, 233 tests | PDF applied count later changed with policy 1.1 | Remove; sections 5, 10, 15. |
| `PHASE9_REPORT.md` | First local API, Unicode selection, 252 tests, vanilla UI history | Vanilla assets and no-npm instructions superseded | Remove; sections 5, 12, 13, 17. |
| `PHASE10A_REPORT.md` | Dataset/splits, exact metrics, hardening, partial-email risk and feedback export | Metrics duplicated in JSON; “no .git” described a different workspace | Remove; sections 5, 8, 11, 15. |
| `PHASE10B_REPORT.md` | React migration, vendor boundaries, CSP/connection/serialization fixes | Old preview ports/paths, process state and 19-test React count | Remove; sections 5, 13, 15, 16. |
| `PHASE10B1_REPORT.md` | First/next review selection and prominent download, 30 React tests | Old bundle sizes and live-process statements | Remove; sections 5, 13, 16. |
| `PHASE10B2_REPORT.md` | Purpose/AI-boundary UX, truthful effective decisions, 36 React tests | Repeated metrics/warnings and old handoff process state | Remove; sections 5, 13, 15. |
| `PHASE11_REPORT.md` | Face algorithms/proof/inventory, provenance, memory failure, 302/40/4 results | OpenCV 4.10 was that environment; current packaging validates 5.0 | Remove; sections 5, 9, 15, 18. |
| `detector_eval_baseline.md` | Policy 1.0 generated exact metrics | Structured body exactly duplicated in retained JSON | Remove; section 15 and `detector_eval_baseline.json`. |
| `detector_eval_development.md` | Intermediate 299-case development evaluation | Structured body duplicated; not frozen held-out certification | Remove; section 15 and `detector_eval_development.json`. |
| `detector_eval_hardened.md` | Policy 1.1 aggregate/split results | Structured body exactly duplicated in retained JSON | Remove; section 15 and `detector_eval_hardened.json`. |
| `docs/REPOSITORY_RUNTIME_AUDIT.md` | Pre-fix install failures, architecture and security observations | NOT READY superseded by stabilization; attribution file exists in this checkout; Faker was not demonstrated as a required tool import | Remove; sections 5, 14, 16–18. |
| `docs/WINDOWS_CMD_SETUP_GUIDE.md` | Working CMD setup, models, Poppler, one-process startup and troubleshooting | Separate guide duplicates README | Remove; README sections 5–12. |
| `docs/WINDOWS_RUNTIME_FIX_REPORT.md` | Actual dependency/native/engine/API/browser/restart evidence and ASCII log repair | Historical “server left running” is not a present-state guarantee | Remove; sections 5, 14–16. |
| `tests/fixtures/visual/README.md` | Fictional image generation date, origin and test-only scope | Provenance repeated in face report | Remove; exact substantive provenance moved to section 9. PNG retained. |
| `frontend/THIRD_PARTY_NOTICES.md` | Upstream commits, copied/adapted files, MIT/Studio boundaries | Some history repeats provenance; obligations remain | **Retain unchanged: required third-party attribution exception.** |
| `app/engines/visual_detection/assets/ATTRIBUTION.md` | Adaptation/upstream paths, model origin/hash/license | Earlier audit's missing-file observation is not current | **Retain unchanged: required model/code attribution exception.** |

Final intended scope: README and this report, plus the **two explicit attribution exceptions**. Twenty-one redundant Markdown files are removed. `docs` remains because its `dependencies/requirements-original.txt` is non-Markdown provenance that must be preserved. No source/test/configuration, JSON, fixture, model or license text is deleted. Original reports can also be retrieved without changing branches using, for example, `git show 1a377f8:PHASE11_REPORT.md`.

## 5. Development history

### Chronology and evidence limits

Available Git history is sparse: root snapshot `655c6fa` (“Last minute fix”, 12 October 2026) already contains the major engine/phase reports and implementation; `316d3cc` (15 October) excludes local environment files; `1a377f8` (“V1”, 15 October) records runtime stabilization and documentation. It does not provide separate commits for every reported phase. The sequence below reconstructs logical development from dated report contents and current source, without inventing individual commit dates or treating old workspace statements as current facts.

Most engine/phase reports explicitly date their measured work to **12 October 2026**. Subphases without an independent date are ordered by their documented dependencies, not assigned a new timestamp. The runtime audit/stabilization is dated **15 October 2026**.

| Milestone | Objective, approach and challenge/resolution | Historical verification and resulting capability |
| --- | --- | --- |
| Early file intelligence/extraction | MIME/PDF page inspection, pypdf plus RapidOCR/Poppler; early scratch alternatives explored Docling/Paddle | Current modules and five early extraction/file tests are present; no separate comprehensive origin report or earlier commit is available. |
| Engine 4 text redaction | Exact offsets, deterministic tags, conservative overlaps and export blocking; integration exposed an upstream dropped-candidate defect | 44 unit cases; 51 combined passes, but one Engine 3 regression still failed. Redaction worked for supplied findings, not whole-document coverage. |
| Engine 3 repair with local Qwen | Move disposition inside page/candidate loops; handle empty candidates and REVIEW separately; preserve confidence/provenance; strict parsing and lazy local model load | 141 tests. CPU Qwen3-1.7B loaded in 4.88 s; three short calls took 1.76/1.40/1.31 s and overconfirmed a context-free name. This is a retired experiment. |
| Initial real-policy diagnostic | Process 35 image-only pages; checkpoint OCR after console-encoding interruption; retain sensitive evidence privately | 1,418 raw candidates, 675 confirmed/743 rejected, 637 replacements; three crossing-overlap groups and residual PII blocked export. Detailed historical evidence in section 15. |
| Deterministic Engine 3 | Remove ambiguous-case generative adjudication and its dependency closure; map AMBIGUOUS directly to REVIEW | 117 tests; same cached 1,418 candidates became 578 confirmed/668 rejected/172 REVIEW, with 577 replacements. Qwen is absent from current source. |
| Engine 5 audit | Immutable baseline/current state, stable IDs, feedback replay and restricted atomic JSON; DACL-only setup fixed `SeSecurityPrivilege` failures | 164 tests and cached audit of all 1,418 records. Review corrections/persistence added without approval/learning. |
| Phase 6 pipeline | Thin lazy synchronous orchestration, stage failures/timings, explicit save/load, no upstream rerun after feedback | 192 tests; native PDF integration. Office extraction was unsupported at this point, not now. |
| Phase 7 Office | Exact run/paragraph maps, nested structures, original-package edits; reject virtual empty placeholder mappings | 217 tests. DOCX mapped successfully; PPTX retained two unmappable confirmed spans pending the next fix. |
| Phase 8 artifacts | Destructive native/raster PDF edits and independent proof; prove same-frame/cell PPTX newline boundaries | 233 tests; historical supported replacements 577/29/25 for PDF/DOCX/PPTX, zero required mapping failures. |
| Phase 9 local web | Same-origin FastAPI plus vanilla UI, safe upload boundary, bounded evidence, Unicode source selection, audit/download workflow | 252 Python and four Node tests; actual PPTX demonstration, PDF preview and saved audit. Vanilla UI is later superseded. |
| Phase 10A policy 1.1 | Seeded synthetic labels, exact scoring, conservative email/employee/number/person validation and offline feedback curation | 279 Python tests. Synthetic precision/F1 improved with unchanged recall; real review burden did not improve and a partial-email warning increased. |
| Phase 10B React | Replace served vanilla UI with React/TypeScript/Vite and selected licensed Bklit/shadcn components; keep existing engines/APIs | 284 Python, 19 React and four Node tests. CSP-safe dialog styles, serialized requests and restored-action serialization fixed integration failures. |
| Phase 10B.1 | Review shortcuts now select first evidence automatically; direct actions, next finding and prominent current download | 284 Python/30 React; real PDF/DOCX/PPTX review/download checks. Earlier browser tests had missed the navigation gap by manually selecting a row. |
| Phase 10B.2 | Clarify business-artifact purpose, blocked AI boundary, human uncertainty, tangible output and governance trail | 284 Python/36 React; responsive/real-browser checks. Presentation changed, not backend clearance. |
| Phase 11 visual privacy | Adapt local CPU YuNet; map media/placements; blur actual pixels; independently check serialized artifacts and per-occurrence proof | 302 Python/40 React/four Node; synthetic face/text/Office/PDF workflows. Sequential reruns resolved host-memory test interference. |
| Pre-fix Windows audit | Verify source instead of old setup assumptions | Frontend 40+4/build checks passed; full Python install failed on PaddleX/PyYAML and backend lacked `magic`; initial readiness NOT READY. |
| Windows runtime stabilization | Separate real runtime/test dependencies, constrain versions, install the exact English model and one native provider per namespace; ASCII OCR completion message | 302 Python/40 React/four Node, real six-fixture engine/HTTP matrix, both Chrome workflows and save/restart/reload passed. Tested local runtime READY, coverage still unverified. |
| Documentation consolidation | Reconcile history/current behavior, retain evidence/attribution and two authoritative guides | Current verification is recorded in section 15 after deletion/link checks. No processing/dependency/test changes are authorized or made here. |

### Important decisions and abandoned paths

Qwen had exact-parsing/failure safeguards but remained slow and capable of confident mistakes. Removing it made ambiguity explicit, reduced installation/inference complexity and kept human review accountable. Deprecated optional LLM fields remain for old cache/constructor compatibility; their presence is not an active inference path. Torch/Transformers/Accelerate/Tokenizers/Safetensors and inactive Docling model parents were removed during the historical deterministic transition. Shared experimental packages lingered until the later runtime manifest separation.

The five-engine organization was preserved as the product grew. Office/PDF artifacts and the face module extend existing pipeline/audit contracts rather than creating parallel review stores. Exact original-source mappings and conservative failure were selected over fuzzy coordinate/span repair. Review regeneration starts from immutable source evidence so rejected decisions can restore characters without trying to reverse previous tags or blur.

The vanilla frontend was replaced by React while retaining the API/trust boundary. Registry components were copied at pinned revisions with their dependency closure/licensing rather than importing a proprietary studio or moving installer output. The optional Sankey was omitted; disposition branches and location maps communicate the available data. No multi-user/auth/job architecture or downstream AI integration was implemented.

## 6. File intelligence engine

[detector.py](app/engines/file_intelligence/detector.py) calls `magic.from_file(..., mime=True)`. Supported MIME identities include PDF, OOXML Word/PowerPoint, PNG and JPEG. The web layer further restricts uploads to PDF/DOCX/PPTX and requires content to agree with the extension.

Windows libmagic can identify Office packages as ZIP/octet-stream, so ZIP files are inspected using `[Content_Types].xml`, expected main-part content type and XML namespace. Duplicate members, ambiguous supported main parts, missing/bad package XML and ordinary ZIPs disguised as Office files fail rather than trusting a suffix. XML parsing disables external entities/network access.

[pdf_analyzer.py](app/engines/file_intelligence/pdf_analyzer.py) counts text-only, image-only, mixed and blank pages. Its exact current routing is:

- At least one image-only **and one text-only** page: `hybrid`.
- Image-only pages without a text-only page: `ocr`.
- Otherwise: `text`.

Mixed pages have both text and images but are not included in the text-only counter. Thus mixed-only PDFs do not trigger image OCR, and mixed plus image-only pages can route the whole document to OCR. This source-level limitation must not be replaced in documentation with a general claim that every mixed image region is inspected for text.

## 7. Extraction and OCR engine

### PDF

Native extraction uses pypdf, retaining ordered positive page IDs and original extracted character sequences, with page text stripped at its boundaries. `full_text` joins pages with two newlines. Hybrid extraction checks each page's text: nonblank extracted text uses the native route; otherwise the existing single-page OCR route is used.

RapidOCR ONNX Runtime performs actual OCR. Poppler `convert_from_path` renders PDF pages at default **150 DPI**; both `pdfinfo` and `pdftoppm` must be discoverable because the source does not pass `poppler_path`. OCR results retain text, confidence and quadrilateral blocks, joined by newlines. Source SHA-256 and OCR DPI are carried in `PDFSourceMap` for later coordinate and integrity verification. OCR confidence is not calibrated extraction or PII-recall assurance.

Page images use `TemporaryDirectory()` and are cleaned by its context manager; default web startup does not redirect it into private app storage. The README documents optional ACL-protected TEMP/TMP configuration. The runtime-stabilization change makes the completion log ASCII-compatible with redirected Windows output.

### DOCX and PPTX

DOCX enumerates body and nested-table paragraphs plus unique header/footer parts, avoiding duplicates from shared headers. Every `ExtractedPage` is a **logical paragraph segment**, not a Word-rendered physical page. Unsupported textbox/control/tracked-change ancestor structures are not silently treated as mapped body text.

PPTX enumerates slide-local text frames/placeholders, recursive groups, table cells and accessible body speaker notes. One extracted page is one slide; newlines separate mapped paragraphs. Original ZIP XML is checked to avoid assigning a mapping to python-pptx-created virtual empty placeholder paragraphs.

`StructureMap` records source hash, location convention, part name, paragraph XPath, source/extracted offsets, text/control-node spans and location metadata such as shape/group/table coordinates. Hyperlink/run text, tabs and breaks retain exact mapping where supported. Generated paragraph separators have no editable text node. Optional mapping fields preserve existing positional model constructors; absent additive metadata is omitted where needed for old fingerprint compatibility.

Office image text, charts, unsupported package structures and physical layout are not comprehensively extracted. The separate face inspection of supported media does not constitute image-text OCR. Extraction cannot repair a PII span that was never recognized correctly; no hidden normalization changes original offsets.

## 8. PII detection engine

[engine.py](app/engines/pii_detection/engine.py) builds one default Presidio `AnalyzerEngine` and registers `CadenceEmployeeIdRecognizer` once. Nonblank pages are analyzed in English; blank pages are skipped, while nonempty zero-candidate pages count as analyzed. Candidate start/end must be integer, ordered, page-local, end-exclusive and in bounds; confidence must be finite in [0,1], with valid entity/recognizer metadata. Failures raise `DetectionProcessingError`, never a successful zero-PII result.

Candidates are ordered by span/entity, retaining raw duplicates/overlaps. Every returned Presidio candidate receives exactly one final disposition: CONFIRM in `detections`, REJECT in `rejected`, or REVIEW in `review`. `raw_candidate_count == disposition_count` is checked. “Raw” covers `AnalyzerEngine.analyze` output after Presidio processing, not every possible identifier or internal recognizer proposal.

Current [validation.py](app/engines/pii_detection/validation.py) declares **policy 1.1** and 12 entity types:

`PERSON`, `EMAIL_ADDRESS`, `PHONE_NUMBER`, `EMPLOYEE_ID`, `US_SSN`, `US_PASSPORT`, `US_DRIVER_LICENSE`, `US_ITIN`, `US_BANK_NUMBER`, `IBAN_CODE`, `CREDIT_CARD`, `MEDICAL_LICENSE`.

Dates, locations/addresses, organizations, URLs, NRP and other types are outside this redaction policy and receive REJECT when returned. This is a scoped policy, not a complete taxonomy of personal information.

| Candidate type | Current validation behavior |
| --- | --- |
| Employee ID | Case-sensitive `EMP-[0-9]{5}` with boundaries excluding surrounding word/hyphen characters. Injected partial spans become REVIEW; undocumented formats are not repaired/broadened. |
| PERSON | Reject known business labels/segmentable OCR labels, acronyms, email/numeric-only spans and positive institutional suffix evidence. Scores below 0.5, single tokens, unusual casing, newline contamination and uncertain structures go to REVIEW; conservative valid multi-token name structures confirm. Validation-only initial spacing never changes source text. |
| Email | Confirm conservative unquoted mailbox structure; reject repeated/leading dot defects; unfamiliar or suspected partial spans require review. |
| Phone | Preserve recognizer evidence for plausible 7–15-digit/character structure; unfamiliar formats require review. |
| Numeric phone/bank/license/passport | Adjacent explicit money-field plus currency, or a labeled internal/project/release code, can reject business-number interpretations. Distant keywords cannot reject a value. |
| Bank / driver license | Otherwise remain ambiguous/REVIEW whether supporting context exists or not; reason records the difference. |
| Remaining policy entities | Retain the existing Presidio evidence and policy confirmation. |

Validation sees up to 200 original characters on each side plus the candidate. Exact relative context positions prevent repeated values borrowing evidence from their first occurrence. The historical default analyzer score threshold was 0; no new global threshold was added. Presidio confidence is preserved, not recalculated as a validator/model probability.

OCR source-block indices are assigned only if newline-joined blocks exactly reconstruct the page and the candidate is fully contained in one block. Cross-block/unreconstructable references remain absent. No detector discovers additional PII merely because a later validator handles formatting better.

The active workflow has **no generative resolver, remote adjudication or automatic learning**. Optional `llm_decision`/`llm_reason` metadata are legacy compatibility fields, left unset on new automated records. Section 15 records synthetic precision/recall and the material limits of those measurements.

## 9. Visual privacy engine

### Provenance and model

The adapter traces to `ivan-george710/PII-Project` commit `63a416b70cf452527f42cbb1a3d9a4f80f94271f`, specifically its face detector, image redaction reference and licensed YuNet asset. It does not import the other repository's pipeline, database, policy or identity contracts. The reference black-fill redactor was adapted to real pixel blur.

The committed `face_detection_yunet_2023mar.onnx` is **232,589 bytes**, SHA-256 **`8f2383e4dd3cfbb4553ea8718107fc0423210dc964f9f4280604804ed2552fa4`**. It originates from OpenCV Zoo's YuNet (Shiqi Yu), upstream commit `47534e27c9851bb1128ccc0102f1145e27f23f98`. [ATTRIBUTION.md](app/engines/visual_detection/assets/ATTRIBUTION.md) and [YUNET_LICENSE.txt](app/engines/visual_detection/assets/YUNET_LICENSE.txt) remain unchanged. There is no runtime weight-download fallback; unavailable/untrusted models or inference errors fail the visual stage.

### Inspection, policy and transformation

Single-frame PNG/JPEG inputs are bounded to **25 MiB** and **16,777,216 pixels**. The inspection tracks source bytes/pixel hashes, dimensions, Office part or PDF xref/raster reference and placement evidence. A per-inspection image-hash/frame cache avoids repeating inference for reused media, retaining every occurrence. Thumbnails/unplaced parts receive package labels rather than invented page locations.

CPU YuNet runs four rotations and, for large images, nine overlapping tiles plus the full frame. Frames are capped at 1280 pixels, overlapping boxes deduplicated and boxes padded 8% horizontally/12% vertically. Policy `face-1.0` emits from threshold **0.35**; confidently mapped candidates at **0.85 or above** automatically CONFIRM, while lower/uncertain placements require REVIEW. Findings are anonymous FACE/BIOMETRIC regions, with no identity/demographic classification. Automatic confirmation is a policy result, not a human review.

Confirmed regions are bounded in source pixels, BOX-downsampled to roughly one pixel per 24 source pixels, bilinearly expanded and Gaussian blurred with radius `max(12, 0.22 * max(region dimensions))`. Fresh encoding strips metadata/previews/profiles; Office EXIF orientation is baked, PDF uses intrinsic frames, and alpha is flattened onto white. This changes real pixels rather than applying CSS or a reversible overlay. It does not prove anonymity.

Office replacement updates the original media part while retaining surrounding XML/geometry; every reuse receives the replacement, including shared/cropped/hidden portions. PDF OCR pages compose blur before text masking in the edited raster. Native PDF image replacement requires safe source-pixel/occurrence mappings; inline/masked/unsafe geometry is inventoried or blocked. Unsupported media/constructs prevent visual-verification success.

Independent verification retrieves actual serialized output media, recomputes expected source-derived edits, compares encoded/pixel evidence, dimensions, regions, placement/structure and source hashes. Each mapped PDF occurrence is checked, including later pages sharing an xref. Tampering clears protected IDs/counts. A post-blur detector result is not used as proof of privacy.

`V...` audit findings use the existing decision/revision/persistence mechanism; text raw counts remain separate from visual totals. Low-confidence/unsupported visual items remain review work. Crop endpoints are revision-bound, no-store and capped to 512 pixels; there is no full-image source endpoint. Text-offset MISSED_PII rejects FACE/BIOMETRIC additions; manual missed-face region annotation is absent. Zero requested transformations can verify while unconfirmed faces still need review.

### Synthetic fixture provenance

`tests/fixtures/visual/synthetic-face.png` is an **entirely fictional adult portrait generated with OpenAI image generation on 12 October 2026 for local software tests**. No real employee, scraped photograph, identity label or demographic annotation was used. Tests create tiled/rotated derivatives and Office/PDF fixtures locally. The image is not from private Cadence documents and supports functional detection/pixel-blur checks only; it does not establish face accuracy, complete visual privacy or anonymity. This preserves the substantive provenance formerly in the fixture README; the PNG remains untouched.

## 10. Redaction engine

### Exact text replacement

`RedactionEngine.redact(extraction, detections)` validates source identity, unique page numbers, exact source slices, entity labels, confidence and available block references. It does not guess or shift invalid spans. Decisions/collections are checked conservatively; unknown, conflicting or unresolved metadata creates blocking issues. Legacy metadata is read for historical compatibility, not active inference.

Connected confirmed overlaps select a candidate covering the entire group's extent. Equal coverage uses entity priority, confidence and stable input order; each superseded finding remains traceable. Crossing spans without a complete covering candidate remain unresolved instead of exposing their tails. REVIEW overlaps remain blocking even if another confirmed replacement covers some text.

Document-scoped tags are keyed by exact `(entity_type, original_text)`, e.g. `[EMPLOYEE_ID_001]`; exact values of the same type reuse the tag across pages. Assignment follows source order, and right-to-left edits preserve unrelated characters. Original and redacted offsets are recorded separately. Feedback may renumber current tags through deterministic regeneration while original snapshots remain available.

`redaction_complete` concerns supplied findings only. `to_text_payload()` raises for blockers and excludes sensitive audit mappings/paths; even an allowlisted text payload requires coverage review. Never serialize the full result to a downstream system.

### Office artifacts

`OfficeArtifactEngine` checks the original source fingerprint and exact mappings, copies the OOXML package and edits reliably mapped nodes. Multi-run replacements insert a tag into the first intersecting node and remove sensitive portions in subsequent nodes, retaining formatting where practical. Source files are not overwritten; output uses a random `review_<uuid>` path and atomic replacement inside private storage.

PPTX mapping can span generated newlines only when consecutive paragraphs have proven shared part, shape/group chain and text frame or table cell. Structural separators remain in the package; unrelated shapes/cells/slides are not combined. Flattened text preview and artifact paragraph newlines can therefore differ legitimately. Unreliable spans remain unchanged in partial copies with internal mapping issues; broken source identity/maps fail generation.

Verification reopens through python-docx/python-pptx, independently extracts supported text, checks expected replacements/offsets, normalized XML structure and unchanged ZIP members. Residual confirmed-string and email/employee/SSN pattern counts are diagnostics, not labeled missed-PII counts. Unsupported properties/controls/notes/fields/embedded objects are inventoried. Longer tags can change wrapping or overflow; no Word/PowerPoint layout renderer or automatic reflow is implemented.

### PDF artifacts

Native pypdf text is aligned to non-whitespace PyMuPDF glyph sequences, with exact rectangles. Mismatched glyph order, unsupported rotation/geometry or rectangles hitting unrelated glyphs fail mapping; no fuzzy search is used. Redaction annotations are applied destructively before optional tags, and output is saved with compression/cleanup and `garbage=4`, removing underlying content rather than drawing a cover.

OCR pages require matching source bytes, trusted DPI and valid block quadrilaterals with exact text reconstruction. Mapped blocks receive white pixel replacement and retained non-sensitive OCR transcription/tags; the new page embeds only the edited raster, not original source streams/image payload. Typography changes within the reconstructed block and OCR mistakes can persist. Tags are optional when no practical size fits; the historical policy PDF omitted 65 block labels while pixel replacement still verified. Page count/dimensions are retained; scanned-page interactive/vector features may be lost.

Verification reopens and extracts native text, rerenders pages, checks dimensions/page counts/render digests and validates exact expected edited raster pixels/payloads. Removed native text, applicable tags and retained transcription must match the mapped evidence. Overlay-only edits or tampered media fail these checks. Unsupported native images/metadata/annotations and unexamined raster regions are not globally cleared.

`DocumentArtifactEngine` dispatches PDF versus Office without a second pipeline. Every feedback/reload generation starts from the unchanged fingerprinted source, not a prior artifact. Legacy audits lacking the additive trusted PDF fingerprint/DPI or Office mapping metadata may load their audit state but fail artifact generation until explicit valid evidence is supplied; automatic retroactive clearance is absent.

## 11. Audit and traceability

`AuditEngine.create_audit(extraction, detection, redaction)` deep-copies original evidence, verifies candidate accounting and reproduces baseline redaction before accepting it. `AuditResult` contains original/effective snapshots, records, current queue, counters, feedback and revision. Baseline fingerprints bind source identity and complete extraction/detections/redaction, not individual PII values; consistency checks are not digital signatures.

Deterministic document-local `D000001`-style IDs identify automated text findings; `H...` identify validated human additions, and `V...` identify visual findings. Metadata retains original/effective decisions separately, source locations/offsets, confidence, recognizer/method, reasons, block/map evidence, applied tag/redacted offsets, superseding record and corrected status. Duplicate raw occurrences remain separate. Statuses include APPLIED, REJECTED, REVIEW_REQUIRED, SUPERSEDED, UNRESOLVED_OVERLAP and VALIDATION_ISSUE, with visual protection derived from actual verified artifact IDs.

| Feedback action | Contract |
| --- | --- |
| `CONFIRM_PII` | Effective CONFIRM, preserving the original automated assessment. |
| `FALSE_POSITIVE` | Effective REJECT; regeneration restores original characters/region content. |
| `MISSED_PII` | Validate original location/end-exclusive offsets/entity, slice source text and add a human-provenance CONFIRM. It accepts no arbitrary replacement/original text. |

Human confidence `0.0` is an uncalibrated sentinel displayed as “Not scored”, not model probability. Reliable block references are derived only from exact reconstruction. UTC timestamps and previous/new decisions remain in immutable events; supplied timestamps must be timezone-aware. Reviewer identity/reason are caller assertions, not authenticated/signed evidence.

Exact retries are idempotent. Reversing an already corrected finding requires `expected_decision` matching current state; stale expectations raise `FeedbackConflictError`. Repeated same span/type additions are idempotent, and an already automated finding must use its ID. A noncontradictory confirmation with different reviewer/reason may be a distinct event. Queue items group connected crossing-overlap problems rather than duplicating the same conflict. Original raw Presidio count remains separate from effective/human-added counts.

`save_audit` is explicit. The store writes/fsyncs a same-directory temporary file and atomically replaces JSON, preserving the preceding snapshot on failure. Changed saves require the expected saved revision; unchanged writes are idempotent. A different automated baseline cannot overwrite historical reviewer state even with a matching revision: use a separate store/run context.

Reload reconstructs original inputs, validates baseline and schema, replays all feedback and checks decisions, tags, counters, queue, revision and traceability. The pipeline then regenerates artifacts from the retained source. Stored artifact paths and old pipeline timings are not a parallel authoritative state. Path/byte changes and inconsistent snapshots fail closed; moving a checkout can invalidate stored source paths.

Standalone defaults are `.private_audit`, `.private_artifacts` and `.private_feedback`; the web factory supplies its own subdirectories. There is no multi-process transaction, signed ledger, automatic retention worker or snapshot migration service.

Offline feedback export includes **actual feedback events only**, metadata by default: entity, original/reviewer decisions, action, recognizer/method/confidence, source category/format and explicit policy version. No candidate text, path, document/reviewer ID or free-text reason is exported by default. Context is explicit opt-in, bounded to 400 code points, inside restricted plaintext output; existing exports are not overwritten. Old unknown policy versions remain `unknown-legacy` unless the operator has evidence. Demonstration overrides are not gold labels and never automatically update detection rules.

## 12. Backend and API

All routes are defined in [server.py](app/web/server.py), with schemas in [models.py](app/web/models.py). Below, `{id}` is an opaque current session ID, not a filesystem path.

| Method | Route | Responsibility |
| --- | --- | --- |
| GET | `/` | Committed React index. `/assets` serves only the static bundle. |
| GET | `/api/config` | Upload cap, supported formats and trusted-local flag; practical liveness check. |
| GET / POST | `/api/documents` | List sessions / validate multipart `file`, privately copy and process it. |
| GET | `/api/documents/{id}` | Sanitized processing, revision, stage and artifact state. |
| GET | `/api/documents/{id}/review` | Authorized display name and page/slide/logical-segment convention. |
| GET | `/api/documents/{id}/findings` | Filtered/paginated metadata; decision/entity/location, metadata search. |
| GET | `/api/documents/{id}/findings/{finding_id}` | Bounded text evidence or visual-crop URLs. |
| GET | `/api/documents/{id}/findings/{finding_id}/comparison` | Actual current mapped replacement/context, or null when unavailable. |
| GET | `/api/documents/{id}/findings/{finding_id}/visual-crop` | Bounded before/verified-after region; optional strict `revision` guard. |
| GET | `/api/documents/{id}/source` | Original source window and highlights. |
| POST | `/api/documents/{id}/feedback` | Confirm/false-positive action with expected revision. |
| POST | `/api/documents/{id}/missed-pii` | Validated manual original-text span with expected revision. |
| GET | `/api/documents/{id}/artifact` | Current private artifact; `inline=true` permits PDF preview. |
| GET | `/api/documents/{id}/audit` | Authorized feedback events plus aggregate summary. |
| POST | `/api/documents/{id}/save` | Explicit persisted revision check/save. |
| GET | `/api/saved-audits` | Opaque references/labels rebuilt from local snapshots. |
| POST | `/api/load-audit` | Restore using a previously listed opaque audit reference. |

There is no `/health`, `/docs`, ReDoc or OpenAPI endpoint. General summaries exclude source text, filenames/paths, hashes, tags, reviewer IDs/reasons; purpose-specific review endpoints intentionally expose bounded sensitive context to the same trusted local boundary.

### Bounds and mutation contracts

The file cap is **30 MiB**; multipart overhead is bounded before parsing and copying. Other mutating bodies are bounded to 16,384 bytes. The multipart spool threshold exceeds the maximum request so raw parsing data stays in memory until private storage. Source views default to **1,600**, maximum **2,000** characters; finding details use up to **120 before / 240 candidate / 120 after**. Comparisons bound context to 480 and replacement to 240. Findings pagination defaults to 50 and caps at 100. Metadata search is bounded to 80 characters.

Feedback requires strict nonnegative `expected_revision`, a finding ID, action, reviewer ID (default `local-reviewer`, max 80) and optional reason (max 1,000). The API derives the current expected effective decision. MISSED_PII supplies positive location, integer start/end, uppercase entity and 1–1,000 source characters. Schemas forbid extra fields; FACE/BIOMETRIC cannot be added with text offsets. Save accepts `expected_revision`; load accepts the opaque 32-hex audit reference.

Missing trusted mutation header or foreign Host/Origin returns 403; stale revisions/source conflicts commonly return 409; invalid selections/fields 422; oversize 413; unsupported web formats 415. Corrupt/mismatched inputs are rejected before pipeline processing. Invalid or absent sessions/findings produce bounded errors.

Artifact URLs include a frontend `revision` cache-refresh hint, but the artifact handler serves the **current** file and does not enforce historical selection with that query. Visual crop revisions are enforced separately and stale crops return 409. Neither URL form is an immutable historical artifact API.

## 13. Frontend and user experience

`src/main.tsx` mounts React with user reduced-motion preferences. `App.tsx` owns document context/navigation; `hooks/useWorkspace.ts` loads/refetches state and guards obsolete document requests. `lib/api.ts` provides safe errors, serialized requests and opaque URLs; review/Unicode helpers convert DOM selection to Python code-point offsets. No Redux, remote query platform, GraphQL, websocket progress system or client-side classifier is introduced.

The product has Overview, Documents, Review Queue, Review & Governance Trail, Artifact Verification and Detector Quality views. Actual metadata supplies counts, Bklit disposition branches/rings and location maps; rings show the five largest groups while the legend retains all types. Human additions are separate from raw counts and can exceed them. Map density is descriptive, not a privacy score. DOCX locations are logical segments, PPTX locations slides and PDF locations pages.

The UI loads all findings **metadata** via 100-record pages for maps/charts, fetching bounded source only on selection. Review shortcuts select the first relevant finding; actions are beside evidence, and successful feedback selects the next document-wide REVIEW item. Map filters retain intended location until feedback moves to the wider queue. Pending/duplicate-submit guards, stale-revision refresh and obsolete-request checks prevent mismatched state. Failed generation disables stale downloads and reports the actual returned failure.

Original automation, actual human decision and current effective transformation remain distinguishable. Actual replacement/comparison data is required before claiming a redaction was applied. The governance timeline timestamps real feedback events; current artifact state is not invented as a past timestamped regeneration event. Explicit save/reload uses existing audit APIs.

Source selection is bounded to its DOM container and counts Unicode code points across text/highlight nodes, preserving Python offsets for emoji/non-BMP input. The dialog freezes the range and sends offsets/entity/reviewer/reason/revision, not selected replacement text. PDF uses a real current-artifact iframe; Office uses extracted evidence and actual downloads, without simulated Word/PowerPoint rendering.

The business-artifact hero, cyan output card, amber review/coverage states and red blocked AI boundary explain purpose. “No AI connection is enabled” and explicit blocking remain visible, including mobile. Detected/effective numbers and timings come from the backend. Synchronous requests show processing/regenerating indicators without fabricated live stage progress or upload/human elapsed time.

The React migration fixed connection bursts by serializing JSON requests instead of changing the four-connection launcher. Radix's injected scroll-lock stylesheet conflicted with strict CSP; external-CSS locking retained focus/dismissal without `unsafe-inline`. A restored `/audit` action serializer was made compatible with enum and string forms. Phase 10B.1 fixed shortcuts opening an empty unselected workspace and added prominent warned downloads; 10B.2 clarified the privacy story without granting clearance.

Selected Bklit MIT components are pinned to `f7b3bd2999a5c993863f9cc5523c9cf60b9f3bc2`, shadcn/ui to `993eb2c3151997ce08a49635c10fd8c8ef3713c7`. The local funnel denominator scales against the largest displayed branch so human-added counts do not overflow. Proprietary Bklit Studio is absent. The detailed [third-party notices](frontend/THIRD_PARTY_NOTICES.md) and both component `LICENSE.txt` files remain required and unchanged.

Historical Chrome checks covered 1540/1280/820/390-pixel layouts, keyboard/semantic controls, reduced motion and no page overflow. Mobile stacks evidence below lists, and contained tables may scroll. Formal accessibility certification and independent usability studies were not performed.

## 14. Dependency management

### Supported installation inputs

| Manifest | Role |
| --- | --- |
| `requirements.txt` | 19 selected application distributions on Windows; platform-specific libmagic alternative elsewhere. |
| `constraints-windows-py313.txt` | 78 exact resolved runtime/test versions; constraints alone do not install optional tests. |
| `requirements-dev.txt` | Runtime plus pytest 9.1.1, httpx 0.28.1 and ReportLab 5.0.1. |
| `requirements-models.txt` | Official English spaCy model 3.8.0 URL and SHA-256. |
| `.python-version` | Selected Python minor 3.13; explicit `py -3.13` commands prevent accidentally using host-default 3.14. |
| `frontend/package.json` / `package-lock.json` | Exact frontend resolution and dev/test/lint/build scripts. |
| `docs/dependencies/requirements-original.txt` | Preserved historical mixed snapshot; intentionally not installable as the supported environment. |

The historical snapshot pinned `paddlex==3.7.2` and `PyYAML==6.0.3`, but PaddleX required `PyYAML==6.0.2`; pip reproduced `ResolutionImpossible`. PaddleX was unused by the application, so stabilization isolated real runtime/test dependencies instead of installing large exploratory frameworks or bypassing the resolver. Docling/Paddle scratch imports are not proof they belong in normal setup. Presidio anonymizer is not the custom redactor's implementation.

Windows installs only `python-magic-bin==0.4.14`, whose native binaries implement the required MIME API; `python-magic==0.4.27` is marked for other platforms. RapidOCR requires standard `opencv-python==5.0.0.93`, which also provides `FaceDetectorYN`; contrib is unnecessary for the validated application. Actual MIME/OCR/face/artifact tests established compatibility after choosing providers, rather than assuming the overlap was safe. NumPy 2.3.5, ONNX Runtime 1.30.0, spaCy 3.8.16 and the other direct runtime pins were preserved from the original manifest. Resolved transitive versions are explicit in the constraints file.

The new Python 3.13.5 x64 venv resolved/installed normally, and a constrained `--ignore-installed` dry run resolved 78 distributions. Including the English model yields 79 non-pip distributions. `pip check` passed. No `--no-deps`, force-install or global Python modification was used. Python 3.12 fallback was unnecessary; other Python/OS combinations remain unverified.

`en_core_web_lg==3.8.0` is compatible with spaCy 3.8.16. Its official release wheel is **400,658,291 bytes**, SHA-256 **`293e9547a655b25499198ab15a525b05b9407a75f10255e405e8c3854329ab63`**. During stabilization a slow download was resumed from the same official asset using checked HTTP byte ranges, length, ZIP CRC and version checks, then installed locally with the normal resolver. The current manifest pins the canonical URL/hash; no smaller/replacement model was substituted. A GitHub release-API metadata query hit HTTP 403/rate limiting, but the release asset was accessible without credentials. A historical PyMuPDF download had similarly required verified range acquisition; those were transport workarounds, not application logic.

Verified Windows prerequisites are Python 3.13.5, Poppler 26.09.0 and Windows PowerShell for ACLs; Node 22.16.0/npm 11.12.0 support optional frontend work. No compiler was needed for the validated wheels. No model/requirements versions change in this consolidation. Exact CMD setup, imports and startup are centralized in README sections 5–9.

Browser-only variables are `CADENCE_UI_URL`, `CADENCE_UI_FIXTURE`, `CADENCE_TEST_CHROME`, and selected harness flags `CADENCE_UI_SYNTHETIC`, `CADENCE_UI_COMPLETE`, `CADENCE_UI_REPRO`. Default test URLs vary: wrapper 8765, phase10b variants 8766, phase11 8767; explicitly set 8765 for the normal launcher. These do not configure backend ports. PATH locates native binaries; TEMP/TMP affect temporary files. Historical Paddle environment assignments are experiment-only. No app `.env`, database URL or API secret is required.

## 15. Testing and verification

### Historical regression progression

These are reported measurements from the retired documents, consolidated without representing them as new executions. Counts differ as tests/scopes evolve, especially when generative tests are removed; they must not be summed as independent coverage.

| Historical scope | Python result | Other meaningful evidence |
| --- | --- | --- |
| Engine 4 initial | 44 redaction unit passes; 51 combined passes; separate upstream detector regression failed | Full-candidate integration exposed 3 raw/2 omitted/0 applied. |
| Repaired Qwen-era Engine 3 | 141 passed, 16.33 s | 70 detection, 20 resolver protocol/parsing, 44 redaction, two PDF integration and five extraction/file cases; actual Qwen smoke was separate. |
| Deterministic transition | 117 passed, 14.01 s | 66 detection/44 redaction/two integration/five existing cases; fresh process makes generative stack unavailable and still runs default Presidio. |
| Audit | 164 passed, 20.46 s | 47 new audit cases; private atomic save/replay and real synthetic PDF review. |
| Pipeline | 192 passed, 22.98 s | 28 orchestration cases, failure boundaries, native PDF and no upstream reruns. |
| Office | 217 passed, 61.23 s | 27 focused Office cases; virtual placeholders/mapped structure and real Presidio. |
| PDF/PPTX artifacts | 233 passed, 40.19 s | 11 PDF and five boundary cases; actual removal/raster/tamper/source checks. |
| Local web | 252 passed, 70.27 s, one warning | 19 API cases; four Node tests; actual browser PPTX download 68,314 bytes/9 slides and saved revision 1. |
| Policy 1.1 | 279 passed, 74.59 s, one warning | 27 evaluation/hardening/curation cases; exact labels/metrics retained in JSON. |
| React migration | 284 passed, 77.93 s, one warning | 19 React/four Node; lint/types/build; historical npm audit reported zero known vulnerabilities after tool upgrades. No current vulnerability scan is inferred. |
| Review UX fix | 284 passed, 77.42 s, one warning | 30 React/four Node; PDF/DOCX complete queue at revision 2. |
| Product-story UX | 284 passed, 85.69 s, one warning | 36 React/four Node; responsive/browser interactions and synthetic-only metric disclaimers. |
| Visual privacy | 302 passed, 87.78 s, one warning | 40 React/four Node; actual serialized blur/per-occurrence proof and browser face/text/PDF preview checks. |
| Windows stabilization | 302 passed, 92.68 s, one warning | 40 React (12.19 s), four Node, lint/types; six real-engine/HTTP fixtures and two real Chrome scenarios. |

Historical tests included injected analyzers and controlled blocks as well as real Presidio/YuNet and actual file libraries. The distinction matters: e.g. a scanned visual-composition fixture supplied known 72-DPI OCR blocks rather than re-running OCR; later stabilization explicitly exercised real Poppler/RapidOCR at the normal route.

### Seeded detector evaluation and exact evidence retained

The unchanged [dataset](tests/data/detector_evaluation.json) contains **420 synthetic cases / 320 positive spans**, seed **10010**, version `synthetic-1`; category-stratified development/held-out splits are **299/121 cases**, with **224/96 gold spans**. Labels were generated independently of detector output. Invented contacts use example.com/fictional telephone ranges; no real-document PII was copied.

Exact `(case, start, end, entity_type)` keys are deduplicated for scoring. TP is confirmed intersect gold, FP confirmed minus gold, FN gold minus confirmed. Boundary/type mistakes count as FP and FN; zero denominators return 0. REVIEW is never an automated TP. Potential recall uses confirmed union REVIEW and assumes correct future human adjudication. Candidate decision rates use raw candidates, retaining duplicate evidence.

| Split / metric | Policy 1.0 | Policy 1.1 |
| --- | ---: | ---: |
| All: TP / FP / FN | 267 / 65 / 53 | 267 / 5 / 53 |
| All: precision / recall / F1 | 80.42% / 83.44% / 81.90% | 98.16% / 83.44% / 90.20% |
| All: CONFIRM / REJECT / REVIEW (672 raw) | 332 / 204 / 136 | 272 / 344 / 56 |
| All: REVIEW rate / potential recall | 20.24% / 89.69% | 8.33% / 89.69% |
| All: reviewed gold / non-gold spans | 20 / 116 | 20 / 36 |
| Development: TP / FP / FN | 189 / 47 / 35 | 189 / 4 / 35 |
| Development: precision / recall / F1 | 80.08% / 84.38% / 82.17% | 97.93% / 84.38% / 90.65% |
| Development: REVIEW rate / potential recall | 20.99% / 90.62% | 8.64% / 90.62% |
| Held-out: TP / FP / FN | 78 / 18 / 18 | 78 / 1 / 18 |
| Held-out: precision / recall / F1 | 81.25% / 81.25% / 81.25% | 98.73% / 81.25% / 89.14% |
| Held-out: CONFIRM / REJECT / REVIEW | 96 / 56 / 34 | 79 / 93 / 14 |
| Held-out: REVIEW rate / potential recall | 18.28% / 87.50% | 7.53% / 87.50% |

The frozen final all-case output includes final development metrics. [detector_eval_development.json](detector_eval_development.json) is an intermediate development run, not evidence that the held-out set was independently free of scenario/template overlap. [Baseline JSON](detector_eval_baseline.json) and [hardened JSON](detector_eval_hardened.json) retain all metrics formerly embedded in their Markdown counterparts, plus per-case candidates, recognizers and fingerprints. Every embedded metric was compared equal before removing the redundant Markdown; the JSON files are unchanged.

| Entity | Baseline TP/FP/FN | Hardened TP/FP/FN | Interpretation |
| --- | --- | --- | --- |
| EMAIL_ADDRESS | 60/20/0 | 60/0/0 | Malformed-dot false positives removed in this synthetic set. |
| EMPLOYEE_ID | 40/0/0 | 40/0/0 | Exact documented format retained. |
| PERSON | 127/0/33 | 127/0/33 | Automated recall 79.38% overall, 75% held-out; discovery misses remain. |
| PHONE_NUMBER | 20/40/0 | 20/0/0 | Explicit business-number false positives reduced. |
| US_BANK_NUMBER | 0/0/20 | 0/0/20 | All 20 gold values stayed in REVIEW, not automated TPs. |
| US_PASSPORT | 0/5/0 | 0/5/0 | Five numeric-business false positives remain; no positive passport ground truth. |
| US_DRIVER_LICENSE | 0/0/0 | 0/0/0 | No positive ground truth; zeros are not perfect performance. |
| US_SSN | 20/0/0 | 20/0/0 | Synthetic examples only. |

There are no positive passport/license/card/IBAN gold labels. Baseline FP categories were malformed email 20, numeric business values 25, project codes 20. FN categories included accounts 20, OCR names 20, tables 7 and smaller person/header/adjacent-email/repeated-context cases. Remaining held-out FNs were 12 PERSON and six accounts. Historical evaluation runtime was 3.704621 s baseline and 4.148903 s hardened, with one analyzer per evaluation. None of these figures is real Cadence-document accuracy.

Rules were chosen on development output; a final employee-boundary refinement preceded the frozen all-case run, with no reported tuning after held-out evaluation. Scenario templates/names recur across splits, and independent real labeled ground truth is absent. Two-capitalized-word project names can still confirm as PERSON; concatenated OCR names often produce no PERSON candidate for validation to recover.

### Historical real-document findings

The initial policy PDF was **14,094,994 bytes**, 35 image-only pages, source SHA-256 `3f4b5db17d2e2694630fe4244754a270d4c0dde68ef009ee0495e43a2bf606f3`. Reported extraction produced **107,571 page characters**, 107,639 joined characters and **3,396 OCR blocks**, with no missing/empty pages. The 68-character difference is 34 two-newline joins. OCR confidence mean/minimum was 0.8684/0.5011; 401 blocks were below 0.8. These are OCR signals, not correctness labels.

The original diagnostic resumed after a checkmark console failure using page-1 checkpoints: every page underwent OCR once. Resumed extraction took 211.97 s. Qwen adjudicated 172 ambiguous findings (87 PERSON, 28 bank, 57 license): 97 CONFIRM / 75 REJECT / zero REVIEW, with one model load and no reported inference failure. Total detection was 769.52 s, including 758.76 s actual generation; successful runner 993.71 s and cumulative interrupted+successful primary attempts 1,037.97 s. Timings nest and must not be added. This retired architecture's zero REVIEW is not evidence of certainty.

| Historical detector state on cached policy | Raw | CONFIRM / REJECT / REVIEW | Applied / superseded / unresolved confirmed |
| --- | ---: | --- | --- |
| Qwen-era initial validation | 1,418 | 675 / 743 / 0 | 637 / 32 / 6 (three crossing groups) |
| Deterministic policy 1.0 | 1,418 | 578 / 668 / 172 | 577 / 1 / 0 |
| Policy 1.1 comparison | 1,418 | 577 / 669 / 172 | 576 applied; no new raw candidates |

The initial 637 replacement-position checks passed; 621 applied spans had reliable OCR block references and 16 did not. Crossing PERSON/phone groups occurred on pages 12/31/32; independent pattern diagnostics found ten remaining email-like, eight telephone-like and one SSN-shaped match. Two labeled name-field anchors generated 43 diagnostic occurrences, eight not fully covered across pages 1/7/12/23/31/32/35. Sixty surviving strings matched values confirmed as PERSON elsewhere, but duplicates/business terms/substring overlap prevent calling these 60 verified identity leaks. Forty government/bank confirmations lacked supporting context; three overlong PERSON and 38 OCR-contamination flags required review. None is a labeled precision/recall estimate.

After model removal, all 172 ambiguous candidates remained REVIEW; deterministic initialization/detection/redaction measured 1.06/4.18/0.015 s on cached OCR. This was not a controlled full-OCR benchmark against Qwen. All 1,418 audit records later survived save/replay: 577 APPLIED + 668 REJECTED + 172 REVIEW_REQUIRED + one SUPERSEDED. The cached audit diagnostic took 12.041 s including startup/validation; a pipeline-only load measured about 1.049 s. Engine 5 did not discover or certify residual PII.

| Phase 7/8 Office/artifact measurement | Training DOCX | Organizational PPTX |
| --- | ---: | ---: |
| Locations / mapped paragraphs | 2,610 logical segments / 2,610 | 9 slides / 424 |
| Nonempty locations / extracted characters | 2,345 / 92,898 | 9 / 8,366 |
| Raw / CONFIRM / REJECT / REVIEW | 424 / 29 / 85 / 310 | 59 / 25 / 19 / 15 |
| Phase 7 artifact replacements / mapping failures | 29 / 0 | 23 / 2 |
| Phase 8 replacements / mapping failures | 29 / 0 | 25 / 0 |
| Phase 8 supported mapping verification / reopen | Pass / Yes | Pass / Yes |
| Phase 8 bytes / pipeline seconds | 4,813,711 / 14.694389 | 68,330 / 1.047623 |

The Phase 8 policy PDF applied/verified 577 mappings, produced **24,333,760 bytes** and measured **12.087934 s** with cached OCR/detection; generation/verification were 7.563962/3.990948 s. All source originals remained unchanged and all three outputs reopened. Supported checks passing did not clear the queue/unsupported content or establish global `artifact_verified`.

The Office unsupported inventory was preserved: DOCX alternative-text objects 58, annotation/note parts 2, controls 8, property parts 3, dynamic fields 44, external relationships 2, media without text OCR 41, text boxes 4, unmapped text nodes 80; PPTX alternative-text objects 9, property parts 3, fields 11, media 1, unmapped nodes 16. Categories overlap and are inventory units, not PII counts.

Phase 10A left DOCX/PPTX distributions unchanged. PDF changed one PERSON REVIEW to REJECT and one partial email CONFIRM to REVIEW, leaving REVIEW total 172. No new confirmations occurred. Exact confirmed-email-string residual warnings in redacted extracted text rose **4 to 7**, while employee warnings stayed 13 and PERSON 72; DOCX/PPTX PERSON warnings stayed 2/1. These instance/string warnings differ from artifact selectable-text counters and are not distinct labeled leaks. The remaining 287 DOCX license-like numeric candidates were not rejected simply to reduce the review burden.

Retained [baseline document aggregates](detector_documents_baseline.json), [hardened aggregates](detector_documents_hardened.json) and [risk checks](detector_documents_risk_checks.json) preserve detailed counts. Artifact residual confirmed-string counts were PDF/DOCX/PPTX **8/33/1**, with separate sensitive-pattern counts **0/0/34**. No independent OCR sweep of all output raster content established complete residual absence.

| Phase 11 real visual inventory | PDF | DOCX | PPTX |
| --- | ---: | ---: | ---: |
| Inspected units | 35 page rasters | 35 decoded assets | 1 decoded asset |
| Faces / CONFIRM / REVIEW / protected | 2 / 0 / 2 / 0 | 0 / 0 / 0 / 0 | 0 / 0 / 0 / 0 |
| Visual mapping failures | 0 | 0 | 0 |
| Unsupported visual inventory | 0 | 12 | 3 |
| Face inspection seconds | 24.072 | 11.443 | 0.058 |
| Pipeline seconds | 39.094 | 27.174 | 2.683 |

DOCX had six unsupported image formats plus six uninspected constructs; PPTX had three constructs. Two PDF candidates were deliberately left unadjudicated; zero protected faces is not zero face risk. Supported text replacements were 576/29/25, but global artifact verification remained false. Total diagnostic elapsed was 69.384 s, reusing trusted PDF OCR/text detection. [Visual JSON](phase11_visual_diagnostic.json) and [inventory recheck](phase11_inventory_check.json) remain: the latter recorded matching inventories with zero OCR/inference runs, not new detection evidence.

Private originals/caches were used only in the historical phases. Their availability is not assumed in a fresh clone, and this consolidation does not open confidential documents, rerun expensive real OCR, apply real reviewer overrides or alter saved application data.

### Historical Windows runtime and browser verification

The 15 October runtime audit first found missing imports and a resolver conflict; its NOT READY verdict is superseded by the stabilization evidence, not silently deleted from history. Stabilization verified imports, native MIME, Poppler/RapidOCR, spaCy/Presidio, YuNet and actual private-storage ACL creation. Six fictional fixtures ran through real default engines and live HTTP (not TestClient mocks): text PDF, scanned PDF, hybrid PDF, DOCX, PPTX and face PPTX.

Their replacement counts were **3/3/6/3/3/4**, with one protected face in the last. Scanned OCR produced three blocks/68 characters. Every fresh HTTP upload completed all eight displayed stages with no summary failure and a reopened, supported-content-verified artifact. Findings/source evidence, reject/reconfirm to revision 2, stale-revision 409, missing-header 403, downloads and explicit save passed. After restarting the actual launcher, saved listing/reload restored revision 2 and two events. Altering only that newly created synthetic source caused 409; restoring its bytes allowed reload and regenerated artifact reopening.

The first added verifier's reload assertion incorrectly expected original extraction/inference timings to be Completed. It was corrected to require persistence load, restored evidence and artifact regeneration, matching the existing source contract; application behavior and existing tests were not changed. Optional Vite proxied config and a real multipart upload successfully with all stages completed.

Real Chrome against the committed FastAPI bundle verified:

- Fictional PDF via `phase10b1_browser.mjs`: initial raw/confirmed/REVIEW/rejected **6/3/2/1**, then **6/4/1/1**; automatic selection, actions/dialog, pending controls and next finding; queue complete at revision **2**; **1,222-byte** downloaded PDF reopened; zero runtime/security errors.
- Fictional face PPTX via `phase11_browser.mjs`: one detected/protected face, two placements, both crops loaded; actual downloaded PPTX reopened, image dimensions preserved; blur gradient ratio **0.0414792062**; zero runtime/security errors.

Earlier React phases also reported real DOCX all-three-action revision-3 save/reload, synthetic PDF inline preview, and a 59-candidate supplied-PPTX demonstration with confirmed 25→26, REVIEW 15→14 and a 68,314-byte download. The real-document overrides were explicitly demonstrations, not gold truth. Historical original build sizes changed with UI phases and do not establish performance/accuracy. Headless acceptance and private screenshot inspections are not independent human usability certification.

### Current consolidation verification

Executed on **15 October 2026**, after the 21 redundant Markdown files were removed. These are fresh consolidation-session results, separate from the historical runtime/browser evidence above. No packages, application code, tests or committed frontend bundle were changed.

| Check / actual command | Current result |
| --- | --- |
| `set PYTHONDONTWRITEBYTECODE=1` then `.venv\Scripts\python.exe -B -m pytest tests -q -p no:cacheprovider` | **302 passed, 0 failed, 0 skipped, 0 errors; 1 warning; 88.92 seconds.** Exit 0. |
| `npm test` in `frontend` | **40 passed**, one test file; 33.55 seconds total, 11.98 seconds test execution. Exit 0. |
| `npm run lint` in `frontend` | Passed; `eslint src --max-warnings 0`, exit 0. |
| `npx tsc --noEmit` in `frontend` | Passed, exit 0. |
| `node --test tests\frontend\core.test.mjs` | **4 passed**, no failures/skips/cancellations, 82.3193 milliseconds total. Exit 0. |
| `.venv\Scripts\python.exe -B -m pip check` | `No broken requirements found.` Exit 0; no installation performed. |
| Import/startup and real HTTP smoke | Imported `run` and `create_app`; started the existing app factory with the launcher's loopback/security/concurrency settings on free port **8768**, in a fresh private work directory. Root UI **HTTP 200**, 456-byte HTML; `/api/config` **HTTP 200** with exactly the README configuration. Isolated server stopped successfully. Existing app processes, sessions and data were preserved. |
| Documentation inventory and links | **24 original Markdown files reviewed; 21 removed; two authorities plus two unchanged attribution exceptions remain.** All **51 relative links**, including fragments, resolve in the authorities. **14 original links to retired documents** were replaced or removed through consolidation; zero such dangling links remain. |
| Preservation and hygiene | SHA-256 comparison of **259 tracked non-Markdown files** against the pre-edit baseline found **zero changes**. Both retained attribution files match their original hashes. Generated evaluation Markdown payloads match retained JSON. `git diff --check` passed. |

The one Python warning is the existing Starlette TestClient/httpx deprecation described above; it is nonblocking and no dependency version was changed. Current smoke testing checks startup/configuration, not a new full browser or document-accuracy certification. Earlier real-engine/browser/restart checks remain explicitly historical.

Local, ignored evidence is retained under `.venv/documentation-consolidation`: original/classified inventories, non-Markdown hash baseline, `pytest.log`, `smoke.log`, `smoke.json` and `verification.json`. The smoke created only its own fresh `.private_review_app/documentation-smoke-9fa8fb6529ca4f68a72086a78be1ffda` storage; it uploaded no documents and wrote no user review data. Frontend test/lint/type results and Node/pip output were also checked directly from their successful command executions. No frontend production build was run.

### Reproduction and optional tooling

README section 9 gives normal regression and real-engine/HTTP/restart commands. For optional current browser face acceptance, using a running normal backend and the new synthetic-check fixture directory created there:

```cmd
set "CADENCE_UI_URL=http://127.0.0.1:8765"
set "CADENCE_UI_FIXTURE=%CD%\%CADENCE_CHECK_DIR%\fixtures\faces\synthetic-face.pptx"
set "CADENCE_UI_SYNTHETIC=1"
python -c "from app.engines.audit.storage import AuditStore; AuditStore('.private_phase11')._prepare()"
node tests\frontend\phase11_browser.mjs
```

`phase10b1_browser.mjs` accepts a compatible synthetic PDF/DOCX and optional `CADENCE_UI_COMPLETE=1`; the broader `phase10b_browser.mjs` exercises CONFIRM/MISSED_PII/FALSE_POSITIVE and revision-3 save/reload in synthetic mode. `CADENCE_TEST_CHROME` selects a different installed Chrome path. The default Chrome path is `C:/Program Files/Google/Chrome/Application/chrome.exe`. Evidence remains private.

Evaluator output should use a fresh generated/private path, not overwrite the retained baseline:

```cmd
python -c "from app.engines.audit.storage import AuditStore; AuditStore('.private_detector_eval')._prepare()"
python -B tools\evaluate_detector.py --output ".private_detector_eval\evaluation-%RANDOM%-%RANDOM%.json"
```

It also generates a Markdown rendering beside the JSON, under the excluded private/generated evidence area. Running current code with an old baseline filename does not recreate historical policy 1.0. Existing output is refused. The legacy `tools/write_phase10a_report.py` can emit a historical root report from saved aggregates; it is unchanged tooling, not a runtime documentation dependency or a required setup step. Its output filename is intentionally not treated as a live link; running it would recreate redundant documentation and is unnecessary for maintenance of these two authorities.

Other tools remain available unchanged: dataset building/evaluation; detector document comparison/inspection; metadata-only quality export (which intentionally writes frontend source JSON); private feedback export; historical Cadence/Office/Phase 8/Phase 11 diagnostics and inventory checks. They require their documented private caches/input evidence and may write outputs; do not run them as harmless installation probes. The source tree's optional report generators are not read by the application, tests or build for startup.

## 16. Major bugs and resolutions

| Defect / diagnostic symptom | Root cause, resolution and boundary |
| --- | --- |
| Earlier candidates/pages disappeared | Disposition/appending was outside page and candidate loops. Move it inside; retain every raw disposition and test multi-page/duplicate/zero-candidate cases. Historical small integration changed from two omitted of three to zero omitted. |
| `UnboundLocalError` on no candidate | Candidate-local variables were used after an empty loop; separate blank/nonempty-zero results and explicit analyzer failures. |
| REVIEW hidden as rejection / unsafe resolver words | Historical resolver substring parsing and contradictory prompts could imply false certainty. Strict parsing/lazy inference helped, then deterministic REVIEW replaced generative adjudication entirely. |
| Context-free Qwen confirmations / CPU expense | Actual model overconfirmed uncertainty; 172 real calls consumed most detection time. Retired model decision path, leaving uncertainty to review. |
| Windows `SeSecurityPrivilege` persistence errors | ACL setup requested broader security sections. DACL-access-only handling preserved intended permissions and passed real atomic persistence/replay tests. |
| Phantom PPTX paragraph mapping | python-pptx could create empty paragraphs absent from source ZIP; validate original XML and skip only empty virtual nodes, otherwise fail. |
| Two real PPTX confirmed spans unmappable | Generated trailing/adjacent paragraph newlines had no editable nodes. Prove shared frame/cell, edit actual characters and retain separators; never merge unrelated shapes. |
| PDF review overlay insufficient / OCR small-block edge cases | Destructive native edits plus fresh edited raster payloads and independent checks; mapped OCR separators and tiny/overlapping text checks corrected generically. Optional labels do not gate actual pixel removal. |
| Wrong repeated-number context | Search could borrow the first occurrence's field; pass exact relative context offset and require adjacent explicit business evidence. |
| Concurrent React request overload | Initial mount bursts exceeded four connections; serialize JSON requests without relaxing the launcher or security boundary. |
| Radix CSP error | Injected scroll-lock stylesheet violated strict style policy; external-CSS class retained modal interaction without `unsafe-inline`. |
| Reloaded audit timeline serialization | Restored feedback actions were strings, while `/audit` assumed enum `.value`; compatible serialization and replay/timeline regression fixed it. |
| Review shortcut opened empty evidence | Navigation changed filters but did not select detail. Unified shortcuts, first/next selection and direct warned downloads repaired the interaction; old browser test had manually selected a row. |
| Dependency `ResolutionImpossible` / missing `magic` | Mixed historical snapshot and no project environment. Isolate actual runtime/test/model manifests, constrain compatible versions, select/verify one native namespace provider. |
| Redirected OCR `UnicodeEncodeError` | U+2713 completion glyph failed under Windows cp1252 after OCR had succeeded. One ASCII log-message change fixed supported redirected execution without changing OCR data/parameters. |
| Apparent reload stage failure in new smoke tool | Incorrect verifier expectation, not application failure: reload intentionally omits original stage timings. Correct only the new verifier to the actual saved-evidence contract. |
| Phase 11 concurrent validation memory failures | Full regression plus multiple Chromium/model processes exhausted host memory, causing eight failures and one setup error. Sequential full suite and later browser runs passed; no safety checks were weakened. |

Historical transient test-selector/wrapper failures (e.g. stale fake-resolver references, exact-offset wrapper signature, duplicate accessible labels, unsupported role options and unused imports) were corrected without inventing product fixes or removing assertions. They are distinct from production defects. A nested CMD/npx temp-path build failure was an audit-command quoting error; a relative isolated build succeeded and matched committed assets after CRLF normalization.

## 17. Security architecture

### Implemented controls

- Loopback-only launcher, allowed Host names, same-origin checks and a mandatory local mutation header; CORS/proxy-header trust are not enabled. These reduce ordinary browser cross-origin/DNS-rebinding exposure but are not authentication.
- Request/file limits, content/extension agreement, filename validation and unrelated UUIDs for uploads versus sessions. No arbitrary-path upload/download API exists.
- Private-root/symlink/junction/UNC checks, dedicated directories, current-user/SYSTEM Windows ACLs and local/root gitignore rules. POSIX 0700 directories/0600 snapshots are implemented but not independently exercised on the verified Windows host.
- Same-directory fsync/atomic snapshot replacement, baseline consistency and revision conflict checks; retained source-byte fingerprints before regeneration.
- Bounded evidence/crops, no-store, nosniff, no-referrer, frame restrictions and same-origin CSP. Source/reason rendering uses text/React nodes, not HTML injection.
- Sanitized public failures/aggregate summaries; full internal artifacts and traces do not become downstream payloads. Launcher access logs are disabled. Protected visual status comes from serialized transformation proof, not model confidence alone.

### Remaining trust and operational risks

Storage is restricted **plaintext**, not encrypted or immutable/signed. The filesystem owner can alter/delete files or recompute fingerprints. Atomic replacement is not a cross-process database transaction; multiple Uvicorn workers are unsuitable for shared in-memory sessions. Reviewer IDs, reasons and timestamps are assertions, with no login, role model, external identity attestation or independent audit signature.

Other local processes can call the API and know the non-secret mutation header. Do not infer enterprise authorization from Host/Origin checks or opaque identifiers, and do not expose the application on a network merely by changing its bind address. The UI's blocked AI story describes the absent/disabled downstream capability, not an independently deployed organization-wide DLP gateway.

OCR page images use normal account temp storage unless the documented optional private-temp step is selected. Upload memory/spool limits do not universally bound decompressed ZIP/XML, PDF rendering, image complexity, CPU time, total retained sessions or disk growth. Media-specific size guards exist, but no comprehensive adversarial resource-exhaustion validation was performed. There is no automatic retention/cancellation/job worker, encrypted backup policy or production security configuration.

Gitignore cannot prevent deliberate force-adds. Keep private caches, profiles, crops, original inputs and partial review artifacts out of public/static hosting. Source/audit consistency is useful traceability, not independent proof every PII value was found or transformed.

## 18. Known limitations

| Area | Current limit |
| --- | --- |
| Text/face discovery | Probabilistic Presidio/YuNet and scoped deterministic rules; no complete labeled real-document recall, face precision or anonymity guarantee. |
| Policy scope | Addresses/dates/organizations are not policy PII; many identifier types have no positive evaluation labels. Business/person heuristics can err. |
| OCR | Concatenation, boundaries, bad transcription and block geometry can prevent discovery/mapping; native mixed-image regions are not comprehensively OCRed. |
| Office coverage | Unsupported controls, tracked changes, dynamic fields, properties, alt text, charts, embeds, master/layout content, annotations and image text can remain. |
| PDF coverage | Unverified images, annotations/metadata and unsupported native geometry; OCR raster rebuilding changes typography/features; optional visible tags can be omitted when too small. |
| Visual coverage | Single-frame PNG/JPEG only; unsupported vectors/charts/OLE/masks/inline/unsafe mappings are inventoried. No manual missed-face region annotation. |
| Layout | Library-readable output and unchanged supported structure are checked, not rendered Word/PowerPoint fidelity or automatic overflow repair. |
| Review semantics | An empty known queue is not full source review; no action enables safe/export/coverage flags. Unknown/unresolved/unsupported values can remain in downloadable private copies. |
| Persistence | Local sequential snapshots and source-path retention; no automatic migrations, multi-user transactions, append-only signatures or cross-version entity resolution. |
| Frontend | All metadata loaded for maps/charts; large sets cost pagination time. Source selections cannot span separate windows. Synchronous work has no real-time stage feed/cancel. |
| Operations | Windows 11/Python 3.13.5 verified; Windows 10, other interpreters/platforms, a fresh second host and network-disabled offline operation not independently certified. |
| Historical evidence | Old report claims span different workspaces/test scopes; private inputs/caches may be absent. Saved byte fingerprints can differ after edits/line-ending conversion, so reuse must validate actual evidence. |
| Hygiene | Tracked bytecode, experimental scripts and optional legacy report generators remain; this task deliberately changes Markdown only. |

The initial audit's missing-attribution statement is superseded: the current source commit includes the retained YuNet attribution. Earlier “no Git metadata” or “server is running” statements are workspace/handoff observations, not current universal project facts. Retired absent-Qwen/vanilla assets are not current installation requirements. Historical aggregate measurements are preserved with their scope rather than blended into one alleged current accuracy score.

## 19. Current development status

| Requirement | Implementation status | Verification evidence | Remaining work |
| --- | --- | --- | --- |
| Content identification and valid supported formats | Implemented | File intelligence regressions and real PDF/Office HTTP uploads | Broader hostile/unusual-package validation. |
| Native/scanned/hybrid PDF | Implemented on supported routes | Real stabilization OCR/engine/HTTP matrix; PDF regressions | OCR ground truth and mixed-page/geometry coverage. |
| DOCX/PPTX extraction/artifacts | Implemented for mapped structures | Office/boundary/artifact regressions and actual reopened downloads | Unsupported structures and visual-layout certification. |
| Presidio/custom employee rules | Implemented, policy 1.1 | Default analyzer, exact recognizer tests, seeded retained evaluation | Labeled real-document discovery/recall and unsupported entity gold data. |
| Face discovery/blur | Implemented for mapped media | Real YuNet, per-occurrence/tamper tests, Chrome serialized blur | Real labeled accuracy, other media formats and manual face annotation. |
| Exact redaction and artifact verification | Implemented, scope-limited | Destructive/raster/OOXML/visual checks, feedback regeneration | Whole-document independent coverage and unsupported-content remediation. |
| Human decisions/evidence/revisions | Implemented | All three actions, Unicode offsets, conflict/replay tests and browser workflows | Identity/role attestation for any future multi-user product. |
| Save/restart/reload/source checks | Implemented for local retained originals | Real revision-2 restart/reload and altered-source 409; audit tests | Migration/retention/backups and cross-process transactions if needed. |
| React UI and optional Vite proxy | Implemented | 40 React/four Node, lint/types, Chrome and real proxy upload | Formal accessibility/usability/performance assessment. |
| Supported Windows installation | Stabilized | Constrained fresh resolution, installed model/native engines, actual launcher/config | Other Windows/Python/platform/second-machine validation. |
| Multi-user/authenticated deployment | Absent | Source inspection; no identity service/roles/transactions | A separately designed and authorized deployment scope. |
| Background jobs/progress/cancel | Absent | Synchronous pipeline/API; no broker/worker | Proposed only if workload requirements justify it. |
| Complete PII coverage / downstream approval | Not implemented or verified | False clearance flags and explicit residual/unsupported evidence | Independent labeling, coverage policy and trustworthy approval contract. |
| Documentation consolidation | Two authorities plus attribution exceptions | Inventory/data preservation/link and current regression checks | Keep README/report synchronized; generated metrics remain JSON evidence. |

## 20. Future development

These are **recommended directions**, not committed schedules or implemented capabilities:

1. Build an independently labeled, privacy-controlled real-document benchmark covering OCR errors, names/boundaries, unsupported structures and all policy identifiers. Keep reviewer demonstrations separate from gold labels and measure exact/coverage errors explicitly.
2. Investigate source-based discovery of known OCR-concatenated identifiers/names and mixed-page image text, with held-out evidence rather than literal document-specific exceptions. Track partial-email risk and the unchanged account/license review burden.
3. Expand deterministic mapping/media support only with source-integrity and serialized-output proof; add explicit manual face-region review before claiming support. Preserve conservative unsupported inventories.
4. Validate actual rendered Office layouts, accessible keyboard/assistive-technology interactions and larger metadata workloads. Improve performance/progress/cancellation only with a deliberate compatible job model.
5. Define retention, secure temp, encrypted backups, threat model and identity/authorization requirements before considering network/multi-user use. Do not convert the existing local header into a credential or disable ACL/coverage checks for convenience.
6. Establish explicit schema/policy migrations and signed/auditable approval semantics if real requirements demand them. Review completion alone must not set coverage/export flags.
7. Validate fresh machines/other supported Windows builds and enforce the intended Python support matrix in an approved packaging workflow. Review the nonblocking TestClient deprecation deliberately rather than changing dependency versions during documentation maintenance.
8. Consider tracked bytecode/experiment cleanup and modernizing legacy report generation as separate hygiene tasks. This consolidation leaves non-Markdown tooling/data intact and centralizes authoritative prose here.

## 21. Final engineering assessment

| Dimension | Evidence-based assessment |
| --- | --- |
| Functional completeness | Supported local inspection, correction, artifact and audit workflows are implemented. Unsupported source/media/coverage areas remain explicit. No completion percentage is defensible. |
| Runtime readiness | The stabilized Windows 11 x64/Python 3.13.5 setup has actual engine/API/browser/restart evidence; one Python process serves the normal app. Other host combinations remain to be validated. |
| Integration readiness | Ready for the tested trusted-local workflow; structured failures, source/revision checks and artifact proof are present. No enabled downstream AI or remote integration is claimed. |
| Security maturity | Useful local boundary, private ACLs, bounded views, no-store/CSP and consistency checks; no authenticated multi-user, encrypted/signed or comprehensive adversarial-security guarantee. |
| Production readiness | Not established for a networked or organization-wide deployment. Detection/coverage limitations, plaintext local persistence and operational controls require explicit further work. |
| Maintainability | Two coherent authorities replace 21 redundant Markdown documents; source/config/JSON evidence and required attribution remain. Historical/verified/proposed claims are separated and reproducible CMD commands are centralized. |
| Remaining risk | Undiscovered PII, unsupported media/structures, OCR/span errors, incomplete independent labels, source-retention dependencies, resource limits and unvalidated environments. Passing regressions does not remove these risks. |

**Documentation consolidation is complete.** The post-deletion checks in section 15 passed, and the final in-scope Markdown inventory contains exactly the two authorities plus the two disclosed attribution exceptions. Git changes comprise one added report, one rewritten README and 21 deleted redundant Markdown files; nothing was committed or pushed. No application algorithms, frontend UI, dependency versions, API contracts, tests, models, original data, Git branches or remotes were changed as part of this operation.
