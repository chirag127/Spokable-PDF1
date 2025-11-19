You are being instructed to build a complete, maintainable, modern, production-ready website called
"Convert to Readable Spokable PDF". The website's job is to take an input PDF (already containing selectable text — NO OCR required),
transform the PDF text and multimodal content (figures, tables, code blocks, formulas) into a **spoken-friendly** readable form,
and produce a new PDF where the text is converted into high-quality, human-friendly, TTS-optimized output.

This prompt contains the full functional specification, UI/UX requirements, technical constraints, processing flow,
error handling, storage, and deployable output format (single-page deliverable + file structure guidance). Do not remove any instruction
below — every requirement must be considered and implemented.

--- PROJECT SUMMARY / GOALS ---
- Provide a web app (HTML/CSS/JavaScript only — no server-side libraries required for deployment) that:
  - Accepts a PDF upload (drag & drop or file picker).
  - Extracts selectable text and multimodal parts (images/tables/figures embedded in PDF pages).
  - Converts content into a **Readable Spokable** format optimized for text-to-speech consumption (e.g., Moon+ Reader Pro or other TTS).
  - Produces a new downloadable PDF containing the converted readable text (with optional table-of-contents, page numbers, font/spacing options).
  - Allows partial-downloads (download partial output while conversion is ongoing).
  - Provides advanced settings for prompt templates, chunking, retries, model selection, and performance (Turbo parallel mode).
  - Stores API keys and files locally in the browser (LocalStorage + IndexedDB) and autosaves settings.
  - Uses the Gemini REST API (Google AI Studio / Generative Language REST endpoints) — **only the REST API** (no external Gemini client libraries).
  - Includes About, Contact, FAQ, Privacy Policy, Terms, Pricing pages — maintainable multi-page structure (multiple HTML/CSS/JS files recommended).
  - Is clearly maintainable: split JS/CSS into modules/files, separate pages, and produce a final `index.html` that imports other files.

--- USER INTERACTION / UX ---
- Homepage UI:
  - Large drag/drop or click-to-upload box that shows file info (name, size, pages) immediately on selection.
  - “Process” button starts conversion; file processing DOES NOT start before user clicks Process.
  - Visual progress indicator that shows stages and percent: e.g., "Extracting text", "Preparing batches", "Calling Gemini API — Batch X of Y", "Generating PDF", "Saving", etc. The progress bar must update live.
  - Option to Download Partial PDF anytime while processing (produced from completed chunks).
  - Option to Download All outputs (e.g., zipped if multiple).
  - Show per-batch logs/errors (human friendly).
- Settings:
  - Save primary Gemini API Key and optional Backup API Key (failover). Keys stored in LocalStorage encrypted or obfuscated and auto-saved (no explicit “Save” needed).
  - Input field linking to "Get your free API key at Google AI Studio" with a short help text + link explaining how to obtain an API key.
  - Model selector (drop-down) with models available late 2025 (e.g., gemini-3-pro-preview, gemini-2.5-pro, gemini-2.5-flash, gemini-2.5-flash-lite, gemini-2.5-pro-preview-tts, gemini-2.5-flash-preview-tts, gemini-2.0-flash, etc.). Default: gemini-2.5-flash or gemini-flash-latest alias.
  - API Timeout, Batch Size (tokens), Overlap Size, Max Retries, Retry Delay, Parallel Chunks (Turbo Mode), Rate Limit Delay, Max Output Tokens, Temperature, Top P, Top K, Presence/Frequency penalties.
  - Transformation Rules UI: toggle convert tables → narrative, code → description, expand math notation, expand acronyms, preserve English unchanged, remove inline citations, phonetic hints for difficult words, add pauses for TTS, etc.
  - Output PDF options: font size, line height, margins, font family, page size, add page numbers, generate TOC.
- Advanced options:
  - “Turbo Mode” (parallel chunk processing) with configurable parallelism and rate-limit delay
  - Auto-retry failed requests and fallback to backup API key
  - Option to try different models in sequence upon failures (rotate through selected model list)
  - Option to stream results when using streaming endpoints (if supported by model) and reflect streaming progress
- Accessibility:
  - Keyboard accessible controls, ARIA labels, readable contrast, and responsive layout.

--- TECHNICAL / ARCHITECTURE REQUIREMENTS ---
- Tech stack: pure HTML, CSS (modern, responsive), and vanilla JavaScript (ES modules). The project should be modular (many JS and CSS files), easy to maintain and deploy on static hosting.
- Storage:
  - Use IndexedDB to store uploaded PDFs, intermediate chunk results, and generated partial outputs so user can resume or download partial PDFs offline.
  - Use LocalStorage for settings and API keys. Keys must be optionally masked in UI and persisted automatically.
- PDF handling:
  - Use a client-side PDF parsing library (pure JS) to extract text and embedded images (for figures/tables). This is allowed client-side for text extraction only; OCR is not required.
  - Extracted text must preserve page boundaries for mapping back to the output PDF.
  - Extract tables and figures as image blobs or structured data (if the PDF contains selectable tabular text).
- Chunking & batching:
  - The app must compute token estimates (approx. tokens per character) and split long documents into batches respecting user-configured Batch Size and Overlap Size.
  - Each batch must include a context overlap to preserve continuity.
  - Implement batching logic to send multiple requests to Gemini REST API sequentially or in parallel (if Turbo Mode enabled).
- Gemini REST API integration:
  - Use **only the REST endpoints** from Google AI Studio / Generative Language API.
  - Authenticate requests with an API key: either in header `x-goog-api-key: YOUR_KEY` or appended as `?key=YOUR_KEY` as fallback. Prefer `x-goog-api-key` in the header.
  - Use `generateContent` or `streamGenerateContent` endpoints depending on desired behavior and model support. For file + text multimodal inputs, call the multimodal endpoint and send `file_data` parts for images (base64) or `file_uri` where supported.
  - Use JSON body `contents` with `parts` arrays; include `system_instruction` when available to set conversion role-based behavior (e.g., "You are an expert at converting technical documentation to natural spoken language...").
  - Example (non-exhaustive) curl pattern (for reference and to be implemented in JS `fetch` calls):
    - curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent" \
        -H "x-goog-api-key: $GEMINI_API_KEY" \
        -H "Content-Type: application/json" \
        -X POST \
        -d '{ "contents": [{"parts":[{"text":"<text or prompt here>"}]}] }'
  - Support `streamGenerateContent?alt=sse` for streaming responses when available and update the UI progressively.
  - Be resilient: if one model fails with a 4xx/5xx or times out, automatically retry (up to Max Retries) and then try the user-selected fallback/backup model(s) or the backup API key. If all models fail, show actionable error message and logs.
  - Respect rate limiting — implement exponential backoff + configurable Rate Limit Delay.
  - If the API can accept images inline (base64) for multimodal reasoning, send figures/tables images with the text prompt asking the model to "Describe this figure for TTS" or "Convert table data into narrative sentences".
- Multimodal support:
  - Use Gemini's multimodal capability to send both text and images when available (images for figures that need description). If the PDF contains vector graphics or tables as images, attach them in the request or send a prompt that references the image.
  - If direct image upload to the API is not feasible from the browser, send a clear fallback: upload image to an accessible temporary URL if the environment allows (note: because this site is static-only, prefer inline base64 where permitted).
- Failover & model rotation:
  - Implement a model-rotation strategy: try primary model (user-chosen), upon failure try next model in list, continue until exhausted. Respect Max Retries and rate-limit delays between attempts.
- Output assembly:
  - Assemble returned converted text chunks in order, deduplicate overlap segments, and produce a single coherent text representation.
  - Generate a new PDF client-side (using a JS PDF generator) with the chosen font, margins, optional TOC, and page numbering.
  - Provide partial PDF generation after each processed batch and allow user to download partial outputs anytime.
- Progress UI:
  - Visual stage indicator and overall % completion displayed and updated as each chunk finishes and as file-level processing stages advance.
  - Show ETA calculation based on processed chunks vs total and API latency (best effort only).
- Logging & Error handling:
  - Show readable errors and internal logs (with option to expand raw API responses).
  - Retry logic should log attempts and backoff delays.
  - If rate-limited, show suggestion: "Enable Turbo Mode? or increase Rate Limit Delay" etc.
- Security & privacy:
  - The project must explain clearly in Privacy page that: API keys are stored only in user browser LocalStorage, PDFs stay in browser IndexedDB and are not uploaded to third-party servers except the Gemini REST calls that the user initiates with their key.
  - Provide guidance on how to get an API key from Google AI Studio (link and short steps).
- Deliverable:
  - The deliverable will be a maintainable multi-file static project (many html/js/css) and a single `index.html` entry that hooks the app together for deployment. Provide clear file structure and instructions to deploy (e.g., host on GitHub Pages or Netlify).
  - Include detailed README with usage, config instructions, and troubleshooting.
  - Provide sample test PDF(s) used for QA containing code examples, tables, formulas, and images.
  - The website should auto-save all settings (no manual save button). Show saved/stale indicator for settings.
  - Include a "Reset Storage" option that cleans LocalStorage and IndexedDB.
  - Provide unit/integration test guidance or minimal test harness pages (e.g., test chunking, test model rotation, test PDF regeneration).
  - Provide a simple, user-facing privacy policy and terms page.

--- PROMPT TEMPLATES & TRANSFORMATION RULES ---
- Provide a set of user-editable prompt templates in Settings:
  - System prompt: "You are an expert at converting technical documentation into natural, spoken language optimized for text-to-speech systems. Convert technical content into natural, spoken format, preserving meaning and improving listenability. Expand abbreviations on first use. Convert tables to narrative sentences. Describe figures and images in detail. Describe code by summarizing what it does rather than reading code verbatim. Convert mathematical notation into spoken words (e.g., x² -> x squared). Remove inline citations. Insert natural pauses and commas for TTS rhythm.)"
  - Table conversion prompt: "Convert the table into flowing narrative sentences describing key rows/columns and trends."
  - Code description prompt: "Describe what the code does in natural language; explain intent and key steps while skipping exact line-by-line reading."
  - Figure description prompt, math prompt, list formatting prompt, etc.
- Transformation rule toggles (user configurable): preserve english text, expand acronyms, convert tables, convert code, expand math, phonetic hints, etc.

--- BATCHING, TOKEN LIMITS, & RETRY STRATEGY ---
- Implement token-estimation and chunking logic. The app must:
  - Estimate token usage from content size and keep batches under user-configured Batch Size (default: 10000 tokens).
  - If the Gemini model has a per-request token limit, split the content into batches and process sequentially (or in parallel in Turbo Mode) including an overlap window (default 200 tokens).
  - For each batch:
    - Send to Gemini REST API.
    - If response truncated due to length, request follow-up with explicit instruction: "Continue from where you left off."
    - Auto-retry failed calls up to Max Retries with exponential backoff (use Retry Delay and Max Retries settings).
    - If a model returns a rate-limit or quota error, attempt failover to the Backup API Key and/or next model in the model list.
  - Maintain ordering and merge batches into final output with deduplication of overlap content.

--- MULTIMODEL & LATE-2025 MODEL AWARENESS ---
- The site must allow the user to choose among commonly available Gemini models in late 2025 (examples to present in the UI for selection):
  - gemini-3-pro-preview
  - gemini-2.5-pro
  - gemini-2.5-flash
  - gemini-2.5-flash-lite
  - gemini-2.5-pro-preview-tts
  - gemini-2.5-flash-preview-tts
  - gemini-2.0-flash
  - (include aliases like gemini-flash-latest)
- The agent should implement logic to prefer small/fast flash models for low-latency transforms and larger pro models for higher-fidelity conversions when configured by the user.

--- IMPLEMENTATION NOTES (for the developer agent) ---
- UI design: modern, minimal, readable typography, soft shadows, rounded cards, a simple color palette — create a professional look with CSS modules or separate CSS files. Ensure mobile responsiveness.
- Code modularity: break JS into modules: ui.js, pdf-extract.js, chunker.js, gemini-api.js, progress.js, storage.js, pdf-generator.js, settings.js, utils.js.
- Comments & documentation: each module should contain clear comments and a header explaining purpose and exported functions.
- Error handling: surface helpful messages and allow user to copy/export raw logs for support.
- Testing: include a page or script that simulates large inputs, slow network, and rate-limited responses for QA.
- Final output: produce a zip of the project and a single `index.html` that imports/bootstraps all modules for quick deployment.

--- UI MICRO-INTERACTIONS & FINAL TOUCHES ---
- Settings auto-save indicator ("Settings saved at HH:MM:SS" or "Unsaved changes"), but the site auto-saves so usually 'saved'.
- Option to enable "Phonetic Hints" for difficult words (e.g., "NIST (en-eye-ess-tee)").
- Provide a built-in short interactive tutorial/walkthrough on first run that explains how to get API key and start converting.
- Provide sample default prompt templates that the user can customize.

--- DELIVERY FORMAT & ADDITIONAL REQUIREMENTS ---
- Deliver a maintainable multi-file static project with:
  - index.html (entry)
  - about.html, contact.html, faq.html, privacy.html, terms.html, pricing.html
  - css/ (multiple css files)
  - js/ (modular JS files listed above)
  - assets/ (icons, sample PDFs)
  - tests/ (QA test files)
  - README.md (deployment instructions + how to get Google AI Studio API key)
- Provide minimal script for local testing (e.g., `npx http-server` instructions) in README.
- Provide a short usage guide for integrating with TTS apps (e.g., how to take the generated PDF's text and feed to Moon+ Reader Pro or other TTS apps).
- Ensure the README and in-app help contain links and instructions on how to:
  - create an API key in Google AI Studio,
  - which Gemini model to choose for speed vs fidelity,
  - expected per-request limits & how batching works,
  - privacy implications.

--- EXTRAS (HIGH PRIORITY) ---
- Always prefer `x-goog-api-key` header for calls; fall back to `?key=` query param if user environment requires it.
- Use streaming `streamGenerateContent` when available for smoother progress; otherwise use `generateContent`.
- When the API accepts images inline, include the figure images inside `parts.file_data.inline_data` (base64 + mime_type) to let the model describe them.
- If an image or table is too complex, produce both (a) a short description for TTS and (b) a detailed alt-text block appended to the PDF for accessibility.
- Provide an “Export prompt profile” option so the user can save and share prompt templates and transformation rules.

--- MANDATORY DOCUMENTATION ADDED TO THE REPO ---
1. README.md with steps:
   - How to set up and deploy.
   - How to get a Google AI Studio API key (link + short steps).
   - How to configure models and what each model is good for (speed vs fidelity).
   - Troubleshooting (common errors and how to fix).
2. LICENSE file (suggest MIT).
3. CHANGELOG.md with recommended release notes structure.

--- FINAL NOTES TO THE AGENT ---
- Do not rely on server-side code. The site must run entirely from the browser and use the Gemini REST API via the provided keys.
- Respect the user's privacy: never store API keys outside the user's browser. Do not send files to any third-party servers beyond the Gemini REST endpoints.
- Provide clear, user-friendly error messages and recovery options.
- Provide sample PDFs used to validate: one technical article with code blocks, one paper with tables, one book-like layout with figures.
- Produce a final deliverable zip with the full project and a short demo video / GIF (optional) showing the Convert -> Partial download -> Final PDF flow.

Implement the full website now following this spec. After implementation, provide:
- a file tree of the delivered project,
- the key source files (index.html + at least the main JS modules),
- a short testing checklist and example test PDF used to validate the behavior,
- and final instructions for deploying on a static host (GitHub Pages, Netlify, etc).
- If you have any questions, feel free to ask it.
- You have to first make the svg files and then convert them to the png files for the icons.
End of specification.
