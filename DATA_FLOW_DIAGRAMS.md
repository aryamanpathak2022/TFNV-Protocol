# TFNV-Protocol: Data Flow Diagrams

## Table of Contents
1. [Complete System Data Flow](#complete-system-data-flow)
2. [Trust Layer Communication Flow](#trust-layer-communication-flow)
3. [GenAI RAG Processing Flow](#genai-rag-processing-flow)
4. [SMART on FHIR Integration Flow](#smart-on-fhir-integration-flow)
5. [Gamification & Donation Flow](#gamification--donation-flow)
6. [Security & Audit Flow](#security--audit-flow)
7. [Notification Delivery Flow](#notification-delivery-flow)

---

## Complete System Data Flow

### End-to-End Adverse Event Submission Journey

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PATIENT JOURNEY START                            │
│                   (Mobile App / Web Browser)                            │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ 1. User clicks "Report Adverse Event"
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      AUTHENTICATION LAYER                                │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                   Keycloak (OAuth 2.0)                           │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │  1. User login (email/password + MFA)                           │  │
│  │  2. Issue JWT access token (15 min expiry)                      │  │
│  │  3. Issue refresh token (7 day expiry)                          │  │
│  │  4. Establish session                                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ JWT Token: {"sub": "user123", "role": "patient"}
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       TRUST VERIFICATION                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                   Trust Service (NestJS)                         │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │  IF user prefers email communication:                           │  │
│  │    ✓ Verify sender domain (SPF/DKIM/DMARC)                     │  │
│  │    ✓ Check VMC certificate validity                            │  │
│  │    ✓ Confirm BIMI logo availability                            │  │
│  │  IF user prefers WhatsApp:                                      │  │
│  │    ✓ Verify WhatsApp Business Account                          │  │
│  │    ✓ Confirm Green Tick status                                 │  │
│  │    ✓ Rate limit check (1000 msg/sec)                           │  │
│  │  RESULT: Trust Score = 100% ✅                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Trust verified, proceed to data gathering
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    EHR DATA PRE-FILLING (Optional)                      │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                SMART on FHIR Service (NestJS)                    │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │  IF user has EHR linked:                                        │  │
│  │                                                                  │  │
│  │  1. SMART App Launch                                            │  │
│  │     ├─> Redirect to EHR OAuth server                            │  │
│  │     ├─> User authorizes data access                             │  │
│  │     └─> Receive authorization code                              │  │
│  │                                                                  │  │
│  │  2. Exchange code for access token                              │  │
│  │     └─> Token includes patient context (patient ID)             │  │
│  │                                                                  │  │
│  │  3. Fetch FHIR Resources:                                       │  │
│  │     ├─> GET /Patient/{id}                                       │  │
│  │     │   └─> Demographics: Name, DOB, Gender, MRN               │  │
│  │     ├─> GET /MedicationStatement?patient={id}                   │  │
│  │     │   └─> Current medications, dosages, start dates          │  │
│  │     ├─> GET /AllergyIntolerance?patient={id}                    │  │
│  │     │   └─> Known allergies and reactions                      │  │
│  │     └─> GET /Condition?patient={id}                             │  │
│  │         └─> Medical history, diagnoses                          │  │
│  │                                                                  │  │
│  │  4. Map FHIR data to AE report form fields                      │  │
│  │     └─> Auto-populate: Patient info, meds, history             │  │
│  │                                                                  │  │
│  │  ELSE (no EHR linked):                                          │  │
│  │     └─> Show blank form for manual entry                        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Pre-filled form data (or blank)
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              CONVERSATIONAL TRIAGE (GenAI RAG)                          │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                 GenAI RAG Service (Python + FastAPI)             │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │                                                                  │  │
│  │  USER: "I took ibuprofen and got a severe rash"                 │  │
│  │                                                                  │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │ STEP 1: Medical Entity Recognition (scispaCy)             │ │  │
│  │  ├────────────────────────────────────────────────────────────┤ │  │
│  │  │  Extract:                                                  │ │  │
│  │  │  • Drug: "ibuprofen" (RxNorm: 5640)                       │ │  │
│  │  │  • Symptom: "rash" (SNOMED: 271807003)                    │ │  │
│  │  │  • Severity modifier: "severe"                            │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                         ↓                                        │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │ STEP 2: Drug Knowledge Query (DrugBank API)               │ │  │
│  │  ├────────────────────────────────────────────────────────────┤ │  │
│  │  │  Query: "ibuprofen"                                        │ │  │
│  │  │  Returns:                                                  │ │  │
│  │  │  • Drug class: NSAID                                       │ │  │
│  │  │  • Known adverse effects: Rash (common), GI bleed, ...    │ │  │
│  │  │  • Contraindications: Aspirin allergy                     │ │  │
│  │  │  • Drug interactions: Warfarin, lithium                   │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                         ↓                                        │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │ STEP 3: Vector Search (Pinecone)                          │ │  │
│  │  ├────────────────────────────────────────────────────────────┤ │  │
│  │  │  Query embedding: "ibuprofen severe rash"                 │ │  │
│  │  │  Top 5 relevant documents:                                │ │  │
│  │  │  1. FDA Guideline: "Skin reactions to NSAIDs" (score: 0.92)│ │  │
│  │  │  2. Case study: "Ibuprofen-induced rash" (score: 0.89)   │ │  │
│  │  │  3. ICH E2B: "Classifying skin AEs" (score: 0.85)        │ │  │
│  │  │  4. MedDRA term: "Rash maculo-papular" (score: 0.81)     │ │  │
│  │  │  5. Similar case #1234 (score: 0.78)                     │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                         ↓                                        │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │ STEP 4: LLM Reasoning (GPT-4o)                            │ │  │
│  │  ├────────────────────────────────────────────────────────────┤ │  │
│  │  │  Prompt:                                                   │ │  │
│  │  │  ─────────────────────────────────────────────────────────  │ │  │
│  │  │  You are an FDA-compliant pharmacovigilance assistant.   │ │  │
│  │  │                                                            │ │  │
│  │  │  Context: {retrieved_docs from Pinecone}                  │ │  │
│  │  │  Drug info: {DrugBank data}                               │ │  │
│  │  │  Patient input: "I took ibuprofen and got severe rash"   │ │  │
│  │  │                                                            │ │  │
│  │  │  Tasks:                                                    │ │  │
│  │  │  1. Assess severity (ICH E2B criteria)                    │ │  │
│  │  │  2. Identify missing critical info                        │ │  │
│  │  │  3. Generate follow-up questions                          │ │  │
│  │  │  4. Be empathetic                                         │ │  │
│  │  │  ─────────────────────────────────────────────────────────  │ │  │
│  │  │                                                            │ │  │
│  │  │  LLM Response:                                             │ │  │
│  │  │  • Severity: MODERATE (skin reaction, no systemic)       │ │  │
│  │  │  • Missing: Timing, duration, medical intervention       │ │  │
│  │  │  • Next questions: When started? Spread? Sought care?    │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                         ↓                                        │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │ STEP 5: Guardrails Check (NeMo Guardrails)               │ │  │
│  │  ├────────────────────────────────────────────────────────────┤ │  │
│  │  │  ✓ No medical advice given                                │ │  │
│  │  │  ✓ No diagnosis attempted                                 │ │  │
│  │  │  ✓ FDA guidelines referenced correctly                    │ │  │
│  │  │  ✓ No hallucinated drug names                             │ │  │
│  │  │  ✓ Appropriate tone (empathetic)                          │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                         ↓                                        │  │
│  │  AI: "Thank you for reporting. Rash is a known side effect   │  │
│  │       of ibuprofen. To help assess severity:                  │  │
│  │       1. When did the rash first appear?                      │  │
│  │       2. Has it spread to other areas?                        │  │
│  │       3. Did you seek medical care?"                          │  │
│  │                                                                  │  │
│  │  [User answers questions over 2-3 turns]                        │  │
│  │                                                                  │  │
│  │  RESULT: Complete triage data collected ✅                      │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Triage complete, generate narrative
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   AUTO-NARRATIVE GENERATION                              │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              GenAI Narrative Generator (GPT-4o)                  │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │                                                                  │  │
│  │  Input: Structured AE data from triage                          │  │
│  │  ────────────────────────────────────────────────────────────    │  │
│  │  {                                                               │  │
│  │    "patient": {age: 45, gender: "M", weight: 80},               │  │
│  │    "suspect_drugs": [{name: "ibuprofen", dose: "400mg", ...}],  │  │
│  │    "event": {description: "rash", onset: "3 days after", ...}   │  │
│  │  }                                                               │  │
│  │                                                                  │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │  Generate ICH E2B(R3) Compliant Narrative                 │ │  │
│  │  ├────────────────────────────────────────────────────────────┤ │  │
│  │  │  "A 45-year-old male patient weighing 80 kg initiated    │ │  │
│  │  │   treatment with ibuprofen 400 mg orally three times     │ │  │
│  │  │   daily for chronic back pain. Approximately 3 days      │ │  │
│  │  │   after starting ibuprofen, the patient developed a      │ │  │
│  │  │   generalized erythematous maculopapular rash on the     │ │  │
│  │  │   trunk and extremities..."                              │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                                                                  │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │  MedDRA Auto-Coding                                       │ │  │
│  │  ├────────────────────────────────────────────────────────────┤ │  │
│  │  │  PT (Preferred Term): "Rash maculo-papular" (10037868)   │ │  │
│  │  │  LLT (Lower Level Term): "Maculopapular rash"            │ │  │
│  │  │  SOC (System Organ Class): "Skin and subcutaneous..."    │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                                                                  │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │  Naranjo Causality Assessment                             │ │  │
│  │  ├────────────────────────────────────────────────────────────┤ │  │
│  │  │  Questions:                                                │ │  │
│  │  │  1. Adverse event appeared after drug? +2                 │ │  │
│  │  │  2. Improved when drug stopped? +1                        │ │  │
│  │  │  3. Reappeared when drug readministered? 0 (N/A)          │ │  │
│  │  │  ... (10 questions total)                                 │ │  │
│  │  │                                                            │ │  │
│  │  │  Total Score: 6 → Causality: PROBABLE                     │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                                                                  │  │
│  │  RESULT: Complete draft AE report ✅                            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Draft report ready for review
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    HUMAN VALIDATION LAYER                                │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │         Pharmacovigilance Specialist Review Portal              │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │                                                                  │  │
│  │  Queue: Draft reports awaiting review                           │  │
│  │  ├─ Report #12345 (Priority: Moderate)                          │  │
│  │  │   ├─ AI Narrative: [Show full narrative]                     │  │
│  │  │   ├─ MedDRA Codes: [List codes]                              │  │
│  │  │   ├─ Causality: Probable (Naranjo: 6)                        │  │
│  │  │   └─ Confidence Score: 92%                                   │  │
│  │  │                                                               │  │
│  │  Specialist Actions:                                             │  │
│  │  [ ] Approve as-is                                               │  │
│  │  [ ] Request changes (with comments)                             │  │
│  │  [ ] Reject (with reason)                                        │  │
│  │                                                                  │  │
│  │  ✓ Specialist APPROVES with digital signature                   │  │
│  │                                                                  │  │
│  │  Audit Trail:                                                    │  │
│  │  • Created by: AI System (2026-01-08 10:30 AM)                  │  │
│  │  • Reviewed by: Dr. Jane Smith (2026-01-08 2:45 PM)             │  │
│  │  • Approved by: Dr. Jane Smith (2026-01-08 2:47 PM)             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Report approved, proceed to submission
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    REGULATORY SUBMISSION                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              AE Report Service (NestJS)                          │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │                                                                  │  │
│  │  1. Format report for FDA MedWatch (E2B XML)                     │  │
│  │  2. Digitally sign with X.509 certificate                        │  │
│  │  3. Submit to FDA Gateway:                                       │  │
│  │     POST https://www.accessdata.fda.gov/scripts/medwatch/submit  │  │
│  │  4. Receive acknowledgment:                                      │  │
│  │     FDA Tracking #: MW-2026-00012345                             │  │
│  │  5. Update report status: FDA_SUBMITTED ✅                       │  │
│  │                                                                  │  │
│  │  6. Write back to EHR (FHIR AdverseEvent):                       │  │
│  │     POST {fhir_server}/AdverseEvent                              │  │
│  │     {                                                            │  │
│  │       "resourceType": "AdverseEvent",                            │  │
│  │       "subject": {"reference": "Patient/123"},                   │  │
│  │       "event": {"text": "Rash maculo-papular"},                  │  │
│  │       "suspectEntity": [{"instance": "Medication/ibuprofen"}],   │  │
│  │       ...                                                        │  │
│  │     }                                                            │  │
│  │  7. FHIR server confirms: AdverseEvent/ae-789 created ✅         │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Submission complete, reward user
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                  GAMIFICATION & REWARDS                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │            Gamification Service (NestJS)                         │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │                                                                  │  │
│  │  1. Award Points:                                                │  │
│  │     ├─ Report submission: +100 points                            │  │
│  │     ├─ Complete details: +50 points                              │  │
│  │     ├─ Streak bonus (7 days): +10 points                         │  │
│  │     └─ Total awarded: 160 points                                 │  │
│  │                                                                  │  │
│  │  2. Update User Profile:                                         │  │
│  │     ├─ Previous: 840 points (Level: Silver)                      │  │
│  │     ├─ New: 1000 points                                          │  │
│  │     └─ Level Up! 🎉 Bronze → Silver → GOLD                       │  │
│  │                                                                  │  │
│  │  3. Check Badges:                                                │  │
│  │     ├─ "First Reporter" ✅ (already earned)                      │  │
│  │     ├─ "Detail Detective" ✅ (already earned)                    │  │
│  │     └─ "Safety Champion" 🆕 UNLOCKED! (10 reports)               │  │
│  │                                                                  │  │
│  │  4. Update Leaderboard:                                          │  │
│  │     Redis ZADD leaderboard:global 1000 user123                  │  │
│  │     New rank: #47 (up from #62)                                  │  │
│  │                                                                  │  │
│  │  5. Offer Donation Opportunity:                                  │  │
│  │     "You have 1000 points! Convert to $10 donation?"             │  │
│  │     Charities:                                                   │  │
│  │     [ ] Doctors Without Borders                                  │  │
│  │     [ ] American Cancer Society                                  │  │
│  │     [ ] UNICEF                                                   │  │
│  │                                                                  │  │
│  │  USER SELECTS: Doctors Without Borders                           │  │
│  │                                                                  │  │
│  │  6. Process Donation:                                            │  │
│  │     ├─ Deduct 500 points (saving 500 for future)                 │  │
│  │     ├─ Stripe payment: $5.00 to DWB                              │  │
│  │     ├─ Generate tax receipt (PDF)                                │  │
│  │     └─ Update impact dashboard                                   │  │
│  │                                                                  │  │
│  │  RESULT: User rewarded and donation made! 🎊                     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Send confirmation notifications
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   NOTIFICATION DELIVERY                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │            Notification Service (NestJS + Queues)                │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │                                                                  │  │
│  │  Notification Types to Send:                                     │  │
│  │  1. AE Report Submitted ✅                                       │  │
│  │  2. FDA Acknowledgment ✅                                        │  │
│  │  3. Level Up! (Gold achieved) 🏆                                 │  │
│  │  4. Badge Unlocked! (Safety Champion) 🏅                         │  │
│  │  5. Donation Confirmed ($5 to DWB) 💝                            │  │
│  │                                                                  │  │
│  │  Delivery Channels:                                              │  │
│  │  ┌──────────────────────────────────────────────────────────┐   │  │
│  │  │  EMAIL (SendGrid with BIMI)                              │   │  │
│  │  ├──────────────────────────────────────────────────────────┤   │  │
│  │  │  To: user@example.com                                    │   │  │
│  │  │  From: noreply@tfnv-protocol.com                         │   │  │
│  │  │  Subject: "Your Adverse Event Report Was Submitted!"    │   │  │
│  │  │  Body: [HTML email with TFNV logo (BIMI)]               │   │  │
│  │  │  • Report #: MW-2026-00012345                            │   │  │
│  │  │  • Status: Submitted to FDA ✅                           │   │  │
│  │  │  • Points earned: +160                                   │   │  │
│  │  │  • New level: Gold 🏆                                    │   │  │
│  │  │  • Donation: $5 to Doctors Without Borders 💝           │   │  │
│  │  └──────────────────────────────────────────────────────────┘   │  │
│  │                                                                  │  │
│  │  ┌──────────────────────────────────────────────────────────┐   │  │
│  │  │  WHATSAPP (Meta Business API with Green Tick)           │   │  │
│  │  ├──────────────────────────────────────────────────────────┤   │  │
│  │  │  [WhatsApp Chat UI]                                      │   │  │
│  │  │  ┌────────────────────────────────────────────────────┐  │   │  │
│  │  │  │ TFNV Protocol                                      │  │   │  │
│  │  │  │ 🏢 Green Tick (Verified Business)                  │  │   │  │
│  │  │  ├────────────────────────────────────────────────────┤  │   │  │
│  │  │  │ 🎉 Great news! Your adverse event report was       │  │   │  │
│  │  │  │    successfully submitted to the FDA.              │  │   │  │
│  │  │  │                                                     │  │   │  │
│  │  │  │ 📋 Report #: MW-2026-00012345                      │  │   │  │
│  │  │  │ ✅ Status: Approved & Submitted                    │  │   │  │
│  │  │  │ 🏆 You leveled up to GOLD!                         │  │   │  │
│  │  │  │ 💝 Your $5 donation was sent to DWB                │  │   │  │
│  │  │  │                                                     │  │   │  │
│  │  │  │ Thank you for helping make medications safer!      │  │   │  │
│  │  │  └────────────────────────────────────────────────────┘  │   │  │
│  │  └──────────────────────────────────────────────────────────┘   │  │
│  │                                                                  │  │
│  │  ┌──────────────────────────────────────────────────────────┐   │  │
│  │  │  PUSH NOTIFICATION (Firebase Cloud Messaging)           │   │  │
│  │  ├──────────────────────────────────────────────────────────┤   │  │
│  │  │  [Mobile App Notification]                              │   │  │
│  │  │  🏆 Level Up!                                            │   │  │
│  │  │  You've reached Gold level! Tap to see your rewards.   │   │  │
│  │  └──────────────────────────────────────────────────────────┘   │  │
│  │                                                                  │  │
│  │  ALL NOTIFICATIONS DELIVERED ✅                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                             │
                             │ Journey complete!
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        IMPACT DASHBOARD UPDATE                           │
├─────────────────────────────────────────────────────────────────────────┤
│  • Total AE reports submitted: +1 (now 10,523)                          │
│  • Total charitable donations: +$5 (now $127,450)                       │
│  • Lives impacted: "Your reports funded 2,549 vaccine doses"            │
│  • Community leaderboard updated                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Trust Layer Communication Flow

### Email with BIMI/VMC Verification

```
┌──────────────────────────────────────────────────────────────────┐
│  TFNV Protocol Server                                            │
│  (Outbound Email System)                                         │
└────────────┬─────────────────────────────────────────────────────┘
             │
             │ 1. Compose follow-up email to patient
             │    Subject: "Follow-up on your adverse event report"
             │    Body: [HTML content with sensitive medical info]
             │
             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    Email Gateway (SendGrid)                            │
├────────────────────────────────────────────────────────────────────────┤
│  Authentication Stack:                                                 │
│  ─────────────────────────────────────────────────────────────────     │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  SPF (Sender Policy Framework)                               │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │  DNS TXT record: v=spf1 include:sendgrid.net ~all           │    │
│  │  Purpose: Verify sending server is authorized               │    │
│  │  Status: ✅ PASS                                             │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  DKIM (DomainKeys Identified Mail)                           │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │  Sign email with private key (RSA-2048)                      │    │
│  │  Add DKIM-Signature header                                   │    │
│  │  Purpose: Verify email not tampered in transit               │    │
│  │  Status: ✅ SIGNED                                            │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  DMARC (Domain-based Message Authentication)                 │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │  Policy: p=quarantine (strict enforcement)                   │    │
│  │  Alignment: SPF + DKIM must align with From: domain          │    │
│  │  Reporting: Send aggregate reports to dmarc@tfnv.com         │    │
│  │  Status: ✅ ALIGNED                                           │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  BIMI (Brand Indicators for Message Identification)          │    │
│  ├──────────────────────────────────────────────────────────────┤    │
│  │  DNS TXT: default._bimi.tfnv-protocol.com                    │    │
│  │  Logo URL: https://tfnv-protocol.com/logo.svg                │    │
│  │  VMC URL: https://tfnv-protocol.com/vmc.pem                  │    │
│  │  Purpose: Display verified brand logo in email client        │    │
│  │  Status: ✅ CONFIGURED                                        │    │
│  └──────────────────────────────────────────────────────────────┘    │
└────────────┬───────────────────────────────────────────────────────────┘
             │
             │ 2. Email sent with all authentication
             │
             ▼
┌────────────────────────────────────────────────────────────────────────┐
│              Recipient Email Provider (Gmail)                          │
├────────────────────────────────────────────────────────────────────────┤
│  Verification Process:                                                 │
│  ──────────────────────────────────────────────────────────────────    │
│                                                                        │
│  1. Check SPF record:                                                  │
│     DNS TXT lookup for tfnv-protocol.com                              │
│     ✅ Sending server matches SPF record                              │
│                                                                        │
│  2. Verify DKIM signature:                                             │
│     Extract public key from DNS                                        │
│     ✅ Signature valid, content not modified                          │
│                                                                        │
│  3. Check DMARC policy:                                                │
│     ✅ SPF aligned + DKIM aligned                                     │
│     ✅ DMARC passes                                                   │
│                                                                        │
│  4. Fetch VMC (Verified Mark Certificate):                             │
│     Download from https://tfnv-protocol.com/vmc.pem                   │
│     ✅ Certificate valid (issued by DigiCert)                         │
│     ✅ Trademark verified (USPTO)                                     │
│                                                                        │
│  5. Display BIMI logo:                                                 │
│     Download logo from https://tfnv-protocol.com/logo.svg             │
│     ✅ Logo displays next to sender name                              │
│                                                                        │
│  RESULT: Email marked as VERIFIED ✅                                   │
└────────────┬───────────────────────────────────────────────────────────┘
             │
             │ 3. Display to user
             │
             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  Gmail Inbox (User View)                               │
├────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  [📧 TFNV Logo] TFNV Protocol ✅                                 │ │
│  │  noreply@tfnv-protocol.com                                       │ │
│  │  🔒 Verified sender                                              │ │
│  │  ───────────────────────────────────────────────────────────────  │ │
│  │  Subject: Follow-up on your adverse event report                │ │
│  │  Preview: We need a few more details to complete your...        │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  User Perception:                                                      │
│  • Sees TFNV brand logo (instant recognition)                         │
│  • Sees blue checkmark (verified sender)                              │
│  • Trusts email is authentic, not phishing                            │
│  • No "scam fear" → Opens and responds ✅                             │
└────────────────────────────────────────────────────────────────────────┘
```

### WhatsApp with Green Tick

```
┌──────────────────────────────────────────────────────────────────┐
│  TFNV Protocol Server                                            │
│  (WhatsApp Integration Service)                                  │
└────────────┬─────────────────────────────────────────────────────┘
             │
             │ 1. Send follow-up message
             │    POST /messages (Meta Business API)
             │
             ▼
┌────────────────────────────────────────────────────────────────────────┐
│           WhatsApp Business API (Meta Cloud API)                       │
├────────────────────────────────────────────────────────────────────────┤
│  Authentication:                                                       │
│  ─────────────────────────────────────────────────────────────────     │
│  • Bearer token: {access_token}                                        │
│  • Phone Number ID: 123456789                                          │
│  • Business Account ID: 987654321                                      │
│  • Green Tick Status: ✅ VERIFIED                                      │
│                                                                        │
│  Message Payload:                                                      │
│  {                                                                     │
│    "messaging_product": "whatsapp",                                    │
│    "to": "14155551234",                                                │
│    "type": "text",                                                     │
│    "text": {                                                           │
│      "body": "Hello! Time for your follow-up on the adverse event     │
│               report. Please reply with: 1. Current symptoms status..."│
│    }                                                                   │
│  }                                                                     │
│                                                                        │
│  Verification Checks:                                                  │
│  ✅ Business verification complete                                     │
│  ✅ Quality rating: High                                               │
│  ✅ Rate limit: 1000 msg/sec (not exceeded)                            │
│  ✅ Template policy: Compliance checked                                │
│                                                                        │
│  Status: Message queued for delivery                                   │
└────────────┬───────────────────────────────────────────────────────────┘
             │
             │ 2. Deliver via WhatsApp network
             │
             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  WhatsApp Client (User's Phone)                        │
├────────────────────────────────────────────────────────────────────────┤
│  [Chats]  [Status]  [Calls]                                    [⚙️]    │
│  ────────────────────────────────────────────────────────────────      │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  🏥 TFNV Protocol                                            │    │
│  │  🏢 Green Tick (Verified Business Account)                   │    │
│  │  ──────────────────────────────────────────────────────────  │    │
│  │                                                              │    │
│  │  [11:30 AM]                                                  │    │
│  │  Hello! Time for your follow-up on the adverse event        │    │
│  │  report. Please reply with:                                  │    │
│  │  1. Current symptoms status                                  │    │
│  │  2. Any new medications                                      │    │
│  │                                                              │    │
│  │  Your safety is our priority. 💙                            │    │
│  │                                                              │    │
│  │  [Reply]                                                     │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  User Perception:                                                      │
│  • Sees Green Tick next to business name                               │
│  • Recognizes official verified account                                │
│  • Trusts message is from real TFNV Protocol                           │
│  • No "scam fear" → Responds confidently ✅                            │
└────────────────────────────────────────────────────────────────────────┘
```

**Trust Layer Outcome**:
- **Before**: Patients hesitate, fear scams, ignore follow-ups
- **After**: Verified channels → Trust established → 95% response rate ✅

---

## Data Residency & Compliance

### Multi-Region Data Storage

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Global User Request                               │
│           (Geo-located: US-East, EU, Asia-Pacific)                     │
└────────────────┬───────────────────────────────────────────────────────┘
                 │
                 │ Route based on user location
                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    Route53 (Geo-routing)                               │
├────────────────────────────────────────────────────────────────────────┤
│  • US users → us-east-1                                                │
│  • EU users → eu-west-1                                                │
│  • APAC users → ap-southeast-1                                         │
└────────────────┬───────────────────────────────────────────────────────┘
                 │
                 ├────────────────┬──────────────────┬────────────────────┐
                 ▼                ▼                  ▼                    │
┌─────────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │
│   US Region         │  │   EU Region     │  │  APAC Region    │      │
│   (us-east-1)       │  │   (eu-west-1)   │  │(ap-southeast-1) │      │
├─────────────────────┤  ├─────────────────┤  ├─────────────────┤      │
│  • EKS Cluster      │  │  • EKS Cluster  │  │  • EKS Cluster  │      │
│  • RDS PostgreSQL   │  │  • RDS PG (🇪🇺) │  │  • RDS PG       │      │
│  • S3 Bucket        │  │  • S3 (🇪🇺)     │  │  • S3           │      │
│  • ElastiCache      │  │  • ElastiCache  │  │  • ElastiCache  │      │
│                     │  │                 │  │                 │      │
│  Compliance:        │  │  Compliance:    │  │  Compliance:    │      │
│  • HIPAA            │  │  • GDPR ✅      │  │  • Local DPAs   │      │
│  • FDA 21 CFR 11    │  │  • Data residency│ │  • Encryption   │      │
└─────────────────────┘  └─────────────────┘  └─────────────────┘      │
                                                                         │
       Cross-region replication (disaster recovery only) ◄──────────────┘
       • Encrypted in transit (TLS 1.3)
       • Automated daily snapshots
       • RTO: 5 minutes, RPO: 15 minutes
```

---

## Security & Audit Flow

### Comprehensive Audit Trail

```
┌────────────────────────────────────────────────────────────────────────┐
│                      User Action Triggers                              │
│     (e.g., View patient data, Edit AE report, Submit to FDA)          │
└────────────┬───────────────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Application Layer Logging                            │
├────────────────────────────────────────────────────────────────────────┤
│  Structured Log Entry (JSON):                                          │
│  {                                                                     │
│    "timestamp": "2026-01-08T14:30:00.123Z",                            │
│    "event_type": "patient_data_accessed",                              │
│    "actor_id": "user-456",                                             │
│    "actor_role": "pharmacovigilance_specialist",                       │
│    "resource_type": "Patient",                                         │
│    "resource_id": "patient-123",                                       │
│    "action": "READ",                                                   │
│    "ip_address": "192.168.1.100",                                      │
│    "user_agent": "Mozilla/5.0...",                                     │
│    "session_id": "sess-789",                                           │
│    "correlation_id": "req-abc-123",                                    │
│    "outcome": "success",                                               │
│    "phi_accessed": ["name", "dob", "mrn"]                              │
│  }                                                                     │
└────────────┬───────────────────────────────────────────────────────────┘
             │
             ├──────────────────────┬─────────────────┬──────────────────┐
             ▼                      ▼                 ▼                  │
┌────────────────────┐  ┌──────────────────┐  ┌──────────────────┐    │
│  CloudWatch Logs   │  │ ELK Stack        │  │  SIEM            │    │
│  (Real-time)       │  │ (Elasticsearch)  │  │  (Splunk)        │    │
├────────────────────┤  ├──────────────────┤  ├──────────────────┤    │
│  • Streaming       │  │  • Index logs    │  │  • Security      │    │
│  • Alerting        │  │  • Kibana UI     │  │    analytics     │    │
│  • Metrics         │  │  • Search        │  │  • Anomaly detect│    │
└────────────────────┘  └──────────────────┘  └──────────────────┘    │
                                                                        │
             ┌──────────────────────────────────────────────────────────┘
             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Compliance Reporting Engine                          │
├────────────────────────────────────────────────────────────────────────┤
│  HIPAA Audit Report (generated monthly):                               │
│  ──────────────────────────────────────────────────────────────────    │
│  • Total PHI access events: 12,543                                     │
│  • Access by role:                                                     │
│    - Patients (own data): 8,234                                        │
│    - PV Specialists: 3,109                                             │
│    - Admins: 1,200                                                     │
│  • Failed access attempts: 23 (investigated ✅)                        │
│  • Average response time: 150ms                                        │
│  • Unauthorized access: 0 🎯                                           │
│                                                                        │
│  FDA 21 CFR Part 11 Report (electronic signatures):                   │
│  ──────────────────────────────────────────────────────────────────    │
│  • Total signed documents: 542                                         │
│  • Signature algorithm: RSA-2048 + SHA-256                             │
│  • All signatures verified: ✅                                         │
│  • Audit trail complete: ✅                                            │
│                                                                        │
│  Export formats: PDF, CSV, JSON                                        │
│  Retention: 7 years (regulatory requirement)                           │
└────────────────────────────────────────────────────────────────────────┘
```

---

*Document Version: 1.0*  
*Last Updated: 2026-01-08*  
*Owner: HealthTech Architecture Team*
