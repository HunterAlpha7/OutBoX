# HelloMed: Core Algorithmic & Pipeline Inventory

This document serves as a technical reference for the core algorithms, data pipelines, and architectural patterns implemented within the HelloMed platform.

---

## 1. pg_trgm Trigram Fuzzy Search
- **Location:** Medicine Search API (`src/app/api/search/medicines/route.ts`), OCR Pipeline Step 2 (`Offload/index.ts`, `Offload/route.ts`)
- **Purpose:** Tolerates typos and partial matches when searching medicines by brand name or generic name. Uses PostgreSQL's `pg_trgm` extension which breaks strings into 3-character grams and computes a similarity score (0–1). 
- **Mechanism:** The `%` operator filters candidates above a default threshold, and `similarity()` ranks them. A GIN index (`gin_trgm_ops`) on `medicines.brand_name` and `generics.name` accelerates this.
- **Thresholds:** ≥ 0.4 in OCR pipeline, ≥ 0.3 (default) in search API. Results ranked by `GREATEST(similarity(brand_name), similarity(generic_name)) DESC`.

## 2. Multi-Model LLM OCR Pipeline (Vision → Judge)
- **Location:** `Offload/index.ts` (Express server on Render), `Offload/route.ts` (Next.js API mirror)
- **Two-stage AI pipeline:**
    - **Stage 1 — "Sniper" Vision Model (`qwen/qwen2.5-vl-72b-instruct`):** Multimodal VLM that reads handwritten Bengali+English prescription images and extracts structured `{medicineName, dosage}` JSON. Includes domain-specific prompt engineering for Bengali digit confusion (৪ vs 8).
    - **Stage 3 — "LLM Judge" (`deepseek/deepseek-chat`):** Takes the raw OCR output + fuzzy DB matches and deduces the correct intended medicine by cross-referencing both. Also corrects dosage misreads (Bengali ৪→8 correction heuristic). Acts as a disambiguation and error-correction layer.

## 3. Image Pre-Processing (Binarization + Normalization)
- **Location:** `Offload/index.ts` Step 0 (server-side via Sharp), `src/components/med-scanner/OCRScanner.tsx` (client-side via Canvas)
- **Server-side (Sharp):** `greyscale()` → `normalize()` — histogram-stretches pixel intensities to full 0–255 range for better contrast on faded handwriting.
- **Client-side (Canvas):** Per-pixel grayscale averaging `(R+G+B)/3` → thresholding at 150 (above = white 255, below = darkened by -50). A simple binarization to separate ink from paper before sending to the API.
- **Optimization:** Uses `browser-image-compression` to downscale to max 1500px / 1MB before upload.

## 4. Tiered Database Lookup with Staggered Racing
- **Location:** OCR Pipeline Step 2 (`Offload/index.ts` lines 127–284)
- **Algorithm:** For each OCR-extracted medicine name, two async search promises race in parallel:
    1. **Primary DB** (`medicines` + `generics` tables) — starts immediately.
    2. **Secondary DB** (`external_medicines` table) — starts after a 150ms delay.
- **Optimization:** If Primary hits (score ≥ 0.4), it sets a `primarySuccess` flag that causes Secondary to abort early. If both miss, Secondary falls through to the Medex web scraper as a final fallback. Results from scraping are cached into `external_medicines` for future queries.

## 5. HTML Web Scraping (Cheerio DOM Parsing)
- **Location:** `src/lib/medex-scraper.ts`, `Offload/lib/medex-scraper.ts`
- **Purpose:** When a medicine isn't in either local database, scrapes a closed-source pharmaceutical API (medex.com.bd):
    - **Search:** Hits an AJAX endpoint, parses returned HTML with Cheerio, extracts first `<a>` link.
    - **Detail Extraction:** Fetches the brand page, parses `<title>` (pipe-delimited) for strength/dosage form, uses CSS selectors + `nextUntil()` DOM traversal for 12+ medical detail sections.
    - **Alternatives Scraping:** Parses `tr.brand-row` elements with `data-*` attributes, uses regex `৳\s*([\d.]+)` to extract Taka prices.

## 6. Cheap Alternative Finder (Generic-Based Price Comparison)
- **Location:** `src/actions/patient-dashboard.ts` → `findCheapAlternative()`
- **Algorithm:**
    1. Find the queried medicine → get its `genericId`.
    2. Query all medicines sharing the same `genericId`.
    3. **Strength normalization:** `toLowerCase().replace(/\s/g, '')` to match "500 mg" with "500mg".
    4. **Price parsing:** Regex `[\d.]+` extraction from strings like "৳ 5.00", stripping commas.
    5. Filter to same-strength, lower-price alternatives → sort ascending by parsed price.
- **Fallback:** If only found in `external_medicines`, triggers on-demand scraping of generic alternatives from Medex, deduplicates by brand name (case-insensitive), inserts new ones, then re-queries.

## 7. IP-Based Geolocation (Anonymized Tracking)
- **Location:** OCR Pipeline Step 4 (`Offload/index.ts` lines 362–405)
- **Algorithm:** Extracts client IP from `x-forwarded-for` / `x-real-ip` headers → calls `ip-api.com/json/{ip}` for reverse geolocation → stores approximate lat/lng, city, region, country into `scan_records`. 
- **Fallback:** Uses a hardcoded Dhaka IP (`103.112.54.1`) for localhost. Insertion is fire-and-forget (non-blocking).

## 8. Coordinate Jitter (Anti-Overlap)
- **Location:** `src/components/pharma/MapVisualization.tsx` line 63–64
- **Algorithm:** `lat + (Math.random() - 0.5) * 0.05` — adds ±0.025° random noise to each scan marker's coordinates to prevent exact overlaps on the Leaflet map when multiple scans resolve to the same city-level lat/lng.

## 9. Role-Based Access Control (RBAC) via Middleware
- **Location:** `src/middleware.ts`
- **Algorithm:** Session-check → role-based redirect matrix:
    - `/dashboard` → admins redirected to `/admin`, pharma_reps to `/pharma/portal`.
    - `/admin` → users redirected to `/dashboard`, pharma_reps to `/pharma/portal`.
    - `/pharma` → users redirected to `/dashboard`, admins allowed through.

## 10. Invitation-Based Role Assignment (Database Hook)
- **Location:** `src/lib/auth.ts` → `databaseHooks.user.create.before`
- **Algorithm:** On user creation, checks the `invitations` table for a matching pending, non-expired invite. If found, overrides the default "user" role with the invite's role (e.g., "pharma_rep") and marks the invite as accepted.

## 11. OTP Generation + bcrypt Password Hashing
- **Location:** `src/lib/auth-actions.ts`
- **OTP:** `otp-generator` — 6-digit numeric-only codes, stored in `verifications` table with 10-minute TTL.
- **Password Hashing:** `bcrypt.hash(password, 10)` — 10 salt rounds for password reset flow.
- **Security Pattern:** Silent success on non-existent emails to prevent user enumeration.

## 12. Batch CSV Seeding with Chunked Inserts
- **Location:** `src/db/seed.ts`
- **Algorithm:** Streams CSV files via `csv-parse`, builds in-memory lookup Maps (manufacturer, drugClass, etc.) for foreign key resolution, then batch-inserts in chunks of 500–1000 rows with `onConflictDoNothing()` for idempotency.

## 13. Regex-Based Price Extraction
- **Location:** `src/fix_prices.ts`
- **Algorithm:** For medicines with null prices but non-null `packageContainer`, applies regex `/৳\s*([\d,.]+)/` to extract the Taka price from free-text packaging strings. Processes in batches of 50 with `Promise.all` concurrency.

## 14. Dosage Form Prefix Stripping
- **Location:** OCR Pipeline Step 2 (`Offload/index.ts` line 131)
- **Algorithm:** Regex `/^(tab|cap|syr|syp|inj|drop|drops|susp|lotion|cream|ointment|supp|sol|soln)s?\b[.\-\s]+/i` strips common pharmaceutical prefixes (Tab., Cap., Syr., etc.) from OCR-extracted names before database search to improve match rates.

## 15. Debounced Search (Client-Side)
- **Location:** `src/components/patient/AlternativeFinder.tsx`
- **Algorithm:** 500ms `setTimeout` debounce on keystroke input — clears previous timer on each new keystroke, only fires the server action after 500ms of inactivity. Prevents excessive API calls during rapid typing.
