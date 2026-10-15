# Windows CMD setup and execution guide

Updated 15 October 2026 for the runtime stabilization of commit `316d3cc96c53e7a17f781ebc9575822429c8d2c8`. The original audit is historical; [WINDOWS_RUNTIME_FIX_REPORT.md](WINDOWS_RUNTIME_FIX_REPORT.md) records the current execution results.

**READY on the verified Windows 11 x64 / Python 3.13.5 configuration:** backend HTTP, actual OCR/PII/face/redaction engines, saved-audit restart/reload, 302 Python tests, 40 React tests, four core tests and both Chrome acceptance workflows passed. Other OS/Python combinations were not independently tested.

All command blocks use Windows Command Prompt (`cmd.exe`). In VS Code select **Terminal > Select Default Profile > Command Prompt** and open a new terminal. Use a local writable checkout under your Windows account.

## 1. Prerequisites

| Requirement | Supported setup |
| --- | --- |
| Windows | x64, Windows 10/11 target. The checks run on the supplied Windows host, not a separate OS matrix. |
| Python | **3.13 x64**; validation interpreter is **3.13.5**. Install the Python launcher and pip. `.python-version` declares the selected minor version; CMD commands explicitly select it. Python 3.14 is not validated. |
| Git | Required to clone. |
| Poppler | Windows binaries on PATH: `pdfinfo.exe` and `pdftoppm.exe`. Version 26.09.0 was available. Needed for scanned/hybrid PDF OCR. |
| Windows PowerShell | `powershell.exe` must exist: Python internally invokes it to enforce private-directory ACLs. Your terminal remains CMD. |
| Model download | The pinned `en_core_web_lg==3.8.0` wheel is about 400.7 MB; allow additional extraction/disk space. Installation requires access to PyPI and the official GitHub release. |
| Browser | Needed for the application UI; Chrome specifically for optional real-browser regression scripts. |
| Node/npm | **Optional for normal application use.** For frontend development use a compatible Node 22 release (22.16.0/npm 11.12.0 tested). Locked Vite requires `^20.19.0 || >=22.12.0`. |
| VS Code | Optional editor. |

Install Python/Node using their official Windows installers. For Poppler, follow the Windows link in [pdf2image installation](https://pdf2image.readthedocs.io/en/latest/installation.html), extract the archive and locate its executable directory.

```cmd
git --version
py -0p
py -3.13 --version
py -3.13 -m pip --version
where powershell
where pdfinfo
where pdftoppm
pdfinfo -v
pdftoppm -v
```

If Poppler is not on PATH, replace this example with the actual extracted directory, then use this same terminal for Python:

```cmd
set "PATH=C:\tools\poppler\Library\bin;%PATH%"
```

No PostgreSQL, Redis, Supabase, Docker, Tesseract, PaddleOCR, GPU/CUDA, Microsoft Office installation, external LLM or cloud credentials are required. Standard Windows wheels are installed by pip; a compiler was not needed in the tested environment.

## 2. Clone and open the project

Replace the genuinely user-specific repository URL:

```cmd
cd /d "%USERPROFILE%"
git clone "YOUR_REPOSITORY_URL" cadence-pii-guard
cd /d "%USERPROFILE%\cadence-pii-guard"
code .
```

`code .` is optional. For an existing checkout elsewhere, use its actual path instead of the default clone directory below.

## 3. Environment configuration

**No `.env` files or application-specific environment variables are required.** Backend host/port and the Vite proxy are defined in source.

Normal storage is `.private_review_app`, created beside `run.py` with restricted Windows ACLs. It contains original `uploads`, generated `artifacts`, and explicitly saved JSON `audits`. Retain original uploads so saved audits can reload. Use the same Windows account after restart. Do not use a network share or a path traversing junctions/symlinks.

`PATH` locates native executables. `TEMP`/`TMP` default to the Windows account's temp directory; OCR renders page images there temporarily. Optional private-temp configuration is described in section 6.

The `CADENCE_UI_URL`, `CADENCE_UI_FIXTURE`, `CADENCE_TEST_CHROME`, `CADENCE_UI_SYNTHETIC`, `CADENCE_UI_COMPLETE`, and `CADENCE_UI_REPRO` variables are browser-test settings, not application configuration. Historical Paddle experiment variables are unrelated to startup.

## 4. Python environment setup

From root, create a project environment only when it does not already exist. Preserve an existing environment; verify its Python version before reusing it.

```cmd
cd /d "%USERPROFILE%\cadence-pii-guard"
if not exist .venv py -3.13 -m venv .venv
call .venv\Scripts\activate.bat
python --version
python -m pip install --dry-run -r requirements.txt
python -m pip install -r requirements.txt
python -m pip install -r requirements-models.txt
python -m pip check
```

Dependency layout:

- `requirements.txt`: application imports and one platform-specific libmagic provider; one OpenCV provider.
- `constraints-windows-py313.txt`: exact resolved runtime/test transitive versions. Constraints do not install unused test packages.
- `requirements-models.txt`: official `en_core_web_lg==3.8.0` release wheel, compatible with spaCy 3.8.
- `requirements-dev.txt`: runtime plus pytest, HTTP test client and synthetic document generation.
- `docs/dependencies/requirements-original.txt`: historical 161-package snapshot, **not an installation input**. It retains the old PaddleX/PyYAML conflict for provenance.

PaddleX/Docling experiments are not in the supported runtime install. No `--no-deps` or force-install workaround is used. Do not mix the historical snapshot into this environment.

Verify imports and local engines:

```cmd
python -c "import fastapi,uvicorn,pypdf,pymupdf,docx,pptx,onnxruntime,cv2,spacy,presidio_analyzer; print('Runtime imports OK')"
python -c "import magic; print(magic.from_file('tests/fixtures/text.pdf', mime=True))"
python -c "from rapidocr_onnxruntime import RapidOCR; RapidOCR(); print('RapidOCR initialized')"
python -c "import spacy; nlp=spacy.load('en_core_web_lg'); print(nlp.meta['version']); from app.engines.pii_detection import PIIDetectionEngine; PIIDetectionEngine(); print('Presidio initialized')"
python -c "from app.engines.visual_detection import FaceDetector; FaceDetector(); print('YuNet initialized')"
```

Expected: clean pip check, MIME `application/pdf`, model version `3.8.0`, and initializer success messages. The YuNet asset is already tracked and hash-checked. RapidOCR includes its local inference assets. Use the pinned model manifest instead of an unpinned spaCy model download.

## 5. Frontend setup

For normal use, **skip this section**: FastAPI serves the committed React bundle. No Node process is required.

For frontend source development, open another CMD window:

```cmd
cd /d "%USERPROFILE%\cadence-pii-guard\frontend"
npm ci
npm run dev
```

Open `http://127.0.0.1:5174/assets/`; leave this terminal open. The API proxy requires the Python backend on 8765. Use `127.0.0.1` exactly: the trusted Vite Origin rewrite does not cover `localhost:5174`. No frontend `.env` is used.

`npm run build` intentionally empties/replaces the committed `app\web\static` bundle. It is unnecessary for daily use or this verification. For routine checks:

```cmd
npm test
npm run lint
npx tsc --noEmit
```

## 6. Backend setup

There is no separate backend directory, engine service or worker. From root:

```cmd
call .venv\Scripts\activate.bat
python run.py
```

The launcher imports `app.main:app`, binds `127.0.0.1:8765`, disables access logs/proxy headers, and limits concurrent connections to four. Successful startup is normally quiet at warning log level. The first document initializes local ML engines and can take longer.

Optional private OCR temp directory, run **before** starting Python and only after dependency setup:

```cmd
python -c "from app.engines.audit.storage import AuditStore; AuditStore('.private_review_app/temp')._prepare()"
```

If that succeeds, use these settings in the same terminal:

```cmd
set "TEMP=%CD%\.private_review_app\temp"
set "TMP=%CD%\.private_review_app\temp"
python run.py
```

This uses the existing ACL helper, without disabling storage protection. Normal startup also works without these optional temp settings.

## 7. Database setup

There is **no database server, migration or seed step**. Sessions are in process memory. **Save audit** explicitly persists JSON snapshots; feedback is not automatically saved. Saved audit reload uses the original fingerprinted upload and regenerates the artifact. No other process must start first.

## 8. Starting the complete application

1. **CMD terminal 1 — backend and bundled frontend.** Working directory: repository root. Keep it open.

   ```cmd
   cd /d "%USERPROFILE%\cadence-pii-guard"
   call .venv\Scripts\activate.bat
   python run.py
   ```

   Expected success: running process, usually no banner. UI: `http://127.0.0.1:8765/`. API configuration: `http://127.0.0.1:8765/api/config`.

2. **CMD terminal 2 — verify and open the browser.** It can close after these commands.

   ```cmd
   curl.exe --fail http://127.0.0.1:8765/api/config
   start "" "http://127.0.0.1:8765/"
   ```

   Expected JSON: `{"max_upload_bytes":31457280,"supported_formats":["pdf","docx","pptx"],"trusted_local_only":true}`.

3. **Optional CMD terminal 3 — Vite** from section 5, only when editing frontend source. Keep it open while developing.

No database, engine, queue, or worker terminal is required. For normal execution the only long-running command is `python run.py`.

## 9. Verification checklist

```cmd
curl.exe --fail http://127.0.0.1:8765/api/config
curl.exe --fail http://127.0.0.1:8765/api/documents
curl.exe --fail http://127.0.0.1:8765/api/saved-audits
netstat -ano -p tcp | findstr :8765
```

There is no `/health`, `/docs`, or `/openapi.json` endpoint. `/api/config` is the practical liveness/configuration check.

For a minimal browser test, create a uniquely named fictional DOCX:

```cmd
python -c "from pathlib import Path; from tempfile import gettempdir; from uuid import uuid4; from docx import Document; p=Path(gettempdir())/('cadence-synthetic-'+uuid4().hex+'.docx'); d=Document(); d.add_paragraph('Contact: ada@example.com. Employee ID: EMP-00001.'); d.save(p); print(p)"
```

Upload the printed file through the UI. Inspect findings and source context, reject then reconfirm the employee identifier, check revision/history, download and reopen the review copy, and **Save audit**. Restart Python and reload from the saved-audit listing. Retain its original upload. Review copies can retain unresolved/unsupported PII; coverage and downstream-export clearance intentionally remain false.

For reproducible real-engine/API checks (only synthetic documents), install the test tools and choose a fresh check directory in a second terminal:

```cmd
python -m pip install -r requirements-dev.txt
set "CADENCE_CHECK_DIR=.private_review_app\runtime-check-%RANDOM%-%RANDOM%"
python -B tools\verify_windows_runtime.py engines --work-dir "%CADENCE_CHECK_DIR%"
python -B tools\verify_windows_runtime.py api --work-dir "%CADENCE_CHECK_DIR%"
```

Restart `python run.py` in terminal 1, then in the same second terminal:

```cmd
python -B tools\verify_windows_runtime.py reload --work-dir "%CADENCE_CHECK_DIR%"
```

These commands test native/scanned/hybrid PDFs, DOCX/PPTX, default Presidio, employee IDs, face detection/blur, artifact reopening, real HTTP uploads/evidence/feedback/save, restart/reload, and rejection of a deliberately changed **synthetic test upload**, which is restored in a `finally` block. Evidence remains in the dedicated private check directory; existing user documents are not opened or changed.

Regression commands:

```cmd
set "PYTHONDONTWRITEBYTECODE=1"
python -B -m pytest tests -q -p no:cacheprovider
node --test tests\frontend\core.test.mjs
cd frontend
npm test
npm run lint
npx tsc --noEmit
```

The bytecode setting also reaches subprocesses and avoids modifying tracked legacy `.pyc` files. Some regression tests use injected analyzers, so use the real-engine/HTTP checks as well.

## 10. Common Windows errors

| Symptom | Action |
| --- | --- |
| Old PaddleX/PyYAML `ResolutionImpossible` | Use the current root requirements, not `docs/dependencies/requirements-original.txt`. Current runtime has no PaddleX requirement. Never bypass dependency resolution. |
| Missing `magic` | Confirm `where python` and `python -m pip --version` point to the active venv, then install current requirements. Windows selects only `python-magic-bin`, bundling libmagic. |
| DLL/architecture error | Use Python x64 3.13 and the constrained Windows wheels; do not mix multiple `magic` or OpenCV providers in an old environment. A fresh separately named venv is safer than deleting an existing one. |
| spaCy `[E050]` / missing English model | Run `python -m pip install -r requirements-models.txt`. A small differently named model is not the configured equivalent. |
| Model download timeout/network failure | Retry the pinned model command when GitHub access is available. Do not claim detection works until the installed model loads. |
| `PDFInfoNotInstalledError` / missing Poppler | Check both `where pdfinfo` and `where pdftoppm`; put the actual binary directory on this terminal's PATH. No `POPPLER_PATH` variable is read by the app. |
| Face model unavailable/untrusted | Restore the tracked YuNet asset from a trusted checkout; required SHA-256 is `8f2383e4dd3cfbb4553ea8718107fc0423210dc964f9f4280604804ed2552fa4`. |
| OpenCV 5 target warning | A warning about the new graph engine can appear; verify actual detection rather than treating the message alone as failure. Do not install a second OpenCV package to suppress it. |
| Private-directory setup failure | `where powershell` must find Windows PowerShell. Use local writable storage under the current user, avoiding symlinks, junctions and network shares. Do not disable ACL checks. |
| Port occupied / WinError 10048 | `netstat -ano -p tcp | findstr :8765` (or `:5174`); inspect `tasklist /fi "PID eq PID_NUMBER"`. Stop only a recognized process through its terminal. Vite uses a fixed proxy and strict port. |
| HTTP 403 | Use same-origin `127.0.0.1` URLs. Curl mutations require `-H "X-Cadence-Local: 1"`; the UI adds it. |
| HTTP 413 / 415 | Upload cap is 30 MiB. Web formats are PDF, DOCX and PPTX, validated by contents. |
| HTTP 409 | Refresh stale revisions; retain the original fingerprinted upload for reload/regeneration. |
| Vite HTML works but uploads disabled | Verify backend `/api/config`; Vite alone cannot process documents. |
| Processing HTTP 200 with a failed stage | Inspect `summary.failure` and stage states. HTTP success alone does not prove extraction/redaction succeeded. |
| Historical scratch script fails | It may require absent private files, old absolute paths or unsupported experimental packages. It is not an application launcher. |

## 11. Stopping and restarting

Save wanted audit revisions and allow active processing to finish. Press **Ctrl+C** in Vite if used, then **Ctrl+C** in the Python terminal. If CMD asks to terminate a batch job, answer `Y` for that terminal. Use `deactivate` to leave the venv.

Restart using section 8. Unsaved in-memory revisions are lost; explicit snapshots persist. Keep original uploads and the same checkout paths. There is no automatic retention worker or database service to stop. The application is a trusted local single-reviewer tool with plaintext private storage, not an authenticated network deployment.

## 12. Quick start

After first-time installation, only one long-running CMD terminal is needed:

```cmd
cd /d "%USERPROFILE%\cadence-pii-guard"
call .venv\Scripts\activate.bat
python run.py
```

In another CMD window:

```cmd
curl.exe --fail http://127.0.0.1:8765/api/config && start "" "http://127.0.0.1:8765/"
```
