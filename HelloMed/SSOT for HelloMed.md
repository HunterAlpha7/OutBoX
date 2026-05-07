# Project HelloMed: Comprehensive AI-Powered Medical Companion Documentation

## 1. Project Vision & Overview

**HelloMed** is a high-utility, advanced healthcare application specifically tailored for the Bangladeshi landscape. Its primary goal is to bridge the gap between complex medical documentation and patient understanding, while also providing valuable, anonymized data insights for pharmaceutical companies. The application is built to handle the pragmatic challenges of the local healthcare sector, including illegible handwritten prescriptions, price sensitivity, and varying levels of health literacy among patients.

The application is meticulously designed to handle the pragmatic challenges of the local healthcare sector, including:

* **Illegible Handwriting:** Deciphering complex, cursive, and often multilingual (Bangla/English) handwritten prescriptions.
* **Price Sensitivity:** Providing cost-saving generic alternatives to prescribed medications, empowering patients with instant price comparisons.
* **Health Literacy:** Offering simplified Bangla explanations for complex English medical reports.
* **Public Health Intelligence:** Aggregating anonymized medical data for public health and pharmaceutical analytics.

### Key UX Principles

* **Minimalistic Patient UI:** Focus on ease of use, high contrast, and accessibility to accommodate all user demographics.

* **Utilitarian Pharma Dashboard:** Data-heavy, clean, and export-focused interface for researchers and pharmaceutical partners.
* **Industry Standard UX:** Implementation of Skeleton Screens (no blank loading states) and seamless transitions for a fluid user experience.
* **Bangladeshi Context Integration:**
  * **Price Transparency:** Instant display of medicine prices and cheaper generic alternatives.
  * **Language Accessibility:** Simplified Bangla summaries for complex English texts.
  * **Audio Aid:** Floating "Accessibility Bubble" for Voice/TTS support.

---

## 2. Technical Stack

The project employs a modern, serverless technology stack designed for performance, modularity, and cost-efficiency.

| Layer | Technology | Reason / Implementation Details |
| :--- | :--- | :--- |
| **Framework** | Next.js 16 (App Router) | Provides a seamless, "reloadless" SSR (Server-Side Rendering) and CSR (Client-Side Rendering) experience. |
| **Language** | TypeScript | Ensures industry-grade type safety and modularity across the codebase. |
| **Authentication** | BetterAuth | Comprehensive session management, credential, and OAuth integration. |
| **Database** | PostgreSQL (via Neon/Supabase) | Serverless, efficient, relational data storage optimized for complex queries and Trigram fuzzy search. |
| **ORM** | Drizzle ORM | Lightweight, high-performance SQL-like control for type-safe database interactions. |
| **Storage** | UploadThing & Backblaze B2 | Best-in-class Next.js integration for secure file, PDF, and image uploads (The Vault). |
| **AI Provider** | OpenRouter SDK | Model-agnostic gateway for accessing DeepSeek, Qwen-2.5-VL, GPT-4o mini, and Whisper models. |
| **UI/Styling** | Shadcn/UI + Tailwind CSS | Professional components with a clean, modular aesthetic and utility-first styling. |

---

## 3. System Architecture & Modularity

### 3.1. Serverless Modular Monolith

The application utilizes **Next.js Server Actions** as its core backend mechanism. This approach eliminates the need for a separate, persistent Node.js server, relying instead on serverless functions that consume resources only during active requests. This optimizes both performance and hosting costs.

### 3.2. Separation of Concerns

HelloMed strictly follows a feature-based modular architecture to ensure high maintainability and code reusability:

1. **Routing Layer (UI):** Clean React Server and Client Components (`app/`). The UI never interacts directly with the database.
2. **UI Components Layer:** Reusable Shadcn primitives and feature-grouped components (`components/ui/`, `components/shared/`, `components/med-scanner/`).
3. **Backend Layer (Server Actions & Services):** Server actions act as entry points for the UI. They delegate complex business logic, database queries, and external AI API calls to dedicated services.
4. **Database Layer:** Drizzle schemas and direct PostgreSQL connections (`src/db/`).

**Finalized File Structure:**

```bash
/src
 ├── /app                   # 1. ROUTING LAYER (UI Only - Clean & Minimal)
 │    ├── (app)/            # Protected Pages (Dashboard, Scan)
 │    │    ├── dashboard/   # page.tsx imports <DashboardView />
 │    │    ├── scan/        # page.tsx imports <ScannerView />
 │    │    └── layout.tsx   # Sidebar & Auth Wrapper
 │    └── loading.tsx       # Global Skeleton Loaders for UX
 │
 ├── /components            # 2. UI LAYER (Feature-Grouped)
 │    ├── /ui/              # Shadcn primitives (Buttons, Cards, Inputs)
 │    ├── /shared/          # Global Nav, Footer, Accessibility Bubble
 │    ├── /med-scanner/     # Scanner features (scanner-view.tsx, camera-frame.tsx)
 │    └── /reports/         # Report features (chat-interface.tsx, upload-dropzone.tsx)
 │
 ├── /server                # 3. BACKEND LAYER
 │    ├── /actions/         # Server Actions (The Entry Points for the UI)
 │    ├── /services/        # The Brain (Complex Business Logic & AI calls)
 │    └── /db/              # Drizzle Schema & Connection
 │
 └── /lib                   # 4. UTILITIES (Helpers)
      ├── ai-config.ts      # OpenRouter configuration
      ├── uploadthing.ts    # Storage Config
      └── utils.ts          # Tailwind Class Merger
```

---

## 4. Database Schema & Architecture

The database is structured using **Drizzle ORM** to support comprehensive medical data and user interactions. The schemas define strict types and relationships ensuring data integrity.

### 4.1 Users & Authentication

Powered seamlessly by BetterAuth, managing robust user sessions and identity logic:

```typescript
export const user = pgTable("users", {
    id: text("id").primaryKey(),
    name: text("name").notNull(),
    email: text("email").notNull().unique(),
    emailVerified: boolean("email_verified").notNull(),
    image: text("image"),
    role: text("role").default('user').notNull(),
    banned: boolean("banned").default(false).notNull(),
    createdAt: timestamp("created_at").notNull(),
    updatedAt: timestamp("updated_at").notNull()
});
```

### 4.2 Medical & Pharmaceutical Data

Central to HelloMed's feature set is the vast relational structure separating actual branded medicines from their underlying generic compounds.

**Generics Schema:** Holds deep pharmacological descriptions for medical reference:

```typescript
export const generic = pgTable("generics", {
    id: text("id").primaryKey(),
    name: text("name").notNull(),
    drugClassId: text("drug_class_id").references(() => drugClass.id),
    indicationDescription: text("indication_description"),
    dosageDescription: text("dosage_description"),
    sideEffectsDescription: text("side_effects_description"),
    pregnancyAndLactationDescription: text("pregnancy_and_lactation_description"),
    // ... extensive documentation fields
});
```

**Medicines Schema:** Maps brand names, strengths, and pricing to their specific generic:

```typescript
export const medicine = pgTable("medicines", {
    id: text("id").primaryKey(),
    brandName: text("brand_name").notNull(),
    strength: text("strength").notNull(),
    genericId: text("generic_id").references(() => generic.id),
    manufacturerId: text("manufacturer_id").references(() => manufacturer.id),
    price: text("price"),
});
```

### 4.3 User Health Analytics

To capture anonymized analytics while separating PII from the activity:

```typescript
export const userActivity = pgTable("user_activities", {
    id: text("id").primaryKey(),
    userId: text("user_id").notNull().references(() => user.id, { onDelete: 'cascade' }),
    type: text("type").notNull(), // SEARCH, SCAN, UPLOAD_REPORT
    details: text("details"), 
    createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

---

## 5. AI OCR Architecture: The Grounded Vision Pipeline & Trigram Search

Handling Bangladeshi prescriptions involves tackling complex cursive, mixed Bangla/English text, aggressive shorthand (e.g., Cef-3 bd), and poor image quality. Standard OCR and naive LLMs often hallucinate in these conditions. To resolve this, HelloMed implements a highly resilient 4-Step (Grounded Vision pipeline utilizing advanced Postgres fuzzy matching.

### Step 0: Client-Side Optimization (The Pre-processor)

* **Process:** Image is heavily downscaled locally (longest edge max 1500px).
* **Binarization Filter:**

  ```javascript
  // Binarization/Contrast enhancement loop in HTML5 Canvas
  for (let i = 0; i < data.length; i += 4) {
      const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
      // thresholding to darken mid-tones and whiten light tones
      const v = avg > 150 ? 255 : Math.max(0, avg - 50);
      data[i] = v; data[i + 1] = v; data[i + 2] = v;
  }
  ```

* **Why:** The optimized black-and-white lightweight image prevents 10-second server timeouts and massively reduces LLM token costs.

### Step 1: The Sniper Vision Model (The Eyes)

* **Model:** `qwen/qwen-2.5-vl-72b-instruct` via OpenRouter.
* **Process:** The AI acts as a Sniper, strictly extracting raw text line-by-line without attempting to format JSON or guess generic names. Output is an array of raw, likely misspelled guesses (e.g., `["Cef 3", "Napa ext"]`).

### Step 2: The Database Intercept via Trigram Fuzzy Search (The Grounding)

* **The Problem:** AI inherently lacks knowledge of local Bangladeshi medicine brands and will hallucinate names if left uncorrected.
* **The Solution (`pg_trgm`):** We intercept the AI's raw guesses and cross-reference them with our verified local PostgreSQL database using Trigram fuzzy matching. A trigram breaks text into chunks of 3 letters to mathematically determine similarities between strings regardless of severe misspellings.
* **The Core SQL Query:**

  ```sql
  SELECT DISTINCT
      m.brand_name,
      g.name as generic_name,
      GREATEST(
          similarity(LOWER(m.brand_name), $searchTerm),
          similarity(LOWER(g.name), $searchTerm)
      ) as sim_score
  FROM medicines m
  LEFT JOIN generics g ON m.generic_id = g.id
  WHERE 
      LOWER(m.brand_name) % $searchTerm OR LOWER(g.name) % $searchTerm
  ORDER BY sim_score DESC LIMIT 3
  ```

* **Output:** This query yields a list of statistical database matches for each raw string (e.g., `"Cef 3"` heavily matches `"Cef-3 250mg"` and `"Cefur"`).

### Step 3: The LLM Judge (The Brain)

* **Model:** `deepseek/deepseek-v3.2-speciale`.
* **Process:** The raw OCR (Step 1) AND the DB Trigram Hits (Step 2) are sent to DeepSeek. Acting as a Bangladeshi pharmacist, the LLM maps the raw readings to the correct DB options, utilizing clinical logic (like understanding shorthand abbreviations) to discard irrelevant hits and return perfectly structured JSON mapped exactly to our DB schema.

**Observability:** The API returns an evidence chain (times for each step, raw outputs, DB hits) allowing robust debugging:

```json
{
  "success": true,
  "debug": {
    "step1_raw_ocr_output": ["Cef 3", "Napa ext"],
    "step2_database_hits": [
      {"query": "Cef 3", "matches": ["Cef-3 250mg", "Cefur"]},
      {"query": "Napa ext", "matches": ["Napa Extend 665mg"]}
    ]
  },
  "final_data": [
    { "name": "Cef-3 250mg", "generic": "Cefixime" },
    { "name": "Napa Extend 665mg", "generic": "Paracetamol" }
  ]
}
```

---

## 6. Medo: AI Chatbot & Health Assistant (Dual Extraction Pipeline)

**Medo** is the patient-facing intelligence layer of HelloMed. To bypass the unreliability and high cost of LLM-based OCR on every single chat message, the app uses a **Vault-Centric Architecture**.

### The Upload Phase (Pre-Chat)

1. **The Router:** When a user uploads a document to their Vault, Medo's backend classifies it as a Prescription or a Pathology/Lab Report.
2. **Dual Extraction:**
   * **Prescriptions:** Processed through the Grounded Vision Pipeline, generating a verified JSON array of drugs and dosages.
   * **Lab Reports:** Sent through an LLM-based OCR pipeline tailored for tabular data, generating a structured JSON of test names, values, units, and reference ranges.
3. Both the original image and the lightweight JSON are stored securely in Backblaze B2.

### The Chat Phase

1. When chatting with Medo, the user attaches a file from their Vault (via a bottom-sheet UI).
2. The backend injects the **structured JSON data** into Medo's LLM context, entirely hidden from the user, rather than sending the heavy image. This practically eliminates hallucinations and severely reduces API costs.
3. **Ephemeral Persistence:** Medo's chat history is stored securely in the browser's `sessionStorage`. If the browser or tab is closed, the context is permanently wiped, ensuring maximum privacy for shared family devices.

---

## 7. UI/UX Specifications & Core Workflows

### 7.1. Voice UX (Review Before Send)

Powered by `openai/whisper-large-v3-turbo`, users can seamlessly interact with Medo using voice (English, Bangla, Banglish).

* **Process:** Tapping the microphone pulses the background. Audio is transcribed and populates the input box.
* **Countdown System:** A 5-second visual countdown ring appears. If untouched, it auto-sends to Medo. If the user edits the text, the countdown is canceled, balancing hands-free usage with accuracy.

### 7.2. Starter Chips & Action Plan Export

* **Starter Chips:** Floating, horizontally scrollable suggestions ("Are my results normal?") to aid users with low health literacy.
* **Exportable Reports:** Because Medo's chat is ephemeral, significant AI responses (like a 7-day meal plan) can be exported to a single-page PDF on the client-side (`html2pdf.js` or `react-to-pdf`) and saved permanently in the Vault.

### 7.3. Medicine Scanner Workflow

1. User scans a prescription or medicine box.
2. Image is compressed locally with the Canvas Binarization filter.
3. Grounded Vision OCR identifies the drug via the Trigram intercept.
4. UI presents Deciphered Name, Pricing, Usage, and Generic Alternatives.

---

## 8. Safety, Guardrails & Emergency Protocols

HelloMed actively prevents misdiagnosis and provides immediate intervention during crises via a **Two-Tiered Emergency Bypass**.

### Tier 1: Smart Regex (Local, Zero Token Cost)

* Executes on the client/edge before any OpenRouter API call to Medo.
* Uses pattern matching to catch English and Banglish emergency phrases.

  ```javascript
  const heartEmergency = /(buk|book|chest).*(betha|bata|pain|dhor|dhuk)/i;
  if (heartEmergency.test(userInput)) {
      abortAiRequest();
      triggerEmergencyUI("Call 999 - Emergency Services");
  }
  ```

* **Action:** Aborts AI request instantly and renders a hardcoded Red UI Block: **"Call 999 - Emergency Services"**.

### Tier 2: The LLM Failsafe (Tool Calling)

* If an unusual emergency phrase bypasses Regex, Medo is equipped with a `trigger_emergency()` tool via the Vercel AI SDK.
* If invoked, it halts text stream generation and triggers the same Red UI Emergency Block.

### Visual Severity Coding

To assist low-literacy users, Medo's reports use color-coded icons:

* 🟢 **Green:** Normal / Routine
* 🟡 **Yellow:** Caution / Consult doctor on next visit
* 🔴 **Red:** Alert / Requires immediate specialist attention

---

## 9. Pharmaceutical Analytics Portal (Experimental B2B Feature)

A dedicated dashboard providing anonymized, real-time market intelligence for pharmaceutical companies and health researchers.

* **Anonymized Data Aggregation:** Every scan or search isolates the core medical data (drug requested, alternatives viewed) from all Personally Identifiable Information (PII) and stores it in the `user_activities` table.
* **Disease Heatmaps:** Visual representations of localized disease outbreaks based on scan frequency by region.
* **Market Trends & Generic Adoption:** Real-time statistics showing patient migration to generic alternatives, aiding supply chain and pricing strategies.
* **Exportable Reports:** Utilitarian UI allowing CSV/Excel exports for external analysis.

---

## 10. Implementation Roadmap (11 Phases)

| Phase | Task Description | Status |
| :--- | :--- | :--- |
| **P1** | **Project Genesis:** Initialize Next.js 16, folder structure, and landing page. | Completed |
| **P2** | **Auth Core:** Implement BetterAuth and protected dashboard routes. | Completed |
| **P3** | **Data Foundation:** Seed database with medicine CSVs (120k+ entries). | Completed |
| **P4** | **Instant Lookup:** Build high-speed medicine search and info cards. | Completed |
| **P5** | **Report Foundation:** UploadThing integration. | In Progress |
| **P6** | **Medo Chatbot (Basic):** Basic OCR and text extraction via OpenRouter. | In Progress |
| **P7** | **Medo Chatbot (Advanced):** Multilingual Bangla summaries and specialist suggestions. | In Progress |
| **P8** | **Vision Scanner:** Prescription/Packaging OCR with hybrid local DB matching. | In Progress |
| **P9** | **Generic Matcher:** Feature to find cheaper alternatives + Accessibility bubble. | Pending |
| **P10** | **Pharma Dashboard:** Disease heatmaps and exportable regional analytics. | Pending |
| **P11** | **Polish & Deployment:** Offline queue logic and final hosting. | Pending |

---

## 11. Security and Data Privacy

* **Strict PII Decoupling:** Analytics data strictly excludes user identities.
* **Ephemeral Processing:** Chat sessions are never stored persistently on the backend, existing only in browser memory.
* **Secure Vaults:** Medical documents are stored with strict access controls tied exclusively to authenticated sessions.
* **Database Access:** Robust row-level configurations and role-based access control (Admin vs User) via Drizzle and Next.js middleware.
