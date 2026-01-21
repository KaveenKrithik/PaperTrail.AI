# PaperTrail.AI — Technical & Agentic Deep Dive

## 1) What PaperTrail.AI Is
PaperTrail.AI is a document verification and audit platform that turns unstructured business documents (contracts, invoices, financial statements, KYC forms, purchase orders) into **verified, evidence-backed findings**.

Instead of only extracting text or summarizing, PaperTrail.AI focuses on:
- **Verifying claims** inside documents (numbers, dates, parties, obligations).
- **Checking consistency** (totals vs line-items, repeated values across sections, signatures vs required signatories).
- **Checking compliance** against rules (organization policy, domain rules, or regulatory constraints).
- **Producing an audit report** with a **risk score** and **evidence links** that a human can validate.

The “product identity” is: **evidence-first auditing**, where every flagged issue is traceable to a source snippet and (optionally) a page coordinate.

---

## 2) Why This Product Matters (Problem + Impact)

### 2.1 The Current Pain
Many teams still review documents manually because:
- OCR and extraction tools provide data, but not verification.
- Generic AI summaries are not reliable enough for compliance decisions.
- Traditional review produces inconsistent outcomes across reviewers.

Common failure modes:
- Invoice total doesn’t match line items (or includes hidden fees).
- Dates, names, or amounts differ between sections.
- Contract clauses conflict (e.g., termination vs renewal terms).
- Missing signatures or mismatched authority.
- Policy violations (e.g., spending approvals missing).

### 2.2 The Business Value
PaperTrail.AI targets measurable improvements:
- **Time**: faster review cycles for routine documents.
- **Quality**: fewer missed inconsistencies; more uniform reviews.
- **Traceability**: audit logs and evidence trails that are defensible.
- **Scalability**: consistent verification even as volume grows.

---

## 3) How It Works (High-Level Workflow)

### 3.1 User Journey
1. User uploads a document (PDF/DOCX).
2. System extracts text (OCR if needed) + basic layout/page mapping.
3. System extracts structured entities (parties, dates, amounts, line items).
4. System runs verification steps:
   - consistency checks
   - compliance checks
   - risk scoring
5. System generates a report:
   - findings (issue, severity, rationale)
   - evidence references (source excerpt, page, coordinates)
   - recommended actions
6. User can dispute a finding:
   - a different reasoning strategy (“counter-auditor”) reevaluates the specific clause.

### 3.2 Product Principle: Evidence-First
Every output should answer: **“Where did you get that?”**
- Text evidence: a quoted snippet + page number.
- Visual evidence: bounding boxes on the page.
- Trace evidence: which step produced it and what input was used.

---

## 4) Competitive Landscape (Where It Fits)
PaperTrail.AI sits between OCR/extraction tools and contract lifecycle management systems.

### 4.1 Comparable Categories

**A) OCR / Data Extraction Tools**
- Strength: extract fields quickly.
- Gap: limited reasoning about whether extracted values are *correct* or *consistent*.
- PaperTrail.AI difference: verification logic + evidence-linked audit trail.

**B) Contract Lifecycle Management (CLM) Platforms**
- Strength: workflows, approvals, repositories.
- Gap: expensive/heavy adoption; verification often not the primary product core.
- PaperTrail.AI difference: audit-grade verification as the core, designed for evidence.

**C) AI Assistants / Summarizers**
- Strength: summarize, answer questions.
- Gap: not deterministic, inconsistent, and often lacks traceability.
- PaperTrail.AI difference: structured steps, explicit evidence mapping, and a dispute mechanism.

### 4.2 Differentiation Summary
- **Skeptical auditing pipeline**: default stance is to validate, not assume.
- **Counter-auditor disputes**: reduces one-model bias and improves trust.
- **Evidence mapping**: every flag links to the exact source.
- **Multi-document verification** (roadmap): invoice vs contract mismatch detection.

---

## 5) Architecture Overview (Opinionated System Design)

### 5.1 Core Modules
1. **Ingestion**
   - File upload, storage, metadata.
2. **Understanding**
   - OCR/text extraction and normalization.
   - Structured entity extraction.
3. **Verification**
   - Consistency checks.
   - Compliance checks.
   - Risk scoring.
4. **Evidence**
   - Text spans + page coordinates + linkable references.
5. **Reporting**
   - Human-readable report + export.
6. **Governance**
   - Audit logs, RBAC, retention.

### 5.2 Data Model (Supabase / Postgres)
Suggested tables (aligned with your existing concept):
- `documents`: id, filename, storage_path, mime_type, owner_id, created_at
- `audits`: id, document_id, status, risk_score, summary, created_at
- `audit_steps`: id, audit_id, step_name, status, input_json, output_json, started_at, ended_at
- `findings`: id, audit_id, severity, category, title, description, evidence_json
- `disputes`: id, finding_id, user_id, dispute_text, counter_output_json, created_at
- `compliance_rules`: id, org_id, rule_name, rule_nl, rule_dsl, enabled
- `audit_embeddings`: id, org_id, embedding vector, metadata_json

Key idea: **store intermediate step outputs** so the system is explainable and debuggable.

---

## 6) Agentic Design (Multi-Agent Pipeline)

### 6.1 Why Agents (Not One Prompt)
A single “summarize and check everything” prompt is hard to trust because:
- It mixes extraction, verification, and scoring in one opaque output.
- It’s difficult to trace which evidence caused which finding.
- It’s harder to isolate failures.

A pipeline makes the system:
- **modular** (each agent has a bounded responsibility)
- **auditable** (step-level logs)
- **improvable** (swap an agent without rewriting everything)

### 6.2 Recommended Agents

**Agent A: Document Preprocessor**
- Input: file
- Output: text per page + layout map
- Responsibilities:
  - OCR (if scanned)
  - cleanup (remove headers/footers noise)
  - page segmentation

**Agent B: Structured Extractor**
- Output schema (example):
  - parties: [{name, role, identifiers?}]
  - dates: {effective_date, due_date, termination_date}
  - amounts: {total, currency, taxes, discounts}
  - line_items: [{description, qty, unit_price, total}]

**Agent C: Consistency Verifier**
- Checks:
  - numeric reconciliation (line items sum == total)
  - repeated entity match (party name consistent)
  - date logic (due date after invoice date)

**Agent D: Compliance Verifier**
- Executes rules in a deterministic DSL:
  - Example: `IF total > 50000 AND signature_count < 2 THEN FLAG`

**Agent E: Risk Scorer**
- Input: findings + severities + confidence
- Output: risk score 0–100 + justification

**Agent F: Counter-Auditor (Dispute Agent)**
- Trigger: user disputes a specific finding
- Output: confirm/overturn + rationale + evidence
- Design: use different prompt strategy (more adversarial) and optionally different model provider.

### 6.3 Agent Output Contracts
To keep the system stable:
- Each agent must output JSON that matches a schema.
- Store raw outputs + validated outputs.
- If schema validation fails, mark the step as failed and generate a recoverable error.

### 6.4 Provider Strategy
Use multiple providers with a routing policy:
- Fast model for extraction.
- More capable model for dispute or complex reasoning.
- Fallback provider for reliability.

Keep provider choice as metadata in `audit_steps` so results are reproducible.

---

## 7) Tech Stack (Deep Detail)

### 7.1 Frontend: Next.js 14 (App Router) + TypeScript
Why:
- Server Components and Server Actions reduce API boilerplate.
- Strong DX for building dashboards.
- Type safety for data models and agent contracts.

Key UI pages:
- Auth (login/signup)
- Document upload
- Audit run view (status + steps)
- Findings list + evidence viewer
- Dispute flow
- Rules editor
- Audit history

### 7.2 Styling: Tailwind CSS + (Optional) Framer Motion
- Tailwind for consistent, fast UI development.
- Framer Motion to improve clarity in step-by-step pipeline status (progress, transitions).

### 7.3 Backend: Next.js Server Actions + Route Handlers
Two patterns:
- **Server Actions** for authenticated, form-based operations (upload, create rule, dispute finding).
- **Route Handlers** for webhooks or file streaming endpoints.

Important: keep large file uploads compatible with your hosting approach (Supabase signed uploads is a common option).

### 7.4 Database & Auth: Supabase (Postgres + Auth)
Why:
- Built-in auth, RLS (Row Level Security), storage.
- Postgres gives transactional integrity for audit trails.

Recommended security approach:
- Enable RLS on tables and write policies (org/user scoped).
- Store minimal PII.
- Maintain immutable audit logs.

### 7.5 Storage: Supabase Storage
- Store raw files + derived artifacts.
- Store per-page images (if you render PDF pages for overlays).

### 7.6 Vector Search: pgvector (Supabase)
Purpose:
- “Audit memory” to find similar past cases and improve consistency.

Examples:
- Similar clause patterns across contracts.
- Previously accepted dispute decisions.
- Known fraud signature patterns.

### 7.7 OCR and Parsing
Options depend on constraints:
- PDF text extraction (if text-based PDF)
- OCR for scanned PDFs (external service or self-hosted)

Keep the OCR output **page-aware** so evidence mapping remains accurate.

### 7.8 AI Provider Layer (Gemini / Groq / OpenAI fallback)
Use a provider abstraction:
- Common interface: `generateText({model, prompt, schema})`
- Central place for:
  - retry logic
  - timeouts
  - cost tracking
  - structured JSON enforcement

### 7.9 Structured Output Enforcement
Critical for product reliability:
- Define JSON schema per agent.
- Validate and reject malformed responses.
- On failure: either retry with stricter prompt or fallback model.

---

## 8) End-to-End Audit Flow (Step-by-Step)

### 8.1 Sequence (Conceptual)
1. Create `document` row.
2. Create `audit` row with status `queued`.
3. Run step: `preprocess` → store in `audit_steps`.
4. Run step: `extract_entities` → store.
5. Run step: `consistency_checks` → produce findings.
6. Run step: `compliance_checks` → produce findings.
7. Run step: `risk_score` → store final score.
8. Generate report summary and mark `audit` as `completed`.

### 8.2 Finding Structure (Recommended)
Each finding should include:
- title
- category (Consistency / Compliance / Fraud / Missing Data)
- severity (Low/Medium/High/Critical)
- confidence (0–1)
- description (human-readable)
- evidence:
  - page
  - quoted text
  - coordinates (optional)
  - extracted fields involved

---

## 9) Reliability, Testing, and Evaluation
A “solid” product requires measurable quality.

### 9.1 Build a Test Corpus
Collect a small dataset:
- 20–50 sample documents across categories (invoice, contract, report)
- include “known-bad” cases (wrong totals, missing signatures, conflicting dates)

### 9.2 Define Metrics
- Extraction accuracy (field-level)
- Consistency detection precision/recall
- Compliance rule correctness
- Latency (end-to-end)
- Dispute overturn rate (signal for false positives)

### 9.3 Determinism Strategy
- Use deterministic rule checks where possible.
- Use AI primarily for extraction and interpretation, but validate outputs.
- Keep step logs for reproducibility.

---

## 10) Security & Privacy
Because documents are sensitive:
- Encrypt storage (at rest) + signed URLs.
- Strict RLS in Supabase.
- Minimal data retention; optional deletion.
- Audit logs are append-only.
- Avoid sending documents to external providers unless necessary; send only relevant excerpts when possible.

---

## 11) Roadmap (Product Logic)

### MVP (Single-Document Trust)
- Upload + OCR
- Structured extraction
- Consistency checks
- Rules engine (basic)
- Evidence-linked report

### Differentiation
- Counter-auditor disputes
- Visual overlays
- Multi-document verification

### Scale
- Email ingestion
- Organization policies at scale
- Monitoring + cost controls

---

## 12) What We Will Build Next (Concrete Milestones)
If you want a stable demo quickly, implement in this order:
1. Document ingestion + storage + audit record creation
2. Page-aware text extraction
3. Structured entity extraction + JSON schema validation
4. Consistency checks (numeric + entity)
5. Report page with evidence snippets
6. Compliance rules CRUD + simple rule execution
7. Risk scoring
8. Dispute system (counter-auditor)

This ordering produces a working product early and adds trust features incrementally.
