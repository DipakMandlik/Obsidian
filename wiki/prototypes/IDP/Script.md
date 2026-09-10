

# IDP CLIENT DEMO — MASTER PRESENTATION SCRIPT

## Recommended Flow

```text
SLIDE 1
Opening / Problem + Vision
        ↓
SLIDE 2
Solution Overview
        ↓
SLIDE 3
Technical Architecture
        ↓
SLIDE 4
What We Built / Design Principles
        ↓
SLIDE 5
End-to-End Detailed Pipeline
        ↓
SLIDE 6
Cost Optimization
        ↓
LIVE STREAMLIT DEMO
Upload → Process → Monitor → Explain
        ↓
SNOWFLAKE COWORK
Ask → Answer → Source
        ↓
SLIDE 7
Thank You
```

---

# 🎬 OPENING — BEFORE SLIDE 1

### Say:

> **“Good morning everyone, and thank you for joining us.”**
> 
> Today we are going to walk you through our **Intelligent Document Processing platform, built entirely on Snowflake**.
> 
> The objective of this solution is very simple:
> 
> **take unstructured documents, turn them into trusted structured information, and finally make that information accessible through analytics, search, and natural-language interaction.**
> 
> But the important part is not just extracting information from documents.
> 
> The important part is **how we process those documents efficiently, how we control AI consumption, how we validate the extracted information, and how we make the entire pipeline observable and production-ready.**
> 
> So we'll first walk through the solution architecture and design, and then we'll move into a live demonstration where we'll actually drop documents into the platform and watch them move through the pipeline end to end.
> 
> And finally, we'll take the resulting information and ask questions directly through **Snowflake CoWork**.”

**Pause.**

> “So let's start with the overall vision.”

---

# 🟦 SLIDE 1 — INTELLIGENT DOCUMENT PROCESSING

The first slide establishes the journey from unstructured documents to intelligent answers.

### Say:

> “At a high level, this is what we are building.”
> 
> “Organizations have information sitting across invoices, contracts, purchase orders, scanned documents, and other unstructured sources.”
> 
> “Traditionally, converting that information into something usable requires a combination of manual processing, document parsing, data engineering, and sometimes expensive AI processing.”
> 
> “Our approach is to bring that entire journey onto **Snowflake**.”
> 
> “We start with the document, process and structure the information, make it searchable and ultimately expose it through intelligent interfaces such as search, agents and Snowflake CoWork.”


### Point toward the right side:

> “And this is the journey we want to demonstrate today.”
> 
> **“Documents → Processing → Structured Information → Search → Answers.”**

### Then emphasize:

> “The key principle behind our implementation is that **AI should not be used simply because it is available.**
> 
> We use deterministic processing wherever the document is predictable, and we introduce AI when the document requires intelligence that deterministic logic cannot reliably provide.”

### Transition:

> “So that's the vision. Now let me show you how we have translated that vision into an actual Snowflake platform.”

---

# 🟦 SLIDE 2 - IDP PLATFORM OVERVIEW

This slide presents the broader platform: ingestion/orchestration, Bronze/Silver/Gold/Serve, governance, and the 75% deterministic / 25% model split.

### Say:

> “This is the overall platform architecture.”
> 
> “On the left, we have our input sources. These can be PDFs, Word documents, Excel files, scanned documents, and other formats.”
> 
> “Those documents enter our Snowflake environment through the ingestion layer.”

### Point to Bronze:

> “From there, the information moves through the traditional data layers.”
> 
> “**Bronze** represents the raw or minimally processed information.”
> 
> “**Silver** is where we perform the intelligence layer — classification, extraction, and validation.”
> 
> “**Gold** is where the information becomes business-ready through entity resolution and cross-document aggregation.”
> 
> “And finally, **Serve** exposes the trusted information for consumption.”

### Point to governance:

> “An important part of the architecture is that governance is not something we add later.”
> 
> “RBAC, access control, security, lineage, and auditability are part of the platform design.”

### Point to 75/25:

> “And this is one of the most important design decisions in the solution.”
> 
> **“Approximately 75 percent of predictable documents can follow our deterministic processing path, while approximately 25 percent require the model lane.”**
> 
> “That means we're not sending every document through an LLM.”
> 
> “We're deliberately controlling where AI is used.”

### Strong line:

> **“The architecture is deterministic-first, AI-when-required.”**

### Transition:

> “Now let's go one level deeper and look at what actually happens inside each stage.”

---

# 🟦 SLIDE 3 — DETAILED TECHNICAL ARCHITECTURE

This is your most important architecture slide. It shows Landing, Bronze, Silver, Gold, Serve, configuration/operations, RBAC, task DAG, model isolation, and the deterministic/model lanes.

## Opening

> “This is the detailed architecture of the platform that we have actually implemented.”

---

## LANDING

Point to Landing.

> “We begin in the Landing layer.”
> 
> “Documents arrive into our Snowflake internal stage, **RAW_INBOUND**.”
> 
> “A stream detects new file arrivals, so we're not relying on a user manually triggering every document.”
> 
> “The document is registered and tracked through its lifecycle.”

Then:

> “At registration, we perform deduplication and maintain document state.”

---

# T10 — REGISTER

> “The first processing task is **T10 Register**.”
> 
> “This is intentionally pure SQL.”
> 
> “There is no LLM involved.”
> 
> “We identify new files, perform deduplication, and create the corresponding record in our document registry.”

Emphasize:

> **“This is a good example of where AI provides no additional value, so we don't use it.”**

---

# T20 — PARSE

Point to Bronze.

> “Next is **T20 Parse**.”
> 
> “Here we use Snowflake's **AI_PARSE_DOCUMENT** capability to extract text from the PDF.”
> 
> “This is one of the places where AI is genuinely required because the system needs to interpret the document structure and extract readable text, including cases such as scanned content.”
> 
> “The result is stored in our Bronze parsed-document layer.”

Important:

> “Notice that we're still not extracting business fields at this point.”
> 
> “We're simply creating a reliable text representation of the document.”

---

# T30 — ROUTE

Point to Route.

> “Once we have parsed text, we move to **T30 Route**.”
> 
> “This is where the platform determines what type of document we're dealing with and, more importantly, which extraction strategy should be used.”
> 
> “We currently have templates for **Contract, Invoice, and Purchase Order**.”
> 
> “The router evaluates structural anchors, regex anchors, and source hints to calculate a template score.”

Then:

> “If the document matches a known template strongly enough, it goes down the deterministic lane.”
> 
> “If it doesn't match, we escalate it to the model lane.”

### Strong line:

> **“The router is effectively our AI cost-control gate.”**

---

# T40A — DETERMINISTIC

Point to green/blue lane.

> “On the deterministic side, we use regex rules defined against the document template.”
> 
> “For example, for an invoice we can identify invoice number, invoice date, vendor, subtotal, tax, total, and other known fields.”
> 
> “This path is fast, predictable, and has zero LLM extraction cost.”
> 
> “For a well-structured invoice, there is simply no reason to call an LLM.”

---

# T40B — MODEL

Point to model lane.

> “The second path is our model lane.”
> 
> “If the document doesn't match our deterministic rules, or deterministic extraction isn't sufficient, we use **Cortex AI / LLM-based extraction**.”
> 
> “This gives us flexibility for documents that are less predictable or have lower-confidence extraction.”

Then:

> “So we're not choosing deterministic versus AI globally.”
> 
> **“We're choosing the appropriate processing strategy document by document.”**

---

# T50 — VALIDATION

Point to validation.

> “Regardless of which extraction path we use, both lanes converge into **T50 Validate**.”
> 
> “This is critical.”
> 
> “We don't blindly trust extracted information.”
> 
> “We validate required fields, arithmetic relationships, formats, and ranges.”

Example:

> “For an invoice, for example, we can verify that subtotal plus tax equals the total.”

Then explain escalation:

> “If deterministic extraction fails validation, we don't immediately send everything to a human.”
> 
> “We first escalate that document to the model lane and give it one retry.”
> 
> “If it still fails, then it goes to the manual queue.”

### Say slowly:

> **“Deterministic first → validate → model retry → manual review.”**

---

# GOLD — ENTITY RESOLUTION

> “Once the data is validated, we move into Gold.”
> 
> “Here we perform entity resolution.”
> 
> “The purpose is to recognize that different documents may refer to the same real-world entity using slightly different names.”
> 
> “For example, the same organization might appear with different naming conventions across invoices and contracts.”
> 
> “We resolve those references into a canonical entity cluster.”

---

# T70 — AGGREGATION

> “After entity resolution, we can build cross-document aggregates and insights.”
> 
> “At this point, we're no longer looking at isolated documents.”
> 
> “We're connecting information across the document population.”

---

# SERVE

> “Finally, the information is exposed through our serving layer.”
> 
> “We have SQL views, a Semantic View, and Cortex Search.”
> 
> “And that becomes the foundation for both analytical consumption and natural-language interaction.”

---

# CONFIGURATION / OPERATIONS

Point left.

> “One important production principle is that configuration is treated as data rather than hard-coded logic.”
> 
> “Templates, prompts, and tunable settings are stored separately.”
> 
> “Operational information such as run logs, cost ledger, errors, and processing state are also captured.”

---

# GOVERNANCE

Point right.

> “And on the governance side, we have role separation, service identities, model isolation, bounded batches, deduplication, retry controls, and monitoring.”
> 
> “So this isn't simply a document extraction workflow.”
> 
> **“It is designed as a governed Snowflake application.”**

### Transition:

> “Now that you've seen the architecture, let me summarize the actual design philosophy behind it.”

---

# 🟦 SLIDE 4 — WHAT WE ARE BUILDING

This slide explains the platform objective, layers, orchestration/intelligence, Snowflake-native services, and the deterministic-first design.

### Say:

> “This slide summarizes the solution from a product and engineering perspective.”
> 
> “We're taking unstructured information and turning it into trusted, searchable, actionable information.”

Point through stages:

> “Landing handles registration.”
> 
> “Bronze handles parsing.”
> 
> “Silver is our intelligence layer.”
> 
> “Gold creates business-ready information.”
> 
> “Serve provides the consumption interfaces.”
> 
> “And finally, business users, analysts, and agents consume the results.”

### Point to orchestration:

> “Underneath this, Snowflake provides the orchestration and intelligence capabilities — Streams, Tasks, Cortex capabilities, and the data platform itself.”

### Point to "What's Different":

> “The main difference in our design is the **deterministic-first approach**.”
> 
> “We don't want to make every document an AI problem.”
> 
> “We want predictable documents to remain predictable.”
> 
> “AI is reserved for the cases where it provides additional value.”

### Strong statement:

> **“The result is a hybrid architecture: deterministic where we can be deterministic, intelligent where we need intelligence.”**

### Transition:

> “Now let's walk through the actual processing sequence that happens when a document enters the platform.”

---

# 🟦 SLIDE 5 — END-TO-END PIPELINE

This slide gives the cleanest stage-by-stage technical representation and shows the supporting CONFIG/OPS rails.

### Say:

> “This is the end-to-end processing sequence.”
> 
> “I'm going to use this exact sequence during the live demonstration.”

---

## Step 1 — LANDING

> “A document arrives in our RAW_INBOUND stage.”

---

## Step 2 — REGISTER

> “T10 registers the document, performs MD5-based deduplication, and establishes the document's processing state.”

---

## Step 3 — PARSE

> “T20 uses AI_PARSE_DOCUMENT to create the parsed text representation.”

---

## Step 4 — ROUTE

> “T30 evaluates the document against the configured templates.”

---

## Step 5 — EXTRACT

> “The document then takes one of two paths.”
> 
> “Approximately 75 percent can be processed deterministically.”
> 
> “The remaining documents go through Cortex-based model extraction.”

---

## Step 6 — VALIDATE

> “Both paths converge at validation.”
> 
> “If deterministic extraction fails, we retry through the model path once.”
> 
> “If the second attempt fails, we send the document to manual review.”

---

## Step 7 — RESOLVE AND AGGREGATE

> “Validated information then moves into entity resolution and cross-document aggregation.”

---

## SERVE

> “And finally, the information becomes available through SQL views, the Semantic View, Cortex Search, and ultimately Snowflake CoWork.”

### Transition to cost slide:

> “The other important question for any production AI platform is: what does this architecture cost to operate?”

---

# 🟦 SLIDE 6 — COST COMPARISON

This slide compares the previous scheduled/LLM-driven architecture with the current optimized architecture and presents the deck's benchmark figures.

### Say:

> “This slide explains why the architecture matters from a cost perspective.”
> 
> “The previous architecture was more heavily scheduled and LLM-driven.”
> 
> “That creates a straightforward but expensive pattern: process more documents through AI and use compute more broadly.”

Point to current architecture:

> “Our current architecture introduces intelligence into the routing layer.”
> 
> “Predictable documents stay on the deterministic path.”
> 
> “Only documents requiring additional intelligence use the model path.”

Point to 75%:

> “So if approximately 75 percent of the documents can be handled deterministically, we avoid unnecessary model consumption on those documents.”

Point to cost:

> “The deck uses the current benchmark of approximately **$0.00007 per document** for the Snowflake IDP platform.”

### Important qualification for client:

> “These numbers should be treated as benchmark estimates based on the current workload assumptions and configuration; actual production cost will vary with document volume, document complexity, warehouse sizing, AI usage, and workload characteristics.”

Then:

> “The key takeaway is not simply that the platform is cheaper.”
> 
> **“The key takeaway is that the architecture gives us control over where AI is consumed.”**

### Transition:

> “With that architecture established, let's stop looking at diagrams and actually run it.”

---

# 🔴 TRANSITION TO LIVE DEMO

This transition is VERY important.

Don't immediately switch screens.

Say:

> “So far, we've shown you the architecture and the design decisions.”
> 
> “Now I'd like to demonstrate that architecture as an actual running platform.”
> 
> “I'm going to upload real PDF documents into the Streamlit application.”
> 
> “From there, we'll start the processing pipeline and watch the documents move through the stages we just discussed.”
> 
> “I'll intentionally include a document that demonstrates the model escalation path, so we can see both sides of the architecture.”
> 
> “And once the processing is complete, we'll take the resulting data into Snowflake CoWork and ask a natural-language question against it.”

**Then switch to Streamlit.**

---

# 🟦 LIVE STREAMLIT DEMO SCRIPT

## 1. OPEN DASHBOARD

Say:

> “This is our operational control center for the IDP platform.”

Point to header:

> “We're running in the IDP_DEV environment on Snowflake.”

Point to KPIs:

> “At the top we have the operational KPIs — documents uploaded, processed, currently running, failures, and AI/model usage.”

Then:

> “The main component here is the live processing pipeline.”

---

# 2. EXPLAIN PIPELINE BEFORE UPLOAD

Point left-to-right:

> “We have Landing, Register, Parse, Route, Extraction, Validation, Entity Resolution, Serve, and finally consumption.”

Then:

> “The important thing is that this isn't just a diagram.”
> 
> **“These indicators are connected to the actual processing state.”**

---

# 3. UPLOAD DOCUMENTS

Click Upload.

Say:

> “I'm going to upload a small set of documents — for example an invoice, a purchase order, and a contract.”

Upload them.

After upload:

> “The files have now entered our RAW_INBOUND landing area.”

Point to dashboard.

> “You can see the documents appearing in the platform.”

---

# 4. START PROCESSING

Click:

**Start Processing**

Say:

> “Now I'm starting the processing flow.”

Pause.

> “From here, we're going to let the platform do the work.”

---

# 5. LANDING

As progress moves:

> “The first stage is Landing.”
> 
> “The document has arrived and been detected.”

---

# 6. REGISTER

> “Next is T10 Register.”
> 
> “This is our SQL-based registration and deduplication step.”
> 
> “No LLM is required here.”

---

# 7. PARSE

When Parse activates:

> “Now we're entering T20 Parse.”
> 
> “This is where we use Snowflake AI_PARSE_DOCUMENT.”
> 
> “The objective here is to convert the document into usable text.”
> 
> “This is one of our genuine AI-required stages.”

---

# 8. ROUTE

When routing begins:

> “Now the router evaluates the parsed document.”

Then explain document by document:

> “The invoice matches our configured invoice template.”
> 
> “The purchase order matches the purchase-order template.”
> 
> “And this contract is less predictable, so we're going to demonstrate the model path.”

Then:

> “This is where our deterministic-first architecture becomes visible.”

---

# 9. DETERMINISTIC DOCUMENT

When invoice enters deterministic lane:

> “The invoice has entered T40A.”
> 
> “We're extracting the configured fields using the deterministic rules.”
> 
> “Because this is a predictable document, we don't need an LLM.”

Then:

> “So this extraction is fast, deterministic, and has zero LLM extraction cost.”

---

# 10. MODEL DOCUMENT

When contract enters model lane:

> “Now this contract has entered T40B.”
> 
> “The deterministic rules weren't sufficient for this document, so we're escalating it to Cortex AI.”

Pause.

> “This is exactly where we want the model to be used.”
> 
> “Not everywhere — only where deterministic processing isn't enough.”

---

# 11. VALIDATION

When validation appears:

> “Now both extraction paths converge into T50 Validate.”

Then:

> “We're checking required fields, formats, ranges, and arithmetic relationships.”

If invoice passes:

> “The invoice has passed validation.”

If contract fails first attempt:

> “Here we have an interesting case.”
> 
> “The deterministic or first extraction attempt did not satisfy validation.”
> 
> “Rather than immediately sending this to manual review, our platform escalates it to the model lane for one retry.”

---

# 12. MODEL RETRY

Say:

> “This is the escalation mechanism we discussed in the architecture.”

Point at movement:

> **“Validation failure → Model retry → Validation again.”**

When it passes:

> “And now the second attempt has passed validation.”

Then:

> “So the human is only involved when the automated processing genuinely cannot resolve the document.”

---

# 13. ENTITY RESOLUTION

When Gold becomes active:

> “Now we're moving into the Gold layer.”

Point to entity resolution.

> “Here we resolve references to the same business entity across documents.”

Use your Testco/Zephyr example if the actual demo data contains it.

> “For example, if one document says ‘Testco Industries’ and another uses a slightly different representation, the platform can associate them with the same canonical entity.”

---

# 14. SERVE

When Serve completes:

> “Now the validated and enriched information is available through our serving layer.”

Point to:

- SQL views
    
- Semantic View
    
- Cortex Search
    

> “This is where the processed information becomes consumable by downstream users and applications.”

---

# 15. FINAL PIPELINE STATE

When complete:

> “And now we have completed the end-to-end pipeline.”

Pause for a second.

Then:

> “So what we just saw was not a conceptual architecture.”
> 
> **“We started with a PDF, registered it, parsed it, routed it, extracted information using the appropriate processing path, validated the result, resolved entities, and published the resulting information.”**

---

# 🟣 TRANSITION TO SNOWFLAKE COWORK

This should be a strong transition.

Say:

> “But extracting the information is only half of the problem.”
> 
> “The real business value comes when users can actually ask questions about that information without having to understand the underlying tables.”
> 
> “So let's now move to the consumption layer.”

Switch to Snowflake CoWork.

---

# 🟦 SNOWFLAKE COWORK DEMO

## OPENING

Say:

> “Here we're using the Semantic View we created for the IDP platform.”
> 
> “This gives CoWork the business context required to understand the structured information.”

Then:

> “Instead of writing SQL manually, I'm going to ask a natural-language question.”

---

# QUESTION 1 — CONTRACT

Ask:

> **“What is the contract value for CTR-2025-C10?”**

Wait.

Then explain:

> “CoWork understands the question through the semantic layer.”
> 
> “It retrieves the relevant structured information and returns the answer.”

Point to source.

> “And importantly, we're not only interested in the answer.”
> 
> **“We also want the answer to be grounded in the underlying IDP data.”**

---

# QUESTION 2 — SUPPLIER

Ask:

> **“Which invoices are from Nimbus Dynamics?”**

Then:

> “This is a different type of question.”
> 
> “We're now querying across the extracted invoice information rather than looking for a specific contract.”

---

# QUESTION 3 — OPERATIONAL

Then ask:

> **“How many documents were processed today?”**

Say:

> “And this demonstrates that the same consumption layer can also answer operational questions against our structured platform data.”

---

# OPTIONAL — CONTENT QUESTION

If Cortex Search is available and working in your environment:

Ask:

> **“What does the termination clause say in the contract CTR-2025-C10?”**

Then:

> “This is where we move from structured analytics into document-content search.”
> 
> “The user doesn't need to know which system or index contains that information.”
> 
> “The platform routes the question to the appropriate capability.”

---

# EXPLAIN THE AGENT ROUTING

Point to the agent architecture.

Say:

> “Conceptually, our consumption layer has three routing patterns.”

```text
Numeric question
       ↓
Cortex Analyst

Document/content question
       ↓
Cortex Search

Question requiring both
       ↓
Reconcile
```

Then:

> “So the user simply asks the question.”
> 
> “The underlying platform determines which capability is appropriate.”

---

# IMPORTANT — GROUNDING

Say:

> “Another important design principle is grounding.”
> 
> “The assistant should answer from the information available in the IDP platform.”
> 
> “Where source information is available, we want the response to reference the underlying document or document ID.”
> 
> “And where the required information is not available, the assistant should not manufacture an answer.”

Strong line:

> **“We prefer a grounded ‘I don't know’ over an ungrounded answer.”**

That is an excellent client-facing statement.

---

# 🔵 FINAL TECHNICAL SUMMARY BEFORE THANK YOU

After CoWork, return briefly to the Streamlit dashboard or architecture slide.

Say:

> “So if we step back and look at what we've demonstrated, there are really four things working together.”

### 1.

> “First, Snowflake provides the data and processing foundation.”

### 2.

> “Second, the IDP pipeline handles the complete document lifecycle.”

### 3.

> “Third, the deterministic-first architecture controls AI consumption.”

### 4.

> “And fourth, the Semantic View, Search, and CoWork layer turns the processed information into something business users can actually interact with.”

Then:

> **“So the journey is not simply document extraction.”**

Pause.

> **“It is document → trusted data → intelligence → answer.”**

---

# 🟦 RETURN TO SLIDE 7 — THANK YOU

The final slide closes with the platform message around turning unstructured information into intelligent answers.

### Say:

> “And that brings us to the end of the demonstration.”
> 
> “What we wanted to show today is how we can take unstructured documents and build a complete Snowflake-native processing and intelligence platform around them.”
> 
> “The platform handles ingestion, parsing, routing, extraction, validation, entity resolution, aggregation, search, and natural-language interaction.”
> 
> “And importantly, AI is introduced where it adds value rather than being applied indiscriminately.”

Then:

> **“The result is a governed, observable, and cost-conscious IDP platform that can turn unstructured information into trusted business answers.”**

Pause.

> **“Thank you.”**

Then:

> “We're happy to take any questions.”

---

# 🎯 THE 5 KEY LINES TO MEMORIZE

If you don't want to memorize the entire script, memorize these five lines. They will hold the whole presentation together.

### Opening

> **“We take unstructured documents, turn them into trusted structured information, and make that information available for search, analytics, and natural-language interaction.”**

### Architecture

> **“Our architecture is deterministic-first and AI-when-required.”**

### AI strategy

> **“We don't send every document through an LLM; we use AI where deterministic processing cannot reliably solve the problem.”**

### Demo

> **“What you're seeing here is the actual document lifecycle — from file arrival through processing, validation, resolution, and serving.”**

### CoWork

> **“The final objective is not simply extracting data; it is making that data available as grounded business answers.”**

---

# ⏱️ RECOMMENDED RECORDING TIMELINE

For a polished **20–25 minute client demo**:

|Section|Time|
|---|--:|
|Opening|1 min|
|Slide 1|1.5 min|
|Slide 2|2 min|
|Slide 3|4 min|
|Slide 4|2 min|
|Slide 5|2.5 min|
|Slide 6|2 min|
|**Streamlit Demo**|**6–7 min**|
|**CoWork Demo**|**3 min**|
|Closing|1 min|
|**Total**|**~25 min**|

For a shorter recording, compress Slides 2–5 but **do not compress the live demo too aggressively**. The live end-to-end execution is what proves that the architecture shown in the slides actually exists.

---

# 🔥 THE STORY YOU ARE TELLING

The entire presentation should ultimately feel like **one story**, not seven disconnected slides:

```text
                    THE PROBLEM
                        │
                        ▼
              Unstructured Documents
                        │
                        ▼
                 SNOWFLAKE IDP
                        │
              ┌─────────┴─────────┐
              │                   │
       PREDICTABLE             EXCEPTION
              │                   │
              ▼                   ▼
       DETERMINISTIC             AI
          ~75%                   ~25%
              │                   │
              └─────────┬─────────┘
                        ▼
                    VALIDATE
                        │
                   ┌────┴────┐
                   │         │
                  PASS      FAIL
                   │         │
                   │      MODEL RETRY
                   │         │
                   └────┬────┘
                        ▼
                ENTITY RESOLUTION
                        │
                        ▼
                   AGGREGATION
                        │
                        ▼
                SERVING LAYER
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
          ANALYTICS          CORTEX / CoWork
                                  │
                                  ▼
                          GROUNDED ANSWER
```

And your **live demo should literally execute this story**.

That is what will make the presentation feel cohesive: **the architecture slides explain what the platform is, Streamlit proves that it runs, and CoWork proves why the processed information is useful.**