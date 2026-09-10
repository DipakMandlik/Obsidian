
## End-to-End IDP Platform Summary

### What We Built

An **Intelligent Document Processing (IDP) platform** entirely on Snowflake - from file drop to natural language Q&A.

---

### The Flow (7 stages)

**1. File Drop (LANDING)**

- PDFs are uploaded to `@IDP_DEV.LANDING.RAW_INBOUND` stage
- A **stream** (`RAW_INBOUND_STREAM`) detects new file arrivals automatically
- Path convention: `/{source_system}/{doc_family}/ingest_date=YYYY-MM-DD/{uuid}__{filename}.pdf`

**2. Register (T10) — No AI**

- `SP_REGISTER` picks up new files, deduplicates (by MD5), and inserts into `DOC_REGISTRY` with status tracking
- Pure SQL — no AI needed here

**3. Parse (T20) — Snowflake AI**

- `SP_PARSE` uses **AI_PARSE_DOCUMENT** to extract raw text from PDFs
- Result goes to `BRONZE.DOC_PARSED`
- This is where **AI is required** - OCR/text extraction from documents

**4. Route (T30) — Template Matching (No AI)**

- `SP_ROUTE` matches the parsed text against **3 pre-configured templates**: `CONTRACT`, `INVOICE`, `PURCHASE_ORDER`
- Uses **structural anchors** (keyword patterns with weights) and **regex anchors** to score each template
- If score >= threshold (0.5-0.6) → **DETERMINISTIC lane** (no AI needed)
- If **no template matches** → document is escalated to the **MODEL lane** (AI required)

**5a. Deterministic Extract (T40A) — No AI**

- When template matched: `SP_EXTRACT_DETERMINISTIC` runs **regex rules** from the template to pull fields (contract_id, dates, amounts, parties, etc.)
- Confidence = 1.0, fast, zero AI cost
- Example: Invoice INV-2025-201 extracted all 8 fields deterministically

**5b. Model Extract (T40B) — AI Required**

- When template didn't match or regex failed: `SP_EXTRACT_MODEL` uses **Cortex AI (LLM)** to extract fields
- Confidence = 0.7 (lower), uses AI credits
- Example: Contract CTR-2025-C10 was **ESCALATED** to model because regex didn't capture all required fields

**6. Validate (T50) — No AI**

- `SP_VALIDATE` checks extracted fields against **validation rules** (required fields present, arithmetic checks like subtotal + tax = total, format checks)
- If validation fails → retry via model lane (T40B_RETRY → T50_RETRY)

**7. Resolve & Aggregate (T60/T70) — AI**

- `SP_RESOLVE` does **entity resolution** — links "Testco Industries" across documents into canonical entity clusters
- `SP_AGGREGATE` builds cross-document insights in `GOLD`

---

### Where Things Are Best (No AI = Fast + Free)

|Step|AI Required?|What Handles It|
|---|---|---|
|File arrival detection|No|Streams + Tasks|
|Registration & dedup|No|Pure SQL|
|Template matching/routing|No|Regex + weighted anchors|
|Deterministic field extraction|No|Regex from templates|
|Validation|No|Rule engine (SQL)|
|Metrics & ops logging|No|Pure SQL|

### Where AI Is Required

|Step|AI Function|Why|
|---|---|---|
|Document parsing (OCR/text)|AI_PARSE_DOCUMENT|Can't read PDFs without it|
|Model-lane extraction|AI_COMPLETE (LLM)|When regex/templates don't match|
|Entity resolution|AI (LLM)|Fuzzy matching across docs|
|Q&A via CoWork|Cortex Analyst + Semantic View|Natural language queries|

---

### The CoWork / Q&A Layer

- A **Semantic View** (`IDP_DEV.SERVE.IDP_SEMANTIC_VIEW`) sits on top of the serving views
- Connected to **Snowflake CoWork** (formerly Snowflake Intelligence) so users can ask questions like:
    - _"What is the contract value for CTR-2025-C10?"_
    - _"Which invoices are from Nimbus Dynamics?"_
    - _"How many documents were processed today?"_

---

### Demo Highlights

**Best for demo (works great now):**

- Drop a PDF → watch it flow through all 7 stages automatically (stream-triggered task DAG)
- Invoice processing: template matched, all fields extracted deterministically, 100% confidence, zero AI cost
- Ask CoWork natural language questions against extracted data

**Where AI shines (demo the escalation):**

- Contract where regex partially failed → automatically escalated to LLM → still extracted all fields (at 0.7 confidence)
- Show the two lanes side-by-side: deterministic (fast/free) vs. model (flexible/costs credits)

**Key message for demo:** The platform uses AI only when it has to - templates handle the predictable documents (zero cost), AI handles the exceptions. This is a cost-optimized, production-grade design.