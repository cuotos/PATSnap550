AGENTS.md — PAT Snap 550 (Martindale HPAT550 Optical Screen Logger)

1. Project Overview

PAT Snap 550 is a mobile-first, offline-capable web application designed for UK Portable Appliance Testing (PAT) engineers using a Martindale HPAT550 tester. Instead of manually transcribing test results, the engineer aims their phone camera at the HPAT550 LCD display (similar to scanning a QR code) to extract, verify, and log compliance records.

Architecture: Single-file, zero-build-step web app (index.html).

Stack: Semantic HTML5, Tailwind CSS (via CDN), Vanilla ES6+ JavaScript (wrapped in an IIFE), and Tesseract.js v5 (client-side WASM OCR).

Compliance Standard: UK IET Code of Practice for In-Service Inspection and Testing of Electrical Equipment (5th Edition) (DD/MM/YYYY dates, Class I / Class II / IEC Lead sequences, Earth Bond in Ω, Insulation Resistance in MΩ at 250V/500V, and Lead Polarity).

2. File & Code Architecture (index.html)

All markup, styling, and application logic live inside index.html:

CSS & Print Media (<style>):

Custom @keyframes scanline animation for the viewfinder reticle.

@media print rules hiding .no-print UI chrome and rendering #printReportContainer as a formal white-background PAT Register Certificate (A4/Letter printable).

UI Views & Modals:

Header: Brand badge, IET CoP 5th Ed tag, and live Next Tag (#headerNextAssetId, e.g., PAT-001).

Tab 1 — Scan Screen (#tabScan):

Rear-camera <video> stream (#cameraVideo) with a Martindale-proportioned alignment reticle (#reticleBox) and real-time HUD (#hudStatusText, #hudClassText, #hudEarthValue, #hudInsValue).

Primary controls: SNAP & SCAN SCREEN (#btnCaptureFrame), LIVE READ toggle (#btnToggleLiveScan), native photo upload fallback (#filePhotoInput), and a Demo Scan (HPAT550) simulator (#btnDemoScan + #demoPresetSelect).

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

Storage (loadFromStorage, saveToStorage): Persists records in localStorage under pat_snap_550_records_v1 and sticky room location under pat_snap_550_last_location_v1. Includes automatic thumbnail pruning if localStorage quota is exceeded.

Camera & Live OCR Loop (startCameraStream, toggleLiveAutoScan, runLiveScanTick, stopLiveAutoScan): Manages navigator.mediaDevices.getUserMedia (facingMode: 'environment') and runs a non-overlapping ~850 ms Tesseract worker loop that updates the viewfinder HUD in real time until locked.

7-Segment LCD Preprocessing (preprocessForOcr): Critical image pipeline tailored to segmented LCD displays:

Upscales smaller crops to ~900px width.

Applies ctx.filter = 'blur(0.75px)' to bridge narrow gaps between seven-segment bars (preventing digits like 0.02 from splitting into strokes misread as 0.04).

Stretches luminance contrast (minLum–maxLum to 0–255).

Computes Otsu's optimal binary threshold plus thresholdBias (-45 to +45) and auto-normalises dark-mode LCD screens to black-on-white ink.

HPAT550 Parser (parseHpat550OcrText): Line-aware regex parser extracting PASS/FAIL, Class I/Class II/Lead, 250V/500V, Insulation (MΩ), Earth Bond (Ω), and Polarity.

3. Coding Conventions & Guardrails for Agents

When modifying or extending this codebase, follow these rules:

Maintain Single-File Portability (unless explicitly asked to split):

Keep index.html runnable directly in a browser or Claude Artifact without requiring Node/npm bundlers, unless the user explicitly asks to migrate to Vite/React/PWA.

Preserve DOM ID Integrity:

Every element referenced in const dom = { ... } must exist in the HTML with the exact id. Never remove or rename an HTML id without updating dom and its event listeners.

Protect 7-Segment OCR Accuracy:

Do not remove preprocessForOcr() or the character whitelist (ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789.<>:/()-ΩM ) when changing OCR calls; raw camera frames without Otsu thresholding and segment gap-bridging suffer from glare and 2 vs 4 / 5 vs 6 digit confusion.

Always terminate Tesseract workers (await worker.terminate()) when stopping liveScan or finishing single-shot OCR to avoid WebAssembly memory leaks on mobile Safari/Chrome.

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
