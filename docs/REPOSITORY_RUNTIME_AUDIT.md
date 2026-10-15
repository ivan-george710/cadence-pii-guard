# Repository runtime audit

Audit date: 15 October 2026 (user timezone: Asia/Calcutta). Checkout: `C:\Users\ivang\cadence-pii-guard`. Commit: `316d3cc96c53e7a17f781ebc9575822429c8d2c8`. Initial Git working tree was clean. No `AGENTS.md` was found in the checkout. This report audits the current source, not a presumed Optiv deployment.

**Overall complete-application readiness: NOT READY.** A real pip resolver run rejected the checked-in requirements. The frontend is runnable, but the complete Python-backed workflow could not be launched with the audited installation instructions. This does not establish that the document algorithms are defective: Python dependency installation is blocked and the available global environment is incomplete.

The audit created documentation only among intended tracked changes. It did not edit source, manifests, lockfiles, static assets, remotes or branches, commit/push, provision cloud resources, access production databases, or upload documents externally. Runtime verification used local frontend tests and HTTP requests. No confidential real-document validation tools were executed.

## 1. Architecture and component inventory

The product calls itself **Cadence PII Guard** (`FastAPI` title, React package and README). There is one deployable Python application and one frontend source package. This is not a multi-service backend or a cloud-database application.

| Area | Current implementation and entry points |
| --- | --- |
| Launcher | Root `run.py`, guarded by `if __name__ == '__main__'`; Uvicorn imports `app.main:app`. |
| ASGI app | `app/main.py` imports `create_app` from `app/web/server.py`. `create_app()` prepares private storage, installs the local boundary middleware, mounts static files, and constructs a lazy `DocumentPipeline`. |
| Frontend | `frontend/package.json`, React/ReactDOM 19.2.0, TypeScript, Vite 7.3.7, Tailwind plugin, Motion, Radix UI, Lucide and Visx. `frontend/index.html` -> `src/main.tsx` -> `src/App.tsx`. |
| Frontend features | `components/review-workspace`, `visual-privacy`, `artifact-download`, `evidence`, `analytics`, `product-story`; UI and Bklit component collections; `hooks/useWorkspace`; API/types/review/Unicode helpers in `src/lib`. Synthetic detector-quality JSON is bundled at build time. |
| Served frontend | Committed `app/web/static/index.html` plus `bundle` JS/CSS. FastAPI serves `/` and `/assets`. A fresh Node build matched these three files after CRLF normalization. Node is unnecessary just to serve them. |
| API models | Pydantic request/response models in `app/web/models.py`; bounded source/findings views, feedback, revisions, saved references. |
| Orchestration | `app/pipeline/pipeline.py` and `models.py`. Synchronous calls, lazy engine initialization, stage timings, sanitized failure results, explicit audit persistence and artifact regeneration. |
| File intelligence | `app/engines/file_intelligence`: libmagic MIME detection, OOXML package checks, PDF text/image classification. Recognizes images, but extraction/web upload do not support standalone images. |
| Extraction | `app/engines/extraction`: pypdf native PDF text, RapidOCR/ONNX OCR plus Poppler rendering, hybrid PDF route, python-docx/lxml DOCX mappings, python-pptx/lxml PPTX mappings. |
| Detection | `app/engines/pii_detection`: in-process Presidio `AnalyzerEngine()`, English spaCy model, custom `EMP-` plus five digits recognizer, deterministic validation and CONFIRM/REJECT/REVIEW disposition. `presidio.py` is empty; `engine.py` is the implementation. No LLM calls. |
| Visual privacy | `app/engines/visual_detection`: image inspection and placement, CPU OpenCV YuNet face detection, hash-checked committed ONNX asset, face review/audit models, cropping/blur support. This is an additional module beyond the historical five-engine description. |
| Redaction/artifacts | `app/engines/redaction`: tag generation, overlap resolution, text regeneration, DOCX/PPTX package editing, PyMuPDF PDF artifacts, visual blur, and independent output checks. Review artifacts may be partial; export/coverage clearance remains false. |
| Audit | `app/engines/audit`: baseline/revision/event validation, explicit reviewer feedback, reporting, atomic local JSON storage, optional offline feedback export. |
| Configuration/utilities | `app/config/settings.py` and config package are empty placeholders; `app/utils` contains a logging helper. No central settings loader or env-driven deployment layer. |
| Tests | 16 top-level Python test modules, PDF fixtures, frontend Node/Chrome harnesses, and one Vitest React test file. Some tests inject engines/analyzers rather than exercising default ML dependencies. |
| Historical evidence | Root `ENGINE*`, `PHASE*`, `PIPELINE_REPORT`, `CADENCE_REAL_DOCUMENT_VALIDATION`, evaluation JSON/Markdown, and phase diagnostics. Retained reports are not execution evidence for this audit. |
| Experimental files | Root `scratch_*`, `scratch_page.png`, and `createfile.py`; not application entry points. Several contain old absolute paths. `createfile.py` regenerates/overwrites a tracked fixture. |
| Generated/cache artifacts | Tracked CPython 3.13/3.14 and pytest `.pyc` files exist. They neither supply dependencies nor prove current tests pass. |

### Communication and stage order

```text
Browser
  |-- bundled mode: HTTP 127.0.0.1:8765 (UI and relative /api requests)
  `-- development: Vite 127.0.0.1:5174/assets/
         `-- /api proxy -> 127.0.0.1:8765
                  |
             FastAPI LocalBoundary -> route/revision validation
                  |
             synchronous DocumentPipeline
                  |-> file intelligence
                  |-> extraction (text / OCR / hybrid PDF, DOCX, PPTX)
                  |-> face detection/visual inspection
                  |-> PII detection
                  |-> text redaction
                  |-> audit construction
                  |-> artifact redaction
                  `-> artifact verification
                  |
           private uploads/artifacts + explicit saved JSON audits
```

The actual pipeline runs `face_detection` before `detection`, even though the API's display-stage constant lists detection first. Feedback replays audit decisions and regenerates redaction/artifacts from the original extraction/source rather than redoing detection. Source SHA-256 checks guard regeneration. An RLock serializes shared review operations; there is no job queue or background worker. Normal FastAPI synchronous route thread handling is not an independent service.

## 2. Database and persistence architecture

No PostgreSQL, SQLite, Redis, Supabase, ORM, database client, migration, seed, Docker service, task scheduler or broker was found in the application/configuration. No cloud service credentials are required.

`create_app()` uses `.private_review_app` under the resolved repository root, with `uploads`, `artifacts`, and `audits`. The `sessions` and `saved_refs` dictionaries are process-local; saved references are re-created when listed. Explicit saves write `audit_<document hash>.json` using temporary-file/fsync/atomic replacement with revision checks. JSON snapshots include sensitive original extraction and audit material. Reload needs the original upload at the stored path and regenerates a review artifact. Unsaved revisions disappear on process restart. Multiple Uvicorn workers would not share sessions/locks and are not an established deployment configuration.

Standalone library defaults are `.private_audit`, `.private_artifacts`, and `.private_feedback`; web startup overrides the audit/artifact paths. Private directories are protected by symlink/junction/UNC/public-path checks, Windows ACLs or POSIX mode checks, and local `.gitignore` files. These filesystem protections are source-confirmed; end-to-end ACL creation/persistence was not executed because application imports are blocked.

## 3. API inventory and authorization

`app/web/server.py` defines the following endpoints. There is no separate API service for the engine.

| Method | Path (prefix `/api` unless specified) | Purpose |
| --- | --- | --- |
| GET | `/` (no prefix) | Committed UI HTML. |
| GET | `/config` | Upload cap/formats/trusted-local flag; usable liveness check. |
| GET, POST | `/documents` | List in-memory sessions; upload one multipart `file`. |
| GET | `/documents/{identifier}` | State, stage status and sanitized summary. |
| GET | `/documents/{identifier}/review` | Review identity and source location count. |
| GET | `/documents/{identifier}/findings` | Filtered/paginated finding list. |
| GET | `/documents/{identifier}/findings/{finding_id}` | Bounded finding/source detail. |
| GET | `/documents/{identifier}/findings/{finding_id}/visual-crop` | Bounded face crop, optionally after redaction. |
| GET | `/documents/{identifier}/findings/{finding_id}/comparison` | Actual replacement comparison. |
| GET | `/documents/{identifier}/source` | Bounded source window/highlights. |
| POST | `/documents/{identifier}/feedback` | Confirm/false-positive decisions with expected revision. |
| POST | `/documents/{identifier}/missed-pii` | Manual source-offset finding with expected revision. |
| GET | `/documents/{identifier}/artifact` | Current private review copy; optional inline PDF. |
| GET | `/documents/{identifier}/audit` | Audit summary and feedback events. |
| POST | `/documents/{identifier}/save` | Explicit save with revision conflict checks. |
| GET | `/saved-audits` | Opaque references to saved snapshots. |
| POST | `/load-audit` | Restore using an opaque listed reference. |

`/assets` is the static mount. FastAPI docs, ReDoc and OpenAPI are disabled. `/health` is absent. JSON requests are serialized in `frontend/src/lib/api.ts`, use same-origin credentials, and carry `X-Cadence-Local: 1`; multipart uploads omit the automatic JSON Content-Type. Vite rewrites only Origin `http://127.0.0.1:5174` to the backend Origin. It is not an arbitrary cross-origin API configuration.

There is **no login, user database, JWT/session authentication, SSO or role-based authorization**. The loopback launcher, Host/Origin checks and required mutation header define a trusted single-reviewer local boundary. `reviewer_id` is supplied by the client, defaults to `local-reviewer`, and is not an authenticated identity. UUID session/reference identifiers are opaque locators, not proof of user identity. Authentication cannot be classified as tested login functionality because no such feature exists.

## 4. Dependency inventory

The exact complete Python declaration is [requirements.txt](../requirements.txt): **161 pinned distributions**, including many transitive/experimental packages. The exact frontend resolution is [frontend/package-lock.json](../frontend/package-lock.json), supplemented by the scripts/ranges in [package.json](../frontend/package.json). No Python lock with artifact hashes, Python version declaration, `pyproject.toml`, Node `engines` field, `.nvmrc`, `.env` template, Dockerfile or CI YAML was found. Dependency presence is not evidence of application use.

| Direct runtime role | Declared Python packages/versions |
| --- | --- |
| HTTP and models | fastapi 0.143.0, uvicorn 0.54.0, starlette 1.7.0, pydantic 2.13.5, python-multipart 0.0.32 |
| File identification | python-magic 0.4.27, python-magic-bin 0.4.14 (both expose `magic`) |
| PDF parsing/rendering/artifacts | pypdf 6.19.0, PyMuPDF 1.28.2, pdf2image 1.17.0; external Poppler binaries |
| OCR | rapidocr-onnxruntime 1.2.3 and onnxruntime 1.30.0 with image/numeric dependencies |
| Office extraction/packages | python-docx 1.2.0, python-pptx 1.0.2, lxml 6.1.3 |
| Images/face processing | pillow 12.3.0, numpy 2.3.5, OpenCV `cv2`; both opencv-python 5.0.0.93 and opencv-contrib-python 4.10.0.84 are pinned |
| PII/NLP | presidio_analyzer 2.2.364, spacy 3.8.16, phonenumbers 9.0.40 and analyzer transitive dependencies; separately installed `en_core_web_lg` |
| Tests/fixtures | pytest 9.1.1, reportlab 5.0.1, httpx 0.28.1; Faker appears in evaluation tooling |

App-source import enumeration found third-party roots `PIL`, `cv2`, `docx`, `fastapi`, `lxml`, `magic`, `numpy`, `pdf2image`, `pptx`, `presidio_analyzer`, `pydantic`, `pymupdf`, `pypdf`, `rapidocr_onnxruntime`, and `starlette`, plus standard library and local app imports. Uvicorn is in the root launcher.

`presidio_anonymizer` is declared but the actual redactor is custom. `doclang`, `docling-core`, `docling-parse`, PaddleX, cloud SDKs, pandas/openpyxl/xlsxwriter, tree-sitter packages, pywin32, multiprocess/mpire, and other broad requirements do not establish extra runtime services. In particular, `scratch_docling.py` imports `docling`, which is not itself declared, and `scratch_paddle.py` imports `paddleocr`, also not declared. Their additional installations are not necessary for the main app. Installed cloud SDKs are not evidence of cloud processing integrations.

### Confirmed installation failure

Executed on Python 3.13 with pip 25.1.1:

```cmd
py -3.13 -m pip install --dry-run --no-cache-dir -r requirements.txt
```

Exit 1: `ResolutionImpossible`. Resolver evidence:

```text
The user requested PyYAML==6.0.3
paddlex 3.7.2 depends on PyYAML==6.0.2
```

Other packages in the reported conflict (`docling-core`, Hugging Face Hub, OmegaConf) accept wider PyYAML ranges; the two exact requirements above alone cannot both be satisfied. The dry run inspected package metadata and used build isolation for source metadata; it did not install application packages into the global environment. It was not a clean-venv installation, but the incompatible exact constraints are independent of already installed packages. Further dependency conflicts could surface once this one is fixed.

No alternate requirements set, ignored-dependency install, PyYAML downgrade or code workaround was applied. Reconciliation is a maintainer follow-up. The overlapping `cv2` and `magic` distributions are additional packaging risks, **not reproduced import failures after a successful installation**.

### Models, external dependencies and network

YuNet file `app/engines/visual_detection/assets/face_detection_yunet_2023mar.onnx` is 232,589 bytes; computed SHA-256 was `8f2383e4dd3cfbb4553ea8718107fc0423210dc964f9f4280604804ed2552fa4`, exactly the detector constant. Its license file is present. The detector's docstring references `assets/ATTRIBUTION.md`, which is absent. Hash verification succeeded; inference did not run.

The default Presidio analyzer uses its spaCy setup, and app analysis requests `language='en'`. The repository does not pin a language-model wheel. [Presidio's installation instructions](https://microsoft.github.io/presidio/installation/) specify `en_core_web_lg` for the default engine and list Python through 3.13; Python 3.13 is therefore the selected setup target, not an inferred successful runtime from tracked bytecode. RapidOCR initialization is unconditional when the extraction engine is constructed, even for DOCX/PPTX, although extraction-engine construction itself is lazy. Thus its imports/assets are not optional for the default application workflow.

Poppler is necessary for OCR PDF rendering because `convert_from_path` is used without `poppler_path`. Both executables ran at version 26.09.0. [pdf2image installation](https://pdf2image.readthedocs.io/en/latest/installation.html) documents adding Windows binaries to PATH. Tesseract was absent and is not used. No CUDA, GPU, Redis, database, Microsoft Office automation, LLM or external HTTP service is called by the application source. Package/model downloads require network access during setup. Dependency-internal network behavior was not inspected at runtime, so complete offline operation is **NOT VERIFIED**.

## 5. Environment variable inventory

There are **zero mandatory application-specific environment variables** and no required `.env` files. `python-dotenv` and `pydantic-settings` being declared does not mean the app uses them. Backend host/port/storage and Vite proxy values are source constants or Python factory arguments.

| Name | Reader/scope | Meaning/default |
| --- | --- | --- |
| `PATH` | OS/executable discovery | Python, Node, Windows PowerShell, Poppler. |
| `TEMP`, `TMP` | Python temp/validation helper | OS temp defaults; `validate_cadence_document.py` temporarily redirects these to restricted storage. Web OCR does not do so automatically. |
| `CADENCE_UI_URL` | Browser harnesses | Wrapper defaults 8765, phase10b defaults 8766, phase11 defaults 8767. Does not start/configure the backend. |
| `CADENCE_UI_FIXTURE` | Browser harnesses | Test document path; many defaults point to absent private real documents. |
| `CADENCE_TEST_CHROME` | Browser harnesses | Chrome executable, Windows Program Files default. |
| `CADENCE_UI_SYNTHETIC` | Browser harnesses | Synthetic-mode assertions when `1`. |
| `CADENCE_UI_COMPLETE` | Selected phase10b variants | Complete-review branch when `1`. |
| `CADENCE_UI_REPRO` | Selected phase10b variants | Reproduce earlier UI issue when `1`. |
| `FLAGS_use_onednn` | `scratch_paddle.py` assignment | Experimental Paddle setting, value `0`. |
| `PADDLE_DISABLE_ONEDNN` | Same experiment | Value `True`. |
| `PADDLE_PDX_DISABLE_MODEL_SOURCE_CHECK` | Same experiment | Value `True`. |

No secret values were read from `.env` files or printed. No credential inventory is needed to run the core application.

## 6. Startup dependency graph and scripts

```text
Consistent Python dependency declaration (CURRENTLY BLOCKED)
  -> project venv + dependencies + model installation + native executables
     -> python run.py (imports and private-directory ACL setup)
        -> 127.0.0.1:8765 UI/API
           -> first upload initializes local OCR/face/NLP processing

Optional frontend editing:
  Node + npm ci -> npm run dev -> 127.0.0.1:5174/assets/
                                     -> API requires backend on 8765
```

No DB initialization or migration precedes launch. There is no separate engine daemon, worker, queue consumer or scheduled process. Port 8765 is fixed by `run.py`; Vite's hardcoded target is 8765 and its fixed `strictPort` is 5174. Ports 8766/8767 occur in historical acceptance harnesses, not the normal launcher.

| Command/file | Purpose and limits |
| --- | --- |
| `python run.py` | Only checked-in application launcher; also serves static UI. No separate production-service configuration. |
| `npm ci` in frontend | Lockfile-based frontend dependency install. Audit used `--ignore-scripts --no-audit --no-fund`; native binaries still loaded successfully in tests/build. Standard lifecycle-enabled installation was not separately tested. |
| `npm run dev` | Vite loopback development server. |
| `npm run build` | TypeScript check and Vite production bundle; empties `app/web/static`. Audit used isolated output instead. |
| `npm test`, `npm run lint` | Vitest and ESLint. |
| `python -m pytest tests -q` | Python regression suite; some fake/injected engines. |
| `node --test tests/frontend/core.test.mjs` | Four Unicode/payload/stage helper tests. |
| `tests/frontend/browser_smoke.mjs`, `phase10b_browser.mjs`, `phase10b1_browser.mjs`, `phase10b2_browser.mjs`, `phase11_browser.mjs` | Real-Chrome acceptance via debugging protocol. Need running Python, suitable synthetic/private fixture, Chrome and `.venv/Scripts/python.exe` for artifact checks; retain private screenshots/downloads. Not run here. |
| `tools/create_phase10b_fixtures.py`, `create_phase11_fixtures.py` | Synthetic review/visual fixtures; not startup prerequisites. |
| `tools/validate_cadence_document.py`, `validate_office_documents.py`, `validate_phase8_documents.py`, `validate_phase11_documents.py` | Historical document runs; depend on absent private inputs/caches. |
| `tools/validate_audit_cache.py`, `inspect_cadence_evidence.py`, `check_phase11_inventory.py` | Private evidence/audit inventory diagnostics. |
| `tools/build_detector_dataset.py`, `evaluate_detector.py` | Synthetic dataset generation/evaluation, not scheduled workers. |
| `tools/compare_detector_documents.py`, `inspect_detector_comparison.py` | Private/historical detector comparisons. |
| `tools/export_reviewer_feedback.py`, `export_detector_quality.py` | Offline feedback curation and aggregate UI metrics export. The latter writes frontend source data; not run for this audit. |
| `tools/write_phase10a_report.py`, `prepare_review_browser_check.py` | Regenerate a historical report or browser-test source; would modify existing files, so not executed. |
| `scratch_docling.py`, `scratch_paddle.py`, `scratch_extraction.py`, `scratch_ocr_pdf.py`, `scratch_hybrid.py` | Experiments, frequently old paths/dependency assumptions. Not a supported full application launch. |

## 7. Windows compatibility findings

1. **Paths:** application code mostly uses `pathlib`, root resolution from `__file__`, and proper POSIX paths for internal ZIP members. Root startup and `cd /d` avoid drive/current-directory mistakes. Windows storage forbids UNC paths and traversal through symlinks/junctions. Scratch files have user-specific absolute paths; `scratch_hybrid.py` uses Windows backslashes. They are not cross-platform launchers.
2. **Shells:** no Bash installer/launcher was found. CMD can activate `.venv\Scripts\activate.bat`. Python internally calls `powershell`, not `pwsh`, to apply ACLs. Its executable was present; actual application ACL creation was not reached.
3. **Python:** default Python was 3.14.6; 3.13 and 3.12 were also installed. Both inspected 3.12/3.13 environments lacked core runtime imports. There is no declared support matrix; 3.13 syntax parsing succeeded, full install did not. Versioned launcher commands prevent accidental interpreter selection.
4. **Native wheels:** dry-run resolution found Windows CPython 3.13/ABI3 metadata for many native packages. This is not proof those DLLs load together. `pywin32` and `python-magic-bin` make the blanket requirements Windows-oriented. No native-runtime alternative was verified; recommending Docker/WSL would not reconcile the PyYAML pin conflict and there is no supplied container setup.
5. **Ports:** no listeners were found at 8765 or 5174 before testing. Vite started on 5174, then was stopped by the audit. Backend never bound 8765. Use netstat/tasklist rather than killing arbitrary processes.
6. **Concurrency:** one Uvicorn process, connection limit four, synchronous CPU-heavy work, local RLock. No multiprocessing worker launch or Unix `fork` requirement in application source. RapidOCR/ONNX/OpenCV internal threading was not exercised. Slow first inference is possible because engines initialize lazily.
7. **OCR/models:** Poppler executables were usable. OCR inference, spaCy model loading and face inference were not exercised. No Tesseract dependency. Office images have no text OCR; standalone image uploads are rejected.
8. **Frontend:** compatible Node 22.16.0/npm 11.12.0 ran Vite, TypeScript, ESLint and tests. Vite's dev URL includes `/assets/`; the trusted proxy origin requires `127.0.0.1`.
9. **Line endings:** rebuilt HTML/JS/CSS differ bytewise from checked-in Windows files but match after CRLF normalization. Static assets were left unchanged.

## 8. Commands actually executed and evidence

The runtime commands below were invoked through `cmd /c` in the supplied Windows environment. Source inspection additionally used Git, ripgrep, file listings/reads and Python AST/import enumeration; no historical report result was adopted as a fresh test result.

| Command/check | Observed result |
| --- | --- |
| `git status --short`; `git rev-parse HEAD` | Initially clean; commit recorded above. |
| `python --version`; `py -0p`; `py -3.13 --version`; `git --version` | Default Python 3.14.6; selected Python 3.13.5; launcher also exposed 3.12. Git 2.45.1.windows.1. |
| `node --version`; `npm --version` | 22.16.0; 11.12.0. |
| `where pdftoppm`; `where pdfinfo`; `pdftoppm -v`; `pdfinfo -v` | Installed Poppler binary directory; both version 26.09.0. |
| `where tesseract`; `where powershell` | Tesseract absent (not required); Windows PowerShell present. |
| `py -3.13 -m pip --version`; `py -3.13 -m pip check` | pip 25.1.1; no broken installed requirements. This global-env check does not establish project installation. |
| Import availability probes on Python 3.13 and 3.12 | `magic`, RapidOCR, Presidio, spaCy, `en_core_web_lg`, `cv2`, PyMuPDF absent. 3.13 had FastAPI, Uvicorn and pytest. |
| `py -3.13 -m pip install --dry-run --no-cache-dir -r requirements.txt` | Failed exit 1, PyYAML/PaddleX `ResolutionImpossible`. |
| `py -3.13 run.py` | Traceback ends `ModuleNotFoundError: No module named 'magic'` from `file_intelligence/detector.py`; no listener. |
| `py -3.13 -m pytest tests -q -p no:cacheprovider --collect-only` | 15 collection errors, zero tests collected; missing magic, reportlab, Presidio, pypdf, PIL, NumPy, etc. No Python test is claimed passed. |
| `node --test tests\frontend\core.test.mjs` | Four tests passed, zero failed. |
| `npm ci --ignore-scripts --no-audit --no-fund` in frontend | Exit 0, 383 packages installed in 29 seconds. No manifests/lockfile changed. No npm vulnerability audit was requested/run. |
| `npm test` in frontend | One file, **40 tests passed**, 14.46 seconds. Fetch is mocked in these React tests. |
| `npm run lint` in frontend | Exit 0. Existing configuration excludes UI/Bklit source folders and disables exhaustive-deps; this is not universal static coverage. |
| `npx tsc --noEmit` in frontend | Exit 0. |
| `npm run dev` in frontend | Vite 7.3.7 ready; `http://127.0.0.1:5174/assets/`. |
| `curl.exe -s -o NUL -w "Vite HTTP %{http_code}\n" http://127.0.0.1:5174/assets/` | HTTP 200. |
| Same HTTP status probe at `http://127.0.0.1:5174/api/config` | HTTP 500; Vite logged `ECONNREFUSED 127.0.0.1:8765`. |
| `curl.exe -s --max-time 3 -o NUL -w "Backend HTTP %{http_code}\n" http://127.0.0.1:8765/api/config` | HTTP 000/connection failure; backend absent. |
| `npx vite build --outDir node_modules/.cadence-audit-build` in frontend | Exit 0; 2,227 modules; generated HTML/JS/CSS; 3.88 seconds. Source/static files preserved. |
| Python AST syntax parse across app/tools/tests/root Python | 95 source files parsed without syntax errors; 55 are under app. Not an import or execution test. |
| SHA-256 of committed YuNet | Matched detector constant; model size noted above. |
| Node comparison of isolated build against static files | All three match after CRLF normalization. |

One initial isolated-build attempt supplied a quoted `%TEMP%` output path through nested CMD/npx invocation and failed with an ENOENT malformed output path. The follow-up relative output path above succeeded. This was an audit-command quoting failure, not a frontend build defect. Initial ripgrep invocations with literal Windows `tools/*.py`/`scratch*.py` arguments also failed glob expansion; directory-based searches and Python enumeration supplied the inventory instead.

The combined probe that ran Python startup then the four Node tests returned the last command's exit status; startup failure is independently evidenced by its traceback and HTTP failure. Likewise the combined frontend-check shell returned the final TypeScript exit status; Vitest explicitly reported 40 passing tests and lint completed without diagnostics.

## 9. Functional verification matrix

Status describes the audited environment and test scope, not historical claims.

| Component/capability | Status | Evidence/blocker |
| --- | --- | --- |
| Frontend install/dev/static delivery | VERIFIED | Lockfile install, Vite ready, dev HTML HTTP 200. |
| Frontend UI behavior | PARTIALLY VERIFIED | 40 mocked-API React tests and four core Node tests passed; no real-browser end-to-end execution. |
| Frontend build/types/lint | VERIFIED | TypeScript and configured lint passed; isolated Vite bundle built. |
| Full Python dependency installation | BROKEN | Resolver rejected incompatible exact PyYAML requirements. |
| Backend launch in available 3.13 environment | BROKEN | Actual launch fails on missing `magic`; application cannot bind. This is an environment failure, not a proven backend logic defect. |
| Python engine loading | NOT VERIFIED | Imports blocked before pipeline can be used. Syntax parsing is insufficient. |
| Backend health/config | NOT VERIFIED | `/api/config` could not connect. Route existence/response confirmed only in source. |
| Frontend-to-backend communication | BROKEN in this environment | Real Vite request returned 500/ECONNREFUSED because backend absent; mocked fetch tests are not integration evidence. |
| Uploads | NOT VERIFIED | Actual multipart endpoint unavailable. Mocked frontend upload/header test passed. |
| Document extraction/OCR | NOT VERIFIED | Runtime dependencies absent; Poppler executable checks alone passed. |
| PII detection | NOT VERIFIED | Presidio/spaCy/model absent; default analyzer did not run. |
| Text redaction | NOT VERIFIED | No Python test collected and no real document pipeline run. |
| Face detection/blur | PARTIALLY VERIFIED | Model exists and hash matches; no inference or serialized blur verification run. |
| Review/sanitized artifact generation/download | NOT VERIFIED | Frontend mocks cover link behavior; actual artifact generation unavailable. Outputs are private review copies, not guaranteed sanitized/export-cleared files. |
| Audit history/save/reload/traceability | NOT VERIFIED end to end | Mocked UI save/history tests passed; actual filesystem persistence and replay unavailable. |
| Authentication/authorization | PARTIALLY VERIFIED by inspection | No user authentication exists; local boundary/revision controls read, not live-tested. |
| Database connectivity | NOT VERIFIED / not applicable | No database connection exists to test. JSON persistence is the applicable capability. |
| Background services | NOT VERIFIED / not applicable | None implemented or required. |

## 10. Confirmed blockers, missing configuration and follow-ups

1. **Release blocker:** reconcile `paddlex==3.7.2` / `PyYAML==6.0.3` and rerun a clean Windows venv install. The main app does not import PaddleX directly; deciding how to revise requirements is outside this inspection task.
2. **Environment blocker:** install runtime dependencies and `en_core_web_lg` into a project venv. No `.venv` existed initially, and no project Python packages were installed by this audit. Available global environments are insufficient.
3. **Reproducibility risk:** no Python version declaration/model lock, broad requirements mixing experiments and runtime, duplicate `magic`/`cv2` providers. Further resolver/native compatibility issues remain possible.
4. **Documentation gaps now addressed:** README assumed an existing venv, used historical command snippets and private fixtures, and did not provide a complete fresh CMD setup/startup sequence. The companion guide explains actual single-process/static serving and its blockers.
5. **Unresolved test gap:** rerun full Python suite, default OCR/NLP/face initialization and synthetic PDF/DOCX/PPTX upload/review/save/restart/download tests once installation works. Test scanned and hybrid PDFs separately. This audit cannot turn those pending checks into successes through documentation.
6. **Historical tools:** private caches/real documents are intentionally not in the clone. No missing real document or secret should be added merely to make routine startup work. Empty config modules and absent `.env.example` reflect current lack of env configuration, not a requirement to invent values.

## 11. Security findings and practical limits

These are implementation observations and bounded risks; this was not a penetration test or dependency vulnerability scan.

- **Trusted local user only.** Host/Origin/mutation-header checks and loopback binding reduce browser cross-origin exposure, but do not authenticate local users/processes. Anyone able to reach the trusted local service can potentially act within it; reviewer IDs are client assertions. Do not deploy it to a network by changing the bind address and treating the header as authentication.
- **Sensitive storage persists in plaintext.** Source uploads, extraction/audit snapshots and review artifacts contain original or unresolved PII. ACLs, opaque filenames and gitignore do not encrypt them. There is no automatic retention cleanup, user account isolation within the app, or multi-user transactional database.
- **OCR uses normal OS temp by default.** The server keeps multipart data in memory until writing into its private directory, but `OCREngine` subsequently uses `TemporaryDirectory()` and renders original page images there. Automatic context cleanup exists; the web launcher does not redirect temp to restricted storage as one historical validation tool does. A private-temp option is described in the runbook and remains untested here.
- **Review copies are not universally sanitized.** Unsupported Office parts, embedded-image text, properties, notes/annotations, external relationships, unmapped structures and unresolved findings can remain. The source keeps `safe_for_llm`/coverage/downstream export false. Download availability is not proof of full PII removal. DOCX logical paragraph segments are not rendered physical pages.
- **Resource limits are partial.** A 30 MiB upload limit and bounded request/source views exist, with local serialization and four connections. Decompressed Office XML/package inspection and PDF rendering do not establish a universal CPU/memory/time bound for malicious documents. Media-size guards exist, but no complete adversarial resource-exhaustion test was run.
- **Defense-in-depth exists but was not live-tested here.** No-store, MIME sniffing protection, referrer/frame/CSP headers; XML external-entity/network restrictions; symlink/junction/private-root guards; source fingerprints; explicit revision checks; sanitized exception responses; access logs disabled by the launcher.
- **Artifact revision URL is not immutable version selection.** Frontend download URLs include `revision`, but the backend artifact handler accepts only `inline` and returns the current session artifact; unknown query parameters do not select an archived revision. No-store helps cache behavior; callers should not assume historical bytes from that query string.
- **Repository hygiene.** Tracked `.pyc` files are machine/interpreter-specific and can be modified by tests. They should not be mistaken for a packaged runtime. Root historical/evaluation evidence is not a substitute for current coverage testing. No credentials are reproduced in this report.

## 12. Audit effects, cleanup and readiness assessment

The audit installed 383 frontend packages under the already ignored `frontend/node_modules` for required verification; these remain available for development. It stopped the Vite process it started. The isolated build output was removed after comparison. Python import/collection changed existing tracked bytecode caches; because the initial tree was clean, only those audit-modified `.pyc` files were restored to their original Git content. No application data directories were created by a successful backend launch; no production/cloud data was touched. Package-manager transient metadata downloads/build isolation are ordinary tooling effects, not project dependency changes.

The complete application is **NOT READY from a fresh clone using its declared installation**, while the frontend is **verified runnable**. The shortest intended full startup is one root `python run.py` process followed by the browser at `http://127.0.0.1:8765/`; there is no additional database or Python-engine service. That command is source-correct but failed in the inspected environment. A guaranteed complete-app runbook cannot exist for this exact unresolved dependency set without a dependency change, which the task prohibits. [WINDOWS_CMD_SETUP_GUIDE.md](WINDOWS_CMD_SETUP_GUIDE.md) provides the exact conditional setup/start sequence, honest verification gates and fixes that can be made without altering application source.
