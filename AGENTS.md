AGENTS.md — PAT Snap 550 (Martindale HPAT550 Optical Screen Logger)

1. Project Overview

PAT Snap 550 is a mobile-first, offline-capable web application designed for UK Portable Appliance Testing (PAT) engineers using a Martindale HPAT550 tester. Instead of manually transcribing test results, the engineer photographs the HPAT550 result screen with the phone camera, then checks and logs the extracted readings.

Architecture: Single-file, zero-build-step web app (index.html).

Stack: Semantic HTML5, Tailwind CSS (via CDN), Vanilla ES6+ JavaScript (wrapped in an IIFE), and Tesseract.js v5 (client-side WASM OCR).

Compliance Standard: UK IET Code of Practice for In-Service Inspection and Testing of Electrical Equipment (5th Edition) (DD/MM/YYYY dates, Class I / Class II / IEC Lead sequences, Earth Bond in Ω, Insulation Resistance in MΩ at 250V/500V, and Lead Polarity).

2. File & Code Architecture (index.html)

All markup, styling, and application logic live inside index.html:

CSS & Print Media (<style>):


@media print rules hiding .no-print UI chrome and rendering #printReportContainer as a formal white-background PAT Register Certificate (A4/Letter printable).

UI Views & Modals:

Header: Brand badge, IET CoP 5th Ed tag, and live Next Tag (#headerNextAssetId, e.g., PAT-001).

Tab 1 — Scan Screen (#tabScan):

Camera preview (#cameraVideo, #viewfinderGuide corner frame, #cameraPlaceholder / #cameraStatusText when blocked) plus the reader badge (#aiModeBadge). No Live Read (continuous OCR was removed).

One big TAKE PICTURE button (#btnCaptureFrame): captures from the preview (ImageCapture.takePhoto full-res still where supported, e.g. Android Chrome; otherwise the current video frame). If the preview isn't running (blocked/unsupported) it opens the phone's camera app via #filePhotoInput (capture="environment"). A collapsed "More options" holds photo upload, Demo Scan (#btnDemoScan + #demoPresetSelect) and the batch location card. Swipe-back / Android back on the check screen discards the scan (history.pushState + popstate).

Sticky batch location card (#currentBatchLocationDisplay).

Tab 2 — Results Log (#tabLog):

4-card KPI strip (Tested, Passed, Failed, Pass %).

Actions: Export CSV, Print / PDF Report, Clear All.

Search input and status filter pills (ALL, PASS, FAIL), plus scrollable record cards with thumbnail lightboxes.

Verification & Asset Tagging Modal (#verifyModal):

Side-by-side RAW LCD and OCR VIEW (binary thresholded preview) thumbnails.

Live LCD Contrast bias slider (#inputOcrBias), Invert LCD toggle (#btnToggleInvertLcd), and ↻ Re-Read button (#btnReRunOcr).

Large tap targets for PASS/FAIL, Visual Check, Asset ID (with auto-increment), Appliance (with quick chips), Location, Test Class, Earth Bond (Ω) and Insulation (MΩ) with −.01 / +.01 micro-steppers, Voltage, Polarity, and Notes.

JavaScript Module Structure (inside <script>):

state & dom: Central state object and cached DOM references.

Storage (loadFromStorage, saveToStorage, initPhotoStorage): record text in localStorage under pat_snap_550_records_v1 (sticky room location under pat_snap_550_last_location_v1). Photos live in IndexedDB (database pat_snap_550_photos_v1, store 'photos', record id → data URL) and are loaded into memory on start; old localStorage photos are migrated on first load. navigator.storage.persist() is requested and a warning shows above 80% of the storage estimate. If IndexedDB can't be opened, photos stay in localStorage with the old trimming (all but the newest 5 photos are dropped when full).

Capture (startCamera/stopCamera/syncCamera, takePicture, handleImageFileUpload, processCapturedImage): the camera runs only on the Scan tab, with no check screen open, while the page is visible. processCapturedImage makes a 1600px copy (AI) and a 1100px copy (offline), then reads once. AI reading was verified 6/6 correct on example photos downscaled to 1280px and 1920px (video-frame sizes). PASS/FAIL is detected by colour (detectStatusByColour: tall green/red blob on the cropped screen) and passed to the parser.

Screen facts (from real photos): the HPAT550 has a colour TFT screen, NOT a 7-segment LCD. Dark mixed-case text on white, e.g. "250V", "IEC lead test [interim] result" / "Class I test result", "Earth cont. RPE: 0.02Ω" (or ">20Ω", or "-" = not tested), "Insulation RISO: >100MΩ", "Polarity: Correct", a big green PASS / red FAIL (absent on interim screens), and a limits line "RPE ≤0.1Ω, RISO ≥1MΩ" that must never be read as a reading.

Preprocessing (cropToScreen + preprocessForOcr):

cropToScreen finds the bright white screen in a photo and crops to it.

Upscales smaller crops to ~900px width; slight blur(0.75px).

Grayscale uses the darkest colour channel (not luminance) so green PASS / red FAIL become ink.

Contrast stretch, then Otsu threshold plus thresholdBias (-45 to +45); auto-normalises dark backgrounds to black-on-white.

HPAT550 Parser (parseHpat550OcrText): Label-aware — each value is only taken from its own labelled line. Returns `found` flags; anything not genuinely read is shown to the engineer as "Not read" and left blank (never filled with a plausible default). If neither PASS nor FAIL is found, the test is treated as not complete (`interim`) and the engineer must choose the outcome.

AI reading (readScreenWithOpenAI / mapOpenAiReading): when an OpenAI key is saved (More options; localStorage pat_snap_550_openai_key_v1, model in pat_snap_550_openai_model_v1, default gpt-5-mini with reasoning_effort minimal via reasoningEffortFor(); measured 18/18 correct and ~40% cheaper per photo than gpt-4.1-mini, which is deprecated; gpt-5-nano / gpt-5.4-nano / gpt-4o-mini made reading errors), the 1600px photo is sent to OpenAI Chat Completions with a strict JSON schema and a prompt describing the HPAT550 screen. This is the primary reader (6/6 example photos fully correct). Any failure falls back to the offline Tesseract reader, which uses a separate 1100px copy (canvas.offlineCanvas) because it reads worse at other sizes. Never commit API keys: .env is gitignored.

AI image size (aiImageFor): the AI gets the whole photo scaled to max 1024px (was 1600px) for a faster, cheaper read. Do NOT crop it to findScreenBox: the voltage at the top of the screen sits outside the white area and was cut off (Dan, real tester).

Debug info (More options → "Debug info", pat_snap_550_debug_v1; renderDebugPanel / #debugPanel on the check screen): shows reader used (and why offline was used), model, time, image size and the exact image sent, tokens and approx cost (OPENAI_PRICES), and the raw answer. When on, the AI request uses OPENAI_DEBUG_SCHEMA, which adds a per-field self-rated confidence (high/medium/low) and notes. That confidence is the model's own guess, not a measured value. state.lastDebug is cleared after showing so edits don't show stale details.

Photo pre-check (assessPhoto + photoLooksLikeScreen, in runOcrOnCanvas before any reading): free, on-device. Rejects photos that are too dark (brightness < 30), have no screen-shaped bright area (aspect 0.9–2.2, 4–97% of the frame), no text on the screen (ink < 2%) or are too blurry (sharpness < 50). Shows #photoCheckModal with Retake / Read anyway. Example photos measure brightness 89–108, aspect 1.33–1.51, ink 13–18%, sharpness 890–1400; re-check those ranges if changing the cut-offs. Demo Scan sets canvas.skipPreCheck.

Sanity checks (findSanityIssues / refreshSanityBanner): readings that contradict the result are flagged in a red banner on the check screen and need a "Save anyway?" confirm — PASS with Earth > 0.1 Ω, Insulation < 1 MΩ (2 MΩ Class II) or Polarity Incorrect; FAIL with every reading within limits and Visual PASS. These are usually misreads that a busy engineer would miss.

Screen finding (findScreenBox): thresholds relative to the whitest pixel, falling back to an Otsu threshold when that finds nothing (a glare spot, or the camera exposing for a screen that fills the frame, made the first method miss). The photo-check prompt shows a small measurements line (#photoCheckDetails) to diagnose wrongly rejected photos.

Tested by + tester serial: set once in #testerSetupModal (openTesterSetup), which opens the first time Take Picture or Upload is pressed and from the "Testing as … · change" line (#btnTesterInfo) on the Scan tab. Remembered on the phone (pat_snap_550_tested_by_v1, pat_snap_550_tester_serial_v1) and stamped on each record as testedBy / testerSerial. The check screen only shows them (#testerFieldsBlock) when editing an old result. Shown in the print report header (distinct values), a "Tested by" report column and the CSV.

Label printer (Phomemo M120/M110 over Web Bluetooth; Chrome only): connectPrinter / printRecordLabel / drawLabel. Service 0xff00, write characteristic 0xff02; sequence ESC N 0x0D speed, ESC N 0x04 density, 1F 11 0A (gap labels), GS v 0 raster (48 bytes = 384-dot head), 128-byte chunks, footer 1F F0 05 00 1F F0 03 00 (from phomemo-tools / phomymo). Labels are 320 dots wide (40 mm) and right-aligned on the head (PRINTER.LABEL_X = 64), which was confirmed on Dan's M120; centring cut the left edge. Sizes 40×30 (240 rows) or 40×20 (160 rows), chosen on the Scan tab (pat_snap_550_label_size_v1). The label shows PASS or FAIL (just the word) in a black bar, Asset ID, date, tester name and a QR code (qrcode-generator 1.4.4 from cdnjs) holding the whole result as plain text (labelQrText). Printing happens from "Save & Print" on the check screen (connects first if needed), "Label" in the Results Log (preview + Print) and "Test label". A plain Save never prints. The check screen shows a live label preview. Previews without a printer: window.PAT_LABEL_PREVIEW(record, size).

Saving rules: PASS/FAIL must be chosen if not read; voltage must be chosen (250V/500V) if not read, unless insulation is "Not tested". No reading is ever defaulted.

Editing: each Results Log card has Edit (openRecordForEdit), which reuses the check screen with scanner tiles hidden; saving updates the record in place (state.editingRecordId).

Regression test: example photos live in "example pictures/". Run them through the real app in headless Chrome (Playwright driving the installed Chrome, uploading via #filePhotoInput) and compare the "Scanner read" tiles against what each photo shows.

3. Coding Conventions & Guardrails for Agents

When modifying or extending this codebase, follow these rules:

Maintain Single-File Portability (unless explicitly asked to split):

Keep index.html runnable directly in a browser or Claude Artifact without requiring Node/npm bundlers, unless the user explicitly asks to migrate to Vite/React/PWA.

Preserve DOM ID Integrity:

Every element referenced in const dom = { ... } must exist in the HTML with the exact id. Never remove or rename an HTML id without updating dom and its event listeners.

Protect OCR Accuracy:

Do not remove preprocessForOcr() or OCR_CHAR_WHITELIST when changing OCR calls. The whitelist must include lowercase letters (the screen labels are mixed case). Never let a missed reading fall back to a fake value — leave it blank and flag it.

Always terminate Tesseract workers (await worker.terminate()) after each OCR run to avoid WebAssembly memory leaks on mobile Safari/Chrome.

Mobile Field Ergonomics:

Keep touch targets large (py-3 / py-3.5+), maintain high-contrast dark-mode slate/yellow styling for outdoor/plant-room visibility, and never use native window.alert() or window.confirm() (use showToast() and showConfirmDialog()).

UK PAT Domain Accuracy:

Default date formatting is en-GB (DD/MM/YYYY).

Class II appliances must default Earth Bond Continuity to N/A and Lead Polarity to N/A.

If Visual Check is marked FAIL, overall test status must automatically switch to FAIL.

4. Suggested Next Enhancements

If the user asks what to build next, good high-value additions include:

PWA / Service Worker + IndexedDB: Cache tesseract.js WASM/eng traineddata for 100% offline basement/plant-room use and move high-res LCD photos from localStorage to IndexedDB.

Configurable Pass/Fail Threshold Warnings: Highlight Earth Bond readings above 0.1 Ω + R_lead or Insulation below 1.0 MΩ (Class I) / 2.0 MΩ (Class II) per IET CoP 5th Edition limits.

Retest Due Date Calculation: Auto-calculate 3 / 6 / 12 / 24 / 48 month retest dates based on environment & equipment type.

Client / Site Metadata Header: Store Client Name, Site Address, Engineer Name, and Instrument Serial Number for the PDF/CSV register header.
