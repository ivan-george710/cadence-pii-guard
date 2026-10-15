# Windows runtime stabilization and verification

Date: 15 October 2026. Base commit: `316d3cc96c53e7a17f781ebc9575822429c8d2c8`, branch `main`. Validation host: Windows 11 x64, build `10.0.26200.9457`; Python **3.13.5**, pip **25.1.1**, Node **22.16.0**, npm **11.12.0**, Poppler **26.09.0**.

**Runtime status: READY on the verified Windows/Python configuration.** Dependency resolution, native imports, the actual launcher, all document engines, live HTTP workflows, two real-Chrome workflows, and audit save/restart/reload succeeded. No application algorithm, frontend behavior, API contract or security control was changed. Windows 10 was not tested separately; “READY” does not mean complete PII coverage or approval for network/multi-user deployment.

## Original failures and root causes

| Original result | Root cause and resolution |
| --- | --- |
| FAILED: pip `ResolutionImpossible` | The 161-package environment snapshot mixed unused experiments with runtime. `paddlex==3.7.2` required `PyYAML==6.0.2`, conflicting with `PyYAML==6.0.3`. The main application does not import PaddleX. The supported installation now includes only real runtime/test dependencies; the complete original snapshot is preserved separately. |
| FAILED: backend `ModuleNotFoundError: No module named 'magic'` | No project venv existed and global Python installations lacked the document dependencies. Created a new `.venv` using Python 3.13.5 and installed normally with dependency resolution. |
| Packaging risk: duplicate `magic` / `cv2` providers | The Windows environment now contains only `python-magic-bin==0.4.14` and `opencv-python==5.0.0.93`. MIME detection, all supported formats, real OCR, YuNet inference and blur were then tested. Neither overlapping alternate provider is installed. |
| Missing default NLP asset | Installed `en_core_web_lg==3.8.0` from its official release; pinned its URL and SHA-256. Model load and actual default Presidio analysis passed. |
| FAILED during scanned-PDF verification: `UnicodeEncodeError: 'charmap' codec can't encode character '\u2713'` | OCR had completed, but its completion print used a checkmark that could not be encoded into redirected Windows cp1252 output. Changed that one log message to ASCII `OCR complete:`. Retested the actual scanned/hybrid pipeline and HTTP uploads with output redirected. No OCR parameters or returned data changed. |

The original audit remains at [REPOSITORY_RUNTIME_AUDIT.md](REPOSITORY_RUNTIME_AUDIT.md) as historical evidence; its NOT READY assessment predates these authorized fixes.

## Files changed and purpose

| File | Change |
| --- | --- |
| `requirements.txt` | Replace the mixed snapshot with 19 selected Windows application dependencies plus the non-Windows `magic` alternative; include tested constraints. Keep original direct runtime package versions. |
| `constraints-windows-py313.txt` | Pin all 78 resolved runtime/test distributions for reproducibility, without installing test tools when only runtime is requested. |
| `requirements-dev.txt` | Add runtime plus pytest, httpx and reportlab for regression tests and fictional fixtures. |
| `requirements-models.txt` | Pin the existing default English model to official version 3.8.0 and its verified wheel SHA-256. |
| `.python-version` | Declare Python 3.13 as the selected minor version. The runbook explicitly uses `py -3.13`; other Python versions are not certified. |
| `docs/dependencies/requirements-original.txt` | Preserve the original snapshot for provenance. It is intentionally not included in any supported installation command. |
| `app/engines/extraction/ocr.py` | One ASCII-only completion-message change for redirected Windows output. |
| `tools/verify_windows_runtime.py` | Add repeatable synthetic checks against real default engines and the actual HTTP server, including restart/reload and fingerprint rejection. It does not replace any application module or modify existing tests. |
| `README.md` | Correct fresh CMD installation, dependency layout, bundled-UI startup, optional frontend workflow and checks. |
| `docs/WINDOWS_CMD_SETUP_GUIDE.md` | Update the previous audit guide to the working installation/startup and real verification commands. |
| `docs/WINDOWS_RUNTIME_FIX_REPORT.md` | This report. |

The pre-existing uncommitted audit documents were retained; the setup guide was updated as requested. No branch/remote changes, reset/clean, commit or push occurred. No existing environment was deleted. The committed frontend bundle, application processing rules and existing regression tests are unchanged.

## Dependency choices and compatibility

**VERIFIED — resolution and installation.** The focused requirements resolved in a newly created Windows CPython 3.13 venv. A second `--ignore-installed` dry run against the exact constraints resolved all **78** distributions successfully. Final `pip check` returned `No broken requirements found`, including after the English model was installed (79 installed non-pip distributions total).

The supported runtime preserves the original versions of FastAPI, Uvicorn, Starlette, Pydantic, multipart, pypdf, PyMuPDF, pdf2image, python-docx, python-pptx, lxml, Pillow, NumPy, RapidOCR, ONNX Runtime, OpenCV Python, Presidio and spaCy. Resolved transitive versions are explicitly recorded in the constraints file. Tests validate the resulting behavior; the old freeze is not silently treated as a coherent lockfile.

`python-magic-bin` bundles the Windows libmagic dependency and provides the `magic` API used by this source. Its platform marker excludes `python-magic` on Windows. The latter remains declared for other platforms, whose native installation was not tested.

RapidOCR requires the `opencv-python` distribution, and standard OpenCV provides the source's `FaceDetectorYN` API. Actual YuNet inference identified the fictional face, and serialized artifacts passed blur checks without `opencv-contrib-python`. This agrees with [OpenCV's packaging guidance](https://pypi.org/project/opencv-python/) to install one provider of the shared `cv2` namespace.

Unused Docling/Paddle/cloud/benchmark packages were not installed. Their old pins remain in the historical snapshot; those scratch experiments are not claimed as a supported executable environment. `presidio_anonymizer` is not used by this application's custom redactor. No Torch, Transformers, LLM, PaddleOCR, Tesseract, database, queue, Docker or GPU was introduced.

### Actual installation commands

From the root, using CMD:

```cmd
py -3.13 -m venv .venv
.venv\Scripts\python.exe -m pip install --dry-run -r requirements-dev.txt
.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
.venv\Scripts\python.exe -m pip install --dry-run --ignore-installed --report .venv\clean-resolution.json -r requirements-dev.txt
```

The model command was initially:

```cmd
.venv\Scripts\python.exe -m pip install -r requirements-models.txt
```

The official GitHub transfer was unusually slow. That transfer was interrupted after preserving its downloaded prefix; the **same official release asset** was completed using eight HTTP byte-range requests, checking each Content-Range, total length, ZIP CRCs and package version. No alternate model or dependency bypass was used. The completed wheel was installed with the normal resolver:

```cmd
.venv\Scripts\python.exe -m pip install -c constraints-windows-py313.txt .venv\en_core_web_lg-3.8.0-py3-none-any.whl
.venv\Scripts\python.exe -m pip check
```

Wheel evidence: **400,658,291 bytes**, SHA-256 **`293e9547a655b25499198ab15a525b05b9407a75f10255e405e8c3854329ab63`**. The model manifest now includes that hash on the official release URL. Version **3.8.0** loaded with spaCy **3.8.16**. Fresh developers should use the standard pinned model-manifest command; the range-transfer fallback was an acquisition workaround for this session's slow download, not an application change. A GitHub release-API metadata probe returned HTTP 403/rate-limit exceeded; release-asset downloads worked and no API credentials were needed.

No `pip install --no-deps`, forced installation or global Python-package modification was used. The environment and cached model wheel remain available; temporary transfer chunks were removed.

## Native environment and backend

| Check | Status/evidence |
| --- | --- |
| FastAPI, Uvicorn, Pydantic, document libraries | VERIFIED: imports succeeded in `.venv`. |
| libmagic | VERIFIED: native fixture MIME `application/pdf`; PDF/DOCX/PPTX uploads accepted by content validation. |
| pypdf/PyMuPDF | VERIFIED: text extraction, rendering, redaction and reopening. |
| python-docx/python-pptx/lxml | VERIFIED: real source extraction, mapped replacements and reopened packages. |
| RapidOCR/ONNX Runtime | VERIFIED: initialization and actual scanned/hybrid OCR. |
| OpenCV/YuNet | VERIFIED: model initialized, fictional face detected, actual transformed artifact checked. |
| spaCy/Presidio | VERIFIED: English model loaded; default analyzer confirmed fictional employee ID and email; full regression suite passed. |
| Poppler | VERIFIED: both `where` probes and version commands; actual scanned/hybrid rendering completed. |
| Private-storage protection | VERIFIED: launcher created restricted directories; `icacls .private_review_app` showed FullControl for current user and SYSTEM with inheritance to children. No ACL protection was bypassed. |
| Actual launcher | VERIFIED: `.venv\Scripts\python.exe -B run.py`, then final CMD activation plus literal `python run.py`; loopback 8765. `-B` only suppresses bytecode writes. |
| Configuration endpoint | VERIFIED: HTTP 200 and expected JSON shown below. |
| Bundled UI | VERIFIED: served by FastAPI; both real Chrome harnesses used port 8765. No Vite server is required for this mode. |
| Optional Vite | VERIFIED: port 5174 `/assets/`; proxied config and a multipart upload with Origin `http://127.0.0.1:5174` succeeded. All eight processing stages completed through the proxy. Stopped after verification. |

```json
{"max_upload_bytes":31457280,"supported_formats":["pdf","docx","pptx"],"trusted_local_only":true}
```

## Engine verification using fictional files

**VERIFIED — real default engines, no injected analyzer.** Fixtures and results are under `.private_review_app/windows-runtime-fix-20261015`. The fictional image is the repository's existing AI-generated test portrait, documented in `tests/fixtures/visual/README.md`.

| Fixture | Actual extraction route | Replacements | Output/evidence |
| --- | --- | --- | --- |
| Text PDF | `text` | 3 | Reopened; existing supported-content verification passed. |
| Scanned PDF | `ocr`, 3 blocks / 68 characters | 3 | Poppler and RapidOCR ran; redirected ASCII completion log succeeded; PDF reopened and verification passed. |
| Hybrid PDF | `text`, then `ocr` | 6 | Both pages processed and PDF output verified. |
| DOCX | Three logical paragraph segments | 3 | Package reopened, mapped text replacements verified. |
| PPTX | One slide | 3 | Package reopened, mapped text replacements verified. |
| Face PPTX | Two slides, one shared fictional portrait | 4 text replacements, 1 protected face | YuNet detected the face; actual requested blur verified in serialized media. |

Each format confirmed `EMPLOYEE_ID` and `EMAIL_ADDRESS` with real Presidio. Confirmed employee values were removed from the text-redaction output. Generated artifacts were reopened independently with PyMuPDF/python-docx/python-pptx. Artifact redaction and verification stages completed; supported-content checks passed. The face artifact's `face_transformations_verified` was true.

## Regression results

| Command | Result |
| --- | --- |
| `python -B -m pytest tests --collect-only -q -p no:cacheprovider` | VERIFIED: **302 collected**, zero collection errors. |
| `python -B -m pytest tests -q -p no:cacheprovider` | VERIFIED on the final source: **302 passed, 0 failed, 0 skipped, 0 errors**, 1 warning, 92.68 seconds. |
| Initial native/file/redaction/audit subset | VERIFIED: 95 passed, 7.15 seconds; included within the 302 unique tests, not additional coverage counts. |
| Post-logging-fix extraction regression | VERIFIED: 1 passed, 0.98 seconds; real OCR logging additionally covered by the engine and HTTP checks. |
| `node --test tests\frontend\core.test.mjs` | VERIFIED: **4 passed**, zero failures. |
| `npm test` | VERIFIED: **40 passed**, one test file, 12.19 seconds. These tests mock fetch and are supplemented by the real API/browser runs. |
| `npm run lint` | VERIFIED: exit 0 under the repository's existing lint scope. |
| `npx tsc --noEmit` | VERIFIED: exit 0. |

The initial full Python run passed all 302 tests in 86.77 seconds. After the ASCII logging change, the complete suite was rerun against the final source: all 302 passed in 92.68 seconds. The extraction regression and default scanned/hybrid engine and live HTTP checks also passed with redirected output. No existing tests were edited. `PYTHONDONTWRITEBYTECODE=1` was propagated to test subprocesses to preserve the tracked legacy bytecode files. No frontend build overwrote the committed bundle in this stabilization task.

Nonblocking warning: Starlette's TestClient reports that its httpx adapter is deprecated in favor of httpx2. Existing tests using pinned httpx 0.28.1 still passed. OpenCV 5 also prints `Targets are not supported by the new graph engine for now`; actual CPU detection/blur passed, so no second OpenCV distribution or warning-suppression workaround was added.

## Real HTTP, restart and browser workflows

**VERIFIED — live TCP/HTTP, not TestClient mocks.** The new verification tool submitted six fictional fixtures to the running launcher. Every fresh upload completed all eight displayed processing stages, had no `summary.failure`, and produced a supported-content-verified, downloadable artifact.

Verified operations:

1. Load frontend/configuration from the real server.
2. Upload text/scanned/hybrid PDF, DOCX/PPTX and face PPTX.
3. List findings, inspect the exact fictional employee candidate, and obtain source windows/highlights/location references.
4. Apply `FALSE_POSITIVE`, then `CONFIRM_PII`; verify revisions 1 and 2 and both audit events. A stale revision returned HTTP 409. A mutation without the local header returned HTTP 403.
5. Download and reopen each artifact; explicitly save the DOCX audit at revision 2.
6. Stop and restart the actual Python launcher. Obtain the new saved-audit reference, reload the saved snapshot and verify revision 2 and two feedback events.
7. Download/reopen the regenerated DOCX and verify its fictional employee identifier is replaced.
8. Deliberately append a marker to **only the synthetic upload created by this test**. Reload returned HTTP 409. Restore the original bytes in a `finally` block; reload then succeeded again. This tests the existing source-fingerprint protection rather than bypassing it.

The first new-verifier reload attempt incorrectly expected extraction/inference timings to be marked Completed. Inspection confirmed reload deliberately reuses saved evidence and runs only artifact regeneration/verification. Corrected the **new verifier's expectation**, not the application or existing tests, to require `persistence_load`, restored extraction/detection/redaction data, no failed stage, and completed artifact stages. The corrected restart/reload check passed.

### Real Chrome results

Both existing browser harnesses ran against the actual FastAPI-served committed UI and default engines:

| Harness | Result |
| --- | --- |
| `phase10b1_browser.mjs`, fictional PDF, synthetic/complete flags | VERIFIED: upload; disposition counts initially `[6,3,2,1]`; confirmation updated counts to `[6,4,1,1]`; automatic review selection, dialog, regeneration controls and revision-aware download passed; final revision **2**, known review queue complete; downloaded **1,222-byte** PDF reopened; zero runtime/security errors. |
| `phase11_browser.mjs`, fictional face PPTX | VERIFIED: **1 detected/protected face**, **2 placements**, before/after crops loaded, actual downloaded PPTX reopened, image dimensions preserved; measured blur gradient ratio **0.0414792062**; zero runtime/security errors. |

Coverage remained **NOT ESTABLISHED** and downstream use remained **BLOCKED**, as required by the existing design. Chrome ran headlessly; no human visual sign-off or separate browser/OS matrix is claimed. No real Cadence documents or confidential images were used.

## Reproduce the synthetic verification

With the backend running and dev/model dependencies installed, from a second CMD terminal:

```cmd
call .venv\Scripts\activate.bat
set "CADENCE_CHECK_DIR=.private_review_app\runtime-check-%RANDOM%-%RANDOM%"
python -B tools\verify_windows_runtime.py engines --work-dir "%CADENCE_CHECK_DIR%"
python -B tools\verify_windows_runtime.py api --work-dir "%CADENCE_CHECK_DIR%"
```

Restart the backend, then in that same second terminal:

```cmd
python -B tools\verify_windows_runtime.py reload --work-dir "%CADENCE_CHECK_DIR%"
```

The check directory contains `fixtures.json`, `engines.json`, `api.json`, `restart.json`, `reload.json` and synthetic downloads. Raw installation/test/browser logs from this execution are under `.venv` (`runtime-resolution.log`, `runtime-install.log`, `clean-resolution.json`, `model-transfer.log`, `model-local-install.log`, `pytest-full.log`, `pytest-final.log`, `engines-verification.log`, `api-verification.log`, `reload-verification.log`, `browser-review.log`, `browser-face.log`). Browser crops/screenshots/downloads/profiles remain in private review storage. The test's one saved web audit and original synthetic upload are retained to make restart verification inspectable; no existing project data was deleted.

## Remaining limits and exact startup

There are **no unresolved blockers for the tested configuration**. Full offline startup after package/model installation was not independently tested with networking disabled. Windows 10, other Python versions, a clean second machine, non-Windows platforms, all possible documents, and adversarial accuracy/resource bounds are **NOT VERIFIED**. The original scratch experiments still require their own investigation and private inputs; they are not included in readiness claims.

The application remains a loopback, trusted local single-reviewer tool without user authentication or encryption at rest. It does not claim universally sanitized artifacts: unsupported content/unresolved findings can remain, and `safe_for_llm`, `coverage_verified` and `downstream_export_permitted` remain false. Optional private OCR temp setup is documented but is not an automatic launcher change.

First-time normal installation (after installing Python and putting Poppler on PATH):

```cmd
cd /d "%USERPROFILE%\cadence-pii-guard"
if not exist .venv py -3.13 -m venv .venv
call .venv\Scripts\activate.bat
python -m pip install -r requirements.txt
python -m pip install -r requirements-models.txt
python -m pip check
python run.py
```

Daily startup needs only:

```cmd
cd /d "%USERPROFILE%\cadence-pii-guard"
call .venv\Scripts\activate.bat
python run.py
```

Open **http://127.0.0.1:8765/**. In another CMD window:

```cmd
curl.exe --fail http://127.0.0.1:8765/api/config
```

The final verification leaves this normal bundled application running on port 8765. The optional Vite server was stopped. Save audit revisions before stopping Python with Ctrl+C; retain original uploads for reload. No database/worker service or Node process is needed for normal execution.
