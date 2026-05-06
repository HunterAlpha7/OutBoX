# Project HelloMed: Comprehensive Documentation

## 1. Project Overview

HelloMed is an advanced healthcare application specifically tailored for the Bangladeshi landscape. Its primary goal is to bridge the gap between complex medical documentation and patient understanding, while also providing valuable, anonymized data insights for pharmaceutical companies. The application is built to handle the pragmatic challenges of the local healthcare sector, including illegible handwritten prescriptions, price sensitivity, and varying levels of health literacy among patients.

### Key Objectives

* Provide a minimalistic, highly accessible patient interface.
* Offer instant deciphering of messy, handwritten prescriptions.
* Provide cost-saving generic alternatives to prescribed medications.
* Ensure data privacy and security, especially for shared family devices.
* Aggregate anonymized medical data for public health and pharmaceutical analytics.

---

## 2. Technical Stack

The project employs a modern, serverless technology stack designed for performance, modularity, and cost-efficiency.

* **Framework:** Next.js 16 (App Router) for a seamless server-side rendering (SSR) and client-side routing (CSR) experience.
* **Language:** TypeScript for industry-grade type safety.
* **Authentication:** BetterAuth for comprehensive session management.
* **Database:** PostgreSQL (via Neon/Supabase) for serverless, efficient relational data storage.
* **ORM:** Drizzle ORM for lightweight, high-performance database interactions.
* **Storage:** UploadThing and Backblaze B2 for secure file and image uploads (Vault).
* **AI Provider:** OpenRouter as a gateway for utilizing various language models (Qwen, DeepSeek, GPT-4o mini, Whisper).
* **UI/Styling:** Shadcn/UI combined with Tailwind CSS for clean, professional, and modular components.

---

## 3. System Architecture

### 3.1. Serverless Modular Monolith

The application utilizes Next.js Server Actions as its core backend mechanism. This eliminates the need for a separate, persistent Node.js server, relying instead on serverless functions that consume resources only during active requests.

### 3.2. Separation of Concerns

HelloMed strictly follows a feature-based modular architecture:

1. **Routing Layer (UI):** Clean React Server and Client Components. The UI never interacts directly with the database.
2. **UI Components Layer:** Reusable Shadcn primitives and feature-grouped components.
3. **Backend Layer (Server Actions & Services):** Server actions act as entry points for the UI, which then call dedicated services handling complex business logic and external AI API calls.
4. **Database Layer:** Drizzle schemas and direct PostgreSQL connections.

   *Example Drizzle Schema snippet (`src/db/schema.ts`):*
   ```typescript
   export const medicine = pgTable("medicines", {
       id: text("id").primaryKey(),
       brandName: text("brand_name").notNull(),
       strength: text("strength").notNull(),
       genericId: text("generic_id").references(() => generic.id),
       price: text("price"),
   });
   ```

### 3.3. AI OCR Architecture (Grounded Vision Pipeline)

To handle complex, cursive, and often multilingual (Bangla/English) prescriptions, a multi-step OCR pipeline is implemented to prevent AI hallucinations and server timeouts.

1. **Client-Side Optimization:** Images are downscaled and binarized (contrast adjustment) directly in the user's browser using HTML5 Canvas before uploading. This significantly reduces payload size and token costs.

   *Example Binarization Logic (`src/components/med-scanner/OCRScanner.tsx`):*
   ```javascript
   // Binarization/Contrast enhancement loop
   for (let i = 0; i < data.length; i += 4) {
       const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
       // thresholding to darken mid-tones and whiten light tones
       const v = avg > 150 ? 255 : Math.max(0, avg - 50);
       data[i] = v; data[i + 1] = v; data[i + 2] = v;
   }
   ```

2. **Vision Model Extraction:** A specialized vision model (e.g., Qwen-2.5-VL) processes the optimized image, strictly extracting raw text line-by-line without attempting to format or guess medicine names.
3. **Database Interception:** The raw extracted text is cross-referenced against a verified local PostgreSQL database using Trigram fuzzy matching (`pg_trgm`) to find potential statistical matches for medicine names.

   *Example Trigram Search Query (`src/app/api/ocr/route.ts`):*
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
4. **LLM Verification:** An advanced reasoning model (e.g., DeepSeek v3.2 Speciale) acts as a judge, analyzing the raw OCR output alongside the database hits to intelligently deduce the correct medicines, accounting for shorthand abbreviations.

---

## 4. Core Workflows

### Medicine Scanner Workflow

1. User scans a prescription or medicine box.
2. The image is compressed on the client side and sent to the OCR pipeline.
3. The AI extracts text and grounds it using local database matching.
4. The user is presented with the deciphered medicine name, current pricing, usage instructions, and cheaper generic alternatives.

---

## 5. Experimental Features (Planned / In Development)

The following features represent the next generation of HelloMed's capabilities. They are currently in the experimental phase and are designed to enhance both patient care and pharmaceutical intelligence.

### 5.1. AI Medical Chatbot

A highly contextual, patient-facing intelligence layer designed for low cognitive load and linguistic flexibility (Bangla, English, Banglish).

**Key Capabilities:**

* **Vault-Centric Context:** Instead of sending heavy images to the LLM during a chat, the system relies on structured JSON data extracted and stored securely in the user's "Vault" upon initial upload. This drastically reduces hallucinations and API costs.
* **Ephemeral Privacy:** Chat sessions are not persisted in the database. They exist entirely within the browser's session storage. Once the chat or browser is closed, the context is wiped completely, ensuring maximum privacy.
* **Voice UX:** Integration with speech-to-text models (like Whisper) allows users to input queries via voice in mixed languages. A visual countdown system allows users to review the transcribed text before it is sent to the AI.
* **Two-Tiered Emergency Bypass:**
  1. **Tier 1 (Regex):** Client-side regex matching intercepts emergency keywords (e.g., chest pain, severe bleeding) in English and Banglish, aborting the AI request and instantly displaying emergency contact information.
     
     *Example Emergency Intercept:*
     ```javascript
     const heartEmergency = /(buk|book|chest).*(betha|bata|pain|dhor|dhuk)/i;
     if (heartEmergency.test(userInput)) {
         abortAiRequest();
         triggerEmergencyUI("Call 999 - Emergency Services");
     }
     ```
  2. **Tier 2 (Tool Calling):** The LLM is equipped with an emergency override tool. If it detects a critical situation not caught by regex, it halts normal generation and triggers the emergency UI block.

### 5.2. Pharmaceutical Analytics Portal

A dedicated, B2B dashboard providing anonymized, real-time market intelligence for pharmaceutical companies and health researchers.

**Key Capabilities:**

* **Anonymized Data Aggregation:** Every time a user scans a prescription or searches for a medication, the core data (medicine requested, generic alternatives viewed, geographic region) is stripped of all Personally Identifiable Information (PII) and stored in a secure analytics table.
* **Disease Heatmaps:** Visual representations of localized disease outbreaks or prevalent conditions based on the frequency of specific medication scans in different regions.
* **Market Trends & Generic Adoption:** Real-time statistics showing how often patients opt for generic alternatives over branded prescriptions, allowing pharmaceutical companies to adjust pricing and supply chain strategies.
* **Exportable Reports:** Data-heavy, utilitarian interfaces that allow researchers to export analytical models and trends into standard formats (CSV, Excel) for external review.

---

## 6. Security and Data Privacy

* **No PII in Analytics:** Strict decoupling of user identity from medical scan data.
* **Ephemeral Processing:** Chatbot data is never stored persistently on the backend.
* **Secure Vaults:** Uploaded medical documents are stored in secure buckets (Backblaze B2/UploadThing) with strict access controls tied exclusively to the authenticated user's session.
