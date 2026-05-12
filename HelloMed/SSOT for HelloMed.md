# TEAM INTERNAL DOCUMENT. ONLY FOR INTERNAL AND TEAM PERSONNELS. NOT INTENDED FOR EXTERNAL USE/ DEMONSTRATION

# Project HelloMed: Comprehensive AI-Powered Medical Companion Documentation

## 1. Project Vision & Overview

**HelloMed** is a high-utility, advanced healthcare application specifically tailored for the Bangladeshi landscape. Its primary goal is to bridge the gap between complex medical documentation and patient understanding, while also providing valuable, anonymized data insights for pharmaceutical companies. The application is built to handle the pragmatic challenges of the local healthcare sector, including illegible handwritten prescriptions, price sensitivity, and varying levels of health literacy among patients.

The application is meticulously designed to handle the pragmatic challenges of the local healthcare sector, including:

* **Illegible Handwriting:** Deciphering complex, cursive, and often multilingual (Bangla/English) handwritten prescriptions.
* **Price Sensitivity:** Providing cost-saving generic alternatives to prescribed medications, empowering patients with instant price comparisons.
* **Health Literacy:** Offering simplified explanations for complex English medical reports.
* **Public Health Intelligence:** Aggregating anonymized medical data for public health and pharmaceutical analytics via a dedicated Role-Based Access Control (RBAC) portal.

### Key UX Principles

* **Minimalistic Patient UI:** Focus on ease of use, high contrast, and accessibility to accommodate all user demographics.
* **Utilitarian Pharma & Admin Dashboards:** Data-heavy, clean, and export-focused interface for researchers, pharmaceutical partners, and system administrators.
* **Industry Standard UX:** Implementation of Skeleton Screens (no blank loading states) and seamless transitions for a fluid user experience.
* **Bangladeshi Context Integration:**
  * **Price Transparency:** Instant display of medicine prices and cheaper generic alternatives using real-time scraping fallbacks.
  * **Language Accessibility:** Simplified summaries for complex texts.

---

## 2. Technical Stack

The project employs a modern, serverless technology stack designed for performance, modularity, and cost-efficiency.

| Layer | Technology | Reason / Implementation Details |
| :--- | :--- | :--- |
| **Framework** | Next.js 16 (App Router) | Provides a seamless, "reloadless" SSR (Server-Side Rendering) and CSR (Client-Side Rendering) experience. |
| **Language** | TypeScript | Ensures industry-grade type safety and modularity across the codebase. |
| **Authentication** | BetterAuth | Comprehensive session management, credential, emailOTP (for forgot password flows), and OAuth integration. |
| **Database** | PostgreSQL (via Neon/Supabase) | Serverless, efficient, relational data storage optimized for complex queries and Trigram fuzzy search. |
| **ORM** | Drizzle ORM | Lightweight, high-performance SQL-like control for type-safe database interactions. |
| **Storage** | UploadThing & Backblaze B2 | Best-in-class Next.js integration for secure file, PDF, and image uploads. |
| **AI Provider** | OpenRouter SDK | Model-agnostic gateway for accessing DeepSeek and Qwen-2.5-VL models. |
| **UI/Styling** | Shadcn/UI + Tailwind CSS | Professional components with a clean, modular aesthetic and utility-first styling. |
| **Mapping** | Leaflet & OpenStreetMap | Open-source geographic rendering for disease and medical heatmaps. |

---

## 3. System Architecture & Modularity

### 3.1. Serverless Modular Monolith

The application utilizes **Next.js Server Actions** as its core backend mechanism. This approach eliminates the need for a separate, persistent Node.js server, relying instead on serverless functions that consume resources only during active requests. This optimizes both performance and hosting costs.

### 3.2. Separation of Concerns & Routing

HelloMed strictly follows a feature-based modular architecture, mapped heavily through Next.js route groups to separate access layers:

1. **Routing Layer (UI):** Clean React Server and Client Components (`app/`). The UI never interacts directly with the database.
   * `(patient)`: Patient-facing dashboards and scanning interfaces.
   * `pharma`: Analytical portal for pharmaceutical representatives.
   * `admin`: Superadmin dashboard for platform management and RBAC invitations.
   * `(marketing)`: Public-facing landing pages and "How It Works" documentation.
2. **UI Components Layer:** Reusable Shadcn primitives and feature-grouped components.
3. **Backend Layer (Server Actions & Services):** Server actions act as entry points for the UI. They delegate complex business logic, database queries, and external AI API calls.
4. **Database Layer:** Drizzle schemas and direct PostgreSQL connections (`src/db/`).

---

## 4. Database Schema & Architecture

The database is structured using **Drizzle ORM** to support comprehensive medical data, user interactions, and robust Role-Based Access Control (RBAC).

### 4.1 Users, Authentication & Invitations

Powered seamlessly by BetterAuth, managing robust user sessions, role logic, and OTP flows.

```typescript
export const user = pgTable("users", {
    id: text("id").primaryKey(),
    name: text("name").notNull(),
    email: text("email").notNull().unique(),
    emailVerified: boolean("email_verified").notNull(),
    image: text("image"),
    role: text("role").default('user').notNull(), // user, pharma_rep, superadmin
    banned: boolean("banned").default(false).notNull(),
    createdAt: timestamp("created_at").notNull(),
    updatedAt: timestamp("updated_at").notNull()
});

export const invitation = pgTable("invitations", {
    id: text("id").primaryKey(),
    email: text("email").notNull().unique(),
    role: text("role").notNull().default('pharma_rep'),
    invitedBy: text("invited_by").notNull(), // Name or Email of the superadmin
    status: text("status").notNull().default('pending'), // pending, accepted
    createdAt: timestamp("created_at").defaultNow().notNull(),
    expiresAt: timestamp("expires_at").notNull(),
});
```

### 4.2 Medical & Pharmaceutical Data (Internal DB)

Central to HelloMed's core feature set is the vast relational structure separating branded medicines from their underlying generic compounds. 

* **Generics Schema:** Holds deep pharmacological descriptions (indications, side effects, dosage).
* **Medicines Schema:** Maps brand names, strengths, and pricing to their specific generic ID and manufacturer.

### 4.3 The External Fallback Schema

When a medicine is not found in our primary database, the system invokes an active web scraper (Medex) and persists the structured response.

```typescript
export const externalMedicine = pgTable("external_medicines", {
    id: text("id").primaryKey(),
    brandName: text("brand_name").notNull(),
    genericName: text("generic_name"),
    strength: text("strength"),
    manufacturer: text("manufacturer"),
    dosageForm: text("dosage_form"),
    price: text("price"),
    packageContainer: text("package_container"),
    packageSize: text("package_size"),
    indicationDescription: text("indication_description"),
    dosageDescription: text("dosage_description"),
    source: text("source").notNull().default('MEDEX'),
    sourceUrl: text("source_url"),
    createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

### 4.4 Geospatial Data & Telemetry (Non-PII)

To capture anonymized analytics while separating PII from the activity, empowering the Pharma Dashboard Heatmaps:

```typescript
export const scanRecord = pgTable("scan_records", {
    id: text("id").primaryKey(),
    medicineName: text("medicine_name").notNull(),
    genericName: text("generic_name"),
    source: text("source").notNull(), // e.g. "HelloMed DB", "Medex API"
    approxLat: text("approx_lat"),
    approxLng: text("approx_lng"),
    city: text("city"),
    region: text("region"),
    country: text("country"),
    createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

---

## 5. AI OCR Architecture: The Grounded Vision Pipeline

Handling Bangladeshi prescriptions involves tackling complex cursive, mixed Bangla/English text, aggressive shorthand, and poor image quality. HelloMed implements a highly resilient 4-Step Grounded Vision pipeline. 

**Infrastructure Offloading:** To successfully bypass Vercel's strict 10-second serverless execution timeout during heavy image processing, the core OCR inference pipeline has been strategically offloaded to a dedicated **Render** backend, ensuring stable and reliable extractions.

### Step 0: Client-Side Optimization
* **Process:** Image is heavily downscaled locally. Binarization and contrast enhancement loops run in an HTML5 Canvas to darken mid-tones and whiten light tones.
* **Why:** The optimized lightweight image prevents server timeouts and massively reduces LLM token costs.

### Step 1: The Sniper Vision Model (The Eyes)
* **Model:** `qwen/qwen-2.5-vl-72b-instruct` via OpenRouter.
* **Process:** Extracts raw text line-by-line without attempting to format JSON or guess generic names. Output is an array of raw, likely misspelled guesses (e.g., `["Cef 3", "Napa ext"]`).

### Step 2: The Database Intercept via Trigram Fuzzy Search (The Grounding)
* **Process:** We intercept the AI's raw guesses and cross-reference them with our verified local PostgreSQL database using Trigram (`pg_trgm`) fuzzy matching to mathematically determine similarities between strings regardless of severe misspellings.

### Step 3: The LLM Judge (The Brain)
* **Model:** `deepseek/deepseek-v3.2-speciale`.
* **Process:** The raw OCR (Step 1) AND the DB Trigram Hits (Step 2) are sent to DeepSeek. Acting as a Bangladeshi pharmacist, the LLM maps the raw readings to the correct DB options, returning structured JSON.

### Step 4: The Medex Fallback & External Sync (The Safety Net)
* **Process:** If Step 3 yields confidence that a medicine is legitimate but it is missing from the primary DB, the backend falls back to an active web scraping pipeline targeting Medex.
* **Action:** It dynamically scrapes the missing generic names, dosage descriptions, and pricing data, actively injecting it into the `externalMedicine` table so that subsequent scans of the same drug resolve instantly.

---

## 6. Planned Features: Medo AI Chatbot & Health Assistant (Dual Extraction)

**Medo** is the planned patient-facing intelligence layer of HelloMed. While temporarily paused in implementation to stabilize core OCR features and resolve Vercel AI SDK dependency conflicts, it remains a central planned feature utilizing a **Vault-Centric Architecture** to bypass the unreliability and high cost of LLM-based OCR on every single chat message.

### The Upload Phase (Pre-Chat)

1. **The Router:** When a user uploads a document to their Vault, Medo's backend classifies it as a Prescription or a Pathology/Lab Report.
2. **Dual Extraction:**
   * **Prescriptions:** Processed through the Grounded Vision Pipeline, generating a verified JSON array of drugs and dosages.
   * **Lab Reports:** Sent through an LLM-based OCR pipeline tailored for tabular data, generating a structured JSON of test names, values, units, and reference ranges.
3. Both the original image and the lightweight JSON are stored securely in Backblaze B2.

### The Chat Phase

1. When chatting with Medo, the user attaches a file from their Vault (via a bottom-sheet UI).
2. The backend injects the **structured JSON data** into Medo's LLM context, entirely hidden from the user, rather than sending the heavy image. This practically eliminates hallucinations and severely reduces API costs.
3. **Ephemeral Persistence:** Medo's chat history is stored securely in the browser's `sessionStorage`. If the browser or tab is closed, the context is permanently wiped, ensuring maximum privacy.

### Voice UX & Starter Chips (Planned)

* **Voice UX:** Powered by `openai/whisper-large-v3-turbo` with a 5-second visual countdown review system for hands-free accuracy.
* **Starter Chips:** Floating suggestions ("Are my results normal?") to aid users with low health literacy.
* **Exportable Reports:** Significant AI responses (like a 7-day meal plan) can be exported to a single-page PDF on the client-side.

---

## 7. UI/UX Specifications & Core Workflows

### 7.1. Patient Dashboard Workflow (Scan & Save)

1. **Capture:** User scans a prescription or medicine box.
2. **Processing:** Grounded Vision OCR identifies the drug via the Trigram intercept or Medex Fallback.
3. **The Cheap Alternative Engine:** Using the identified drug's generic name, the backend instantly queries the DB for alternative manufacturers producing the exact same generic compound, sorting them by lowest price.
4. **Display:** UI presents Deciphered Name, Pricing, Usage, and the list of cheaper "Generic Alternatives".

### 7.2. Pharmaceutical Analytics Portal (Pharma Dashboard)

A dedicated, role-protected dashboard providing anonymized, real-time market intelligence for pharmaceutical companies.

* **Non-PII Telemetry:** Every scan triggers IP-based geolocation (using native web APIs/server headers), stripping user identities but logging approximate coordinates into `scanRecord`.
* **Disease & Trend Heatmaps:** Utilizes Leaflet and OpenStreetMap to render localized visual hotspots of specific generic drug scans, mapping potential regional outbreaks or demand spikes.
* **Market Trends:** Real-time statistics showing patient migration to generic alternatives, aiding supply chain strategies.

### 7.3. Superadmin Identity Management

* **Unified Login:** A single authentication portal that handles RBAC routing.
* **Invite System:** Superadmins possess a dedicated dashboard to generate secure email invitations specifically granting `pharma_rep` roles to corporate clients.
* **Account Management:** Oversight of all user tiers and system ban capabilities.

---

## 8. Safety, Guardrails & Emergency Protocols

HelloMed actively prevents misdiagnosis and provides immediate intervention during crises via a Smart Regex Failsafe.

* Executes on the client/edge before any OpenRouter API calls.
* Uses pattern matching to catch English and Banglish emergency phrases.
* If triggered, it aborts the AI request instantly and renders a hardcoded Red UI Block: **"Call 999 - Emergency Services"**.

---

## 9. Implementation Roadmap (14 Phases)

| Phase | Task Description | Status |
| :--- | :--- | :--- |
| **P1** | **Project Genesis:** Initialize Next.js 16, folder structure, and landing page. | Completed |
| **P2** | **Auth Core:** Implement BetterAuth and protected dashboard routes. | Completed |
| **P3** | **Data Foundation:** Seed database with medicine CSVs (120k+ entries). | Completed |
| **P4** | **Instant Lookup:** Build high-speed medicine search and info cards. | Completed |
| **P5** | **Report Foundation:** UploadThing integration and basic reports. | Completed |
| **P6** | **Vision Scanner:** Prescription/Packaging OCR via Render Backend. | Completed |
| **P7** | **Scraper Fallback:** Dynamic Medex fallback scraping into `externalMedicine`. | Completed |
| **P8** | **Generic Matcher:** "Cheap alternative" engine based on generic cross-referencing. | Completed |
| **P9** | **Pharma Dashboard:** Disease heatmaps via Leaflet and exportable regional analytics. | Completed |
| **P10** | **Admin Identity System:** Superadmin RBAC, corporate invites, and account management. | Completed |
| **P11** | **Forgot Password Flow:** Secure OTP verification loop via BetterAuth emailOTP. | Completed |
| **P12** | **Polish & Deployment:** Build stabilization, dependency cleanup, and Vercel optimization. | In Progress |
| **P13** | **Medo Chatbot (Basic):** Dual extraction integration and chat UI components. | Planned |
| **P14** | **Medo Chatbot (Advanced):** Voice UX (Whisper) and Multilingual Bangla summaries. | Planned |

---

## 10. Security and Data Privacy

* **Strict PII Decoupling:** Analytics data (`scanRecord`) strictly isolates geographic locations from user identities.
* **Secure Document Access:** Medical documents are stored with strict access controls tied exclusively to authenticated sessions.
* **Robust Auth Flows:** Implementation of `emailOTP` plugins ensures safe password recovery without compromising account integrity.
* **Database Access:** Robust row-level configurations and role-based access control (Superadmin vs Pharma vs User) via Next.js middleware and secure Server Actions.
