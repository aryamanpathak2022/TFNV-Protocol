# TFNV-Protocol: System Architecture Specification

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Microservices Design](#microservices-design)
3. [Data Architecture](#data-architecture)
4. [Security Architecture](#security-architecture)
5. [Integration Architecture](#integration-architecture)
6. [Deployment Architecture](#deployment-architecture)

---

## Architecture Overview

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PRESENTATION LAYER                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   Web App    │  │  Mobile App  │  │   WhatsApp   │  │  Email Client│   │
│  │  (React)     │  │(React Native)│  │   Business   │  │   (BIMI)     │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                 │                  │                 │            │
└─────────┼─────────────────┼──────────────────┼─────────────────┼────────────┘
          │                 │                  │                 │
          └─────────────────┴──────────────────┴─────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY LAYER                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      Kong API Gateway / AWS API Gateway                 │ │
│  ├────────────────────────────────────────────────────────────────────────┤ │
│  │  • Rate Limiting  • Authentication  • Request Routing  • CORS          │ │
│  │  • API Versioning • Request/Response Transform • Circuit Breaking      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MICROSERVICES LAYER                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│  │  Trust Service  │  │   GenAI RAG     │  │  FHIR Service   │            │
│  │   (NestJS)      │  │   Service       │  │   (NestJS)      │            │
│  │                 │  │  (Python)       │  │                 │            │
│  │ • BIMI/VMC      │  │ • LLM Calls     │  │ • SMART Launch  │            │
│  │ • WhatsApp API  │  │ • Vector Search │  │ • FHIR CRUD     │            │
│  │ • PKI Validation│  │ • Triage Logic  │  │ • Data Mapping  │            │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘            │
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│  │  AE Report      │  │  Gamification   │  │  Notification   │            │
│  │  Service        │  │  Service        │  │  Service        │            │
│  │  (NestJS)       │  │  (NestJS)       │  │  (NestJS)       │            │
│  │                 │  │                 │  │                 │            │
│  │ • CRUD          │  │ • Points/Levels │  │ • Email         │            │
│  │ • Workflows     │  │ • Donations     │  │ • WhatsApp      │            │
│  │ • FDA Submission│  │ • Leaderboards  │  │ • Push (FCM)    │            │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘            │
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│  │  User/Auth      │  │  Analytics      │  │  Admin          │            │
│  │  Service        │  │  Service        │  │  Service        │            │
│  │  (Keycloak)     │  │  (Python)       │  │  (NestJS)       │            │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            DATA LAYER                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │ PostgreSQL  │  │   Redis     │  │  Pinecone   │  │     S3      │       │
│  │  (Primary)  │  │  (Cache)    │  │  (Vectors)  │  │  (Files)    │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL INTEGRATIONS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   OpenAI    │  │  EHR/FHIR   │  │   Stripe    │  │  SendGrid   │       │
│  │   API       │  │  Servers    │  │  Connect    │  │  (Email)    │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  WhatsApp   │  │   DrugBank  │  │     FDA     │  │  MedDRA     │       │
│  │   Business  │  │     API     │  │  MedWatch   │  │   Coding    │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Microservices Design

### 1. Trust Service

**Responsibility**: Manage verified communication channels and identity verification

**Endpoints**:
```typescript
POST   /api/v1/trust/verify-email      // Verify sender identity via BIMI/VMC
POST   /api/v1/trust/verify-whatsapp   // Verify WhatsApp green tick status
GET    /api/v1/trust/certificate/{id}  // Get VMC certificate details
POST   /api/v1/trust/sign-message      // Cryptographically sign outbound message
GET    /api/v1/trust/health            // Health check
```

**Technology Stack**:
- Runtime: Node.js 20 LTS
- Framework: NestJS
- Database: PostgreSQL (for certificate cache)
- Cache: Redis (for verification results)
- External: DigiCert API, Meta WhatsApp API

**Key Features**:
- DKIM/SPF/DMARC configuration management
- VMC certificate lifecycle management
- WhatsApp Business API integration
- Real-time verification status dashboard

---

### 2. GenAI RAG Service

**Responsibility**: Conversational triage and auto-narrative generation

**Endpoints**:
```python
POST   /api/v1/genai/chat              // Multi-turn conversation
POST   /api/v1/genai/triage            // Classify severity and risk
POST   /api/v1/genai/generate-narrative // Create ICH E2B narrative
POST   /api/v1/genai/similarity-search // Find similar cases
GET    /api/v1/genai/embeddings/{id}  // Get vector embeddings
POST   /api/v1/genai/feedback          // Human feedback for RLHF
```

**Technology Stack**:
- Runtime: Python 3.11
- Framework: FastAPI
- LLM: OpenAI GPT-4o / Azure OpenAI
- Vector DB: Pinecone / Weaviate
- Orchestration: LangChain
- Medical NLP: scispaCy, ClinicalBERT

**RAG Pipeline**:
```python
class RAGPipeline:
    def __init__(self):
        self.embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
        self.vector_store = Pinecone(index_name="tfnv-knowledge")
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
        self.retriever = self.vector_store.as_retriever(k=5)
        
    async def triage_conversation(self, user_input: str, history: List[Message]):
        # 1. Extract medical entities
        entities = self.ner_pipeline(user_input)
        
        # 2. Semantic search for relevant guidelines
        docs = await self.retriever.aget_relevant_documents(user_input)
        
        # 3. Query drug knowledge graph
        drug_info = await self.get_drug_interactions(entities)
        
        # 4. Construct prompt with context
        prompt = self.build_prompt(user_input, docs, drug_info, history)
        
        # 5. LLM call with guardrails
        response = await self.llm.agenerate([prompt])
        
        # 6. Validate and return
        return self.validate_response(response)
```

**Key Features**:
- Multi-turn dialogue state management
- Medical entity recognition and normalization
- Drug interaction checking
- Causality assessment (Naranjo algorithm)
- MedDRA auto-coding
- Hallucination detection and guardrails

---

### 3. FHIR Service

**Responsibility**: SMART on FHIR integration and EHR data synchronization

**Endpoints**:
```typescript
GET    /api/v1/fhir/.well-known/smart-configuration  // SMART config
POST   /api/v1/fhir/launch                           // SMART app launch
GET    /api/v1/fhir/callback                         // OAuth callback
GET    /api/v1/fhir/patient/{id}                     // Fetch patient
GET    /api/v1/fhir/medication-statement             // Fetch meds
POST   /api/v1/fhir/adverse-event                    // Create AE resource
GET    /api/v1/fhir/sync-status/{session}            // Check sync status
```

**Technology Stack**:
- Runtime: Node.js 20 LTS
- Framework: NestJS
- FHIR Library: HAPI FHIR (Java) / fhir.js (TypeScript)
- OAuth: Keycloak (SMART-compliant)
- Message Queue: RabbitMQ (for async sync)

**SMART Launch Flow**:
```typescript
class SmartLaunchService {
  async handleLaunch(iss: string, launch: string): Promise<LaunchResponse> {
    // 1. Discover SMART configuration
    const config = await this.fetchSmartConfig(iss);
    
    // 2. Initiate OAuth flow
    const authUrl = this.buildAuthorizationUrl(config, {
      response_type: 'code',
      client_id: process.env.SMART_CLIENT_ID,
      scope: 'launch patient/*.read',
      launch: launch,
      redirect_uri: process.env.SMART_REDIRECT_URI,
    });
    
    return { authorizationUrl: authUrl };
  }
  
  async handleCallback(code: string, state: string): Promise<FhirContext> {
    // 3. Exchange code for token
    const tokenResponse = await this.exchangeCodeForToken(code);
    
    // 4. Fetch patient context
    const patient = await this.fetchResource(
      'Patient', 
      tokenResponse.patient, 
      tokenResponse.access_token
    );
    
    // 5. Prefill form data
    const medications = await this.fetchMedicationStatements(
      tokenResponse.patient,
      tokenResponse.access_token
    );
    
    return {
      patient,
      medications,
      accessToken: tokenResponse.access_token,
    };
  }
}
```

**Key Features**:
- SMART App Launch (standalone and EHR-embedded)
- OAuth 2.0 authorization with PKCE
- FHIR R4 resource CRUD operations
- Patient/Practitioner context resolution
- Bi-directional sync (read from and write to EHR)
- Multi-EHR vendor support (Epic, Cerner, Allscripts)

---

### 4. Adverse Event Report Service

**Responsibility**: Core adverse event lifecycle management

**Endpoints**:
```typescript
POST   /api/v1/ae-reports                    // Create draft report
GET    /api/v1/ae-reports/{id}               // Get report details
PUT    /api/v1/ae-reports/{id}               // Update report
DELETE /api/v1/ae-reports/{id}               // Delete draft
POST   /api/v1/ae-reports/{id}/submit        // Submit for review
POST   /api/v1/ae-reports/{id}/approve       // Approve report
POST   /api/v1/ae-reports/{id}/fda-submit    // Submit to FDA
GET    /api/v1/ae-reports                    // List reports (paginated)
GET    /api/v1/ae-reports/{id}/history       // Audit trail
```

**Database Schema**:
```sql
CREATE TABLE ae_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id UUID REFERENCES patients(id),
  reporter_id UUID REFERENCES users(id),
  status VARCHAR(50) NOT NULL, -- draft, submitted, approved, fda_submitted
  severity VARCHAR(20) NOT NULL, -- mild, moderate, severe, life_threatening, death
  
  -- Patient info
  patient_age INT,
  patient_gender VARCHAR(10),
  patient_weight DECIMAL(5,2),
  
  -- Event details
  event_date TIMESTAMP NOT NULL,
  event_description TEXT NOT NULL,
  event_outcome VARCHAR(50),
  
  -- Drug details (JSONB for flexibility)
  suspect_drugs JSONB NOT NULL, -- [{name, dose, route, start_date, stop_date}]
  concomitant_drugs JSONB,
  
  -- Coding
  meddra_codes JSONB, -- MedDRA PT, LLT, SOC codes
  
  -- AI-generated
  auto_narrative TEXT,
  naranjo_score INT,
  causality_assessment VARCHAR(50),
  
  -- Regulatory
  fda_submission_id VARCHAR(100),
  fda_submission_date TIMESTAMP,
  
  -- Metadata
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  created_by UUID REFERENCES users(id),
  updated_by UUID REFERENCES users(id)
);

CREATE INDEX idx_ae_reports_status ON ae_reports(status);
CREATE INDEX idx_ae_reports_patient ON ae_reports(patient_id);
CREATE INDEX idx_ae_reports_severity ON ae_reports(severity);
```

**State Machine**:
```
draft → submitted → under_review → approved → fda_submitted → closed
                           ↓
                      needs_revision
                           ↓
                         draft
```

---

### 5. Gamification Service

**Responsibility**: Points, levels, badges, and charitable micro-donations

**Endpoints**:
```typescript
GET    /api/v1/gamification/profile/{userId}      // User game profile
POST   /api/v1/gamification/award-points          // Award points
GET    /api/v1/gamification/leaderboard           // Global leaderboard
GET    /api/v1/gamification/badges/{userId}       // User badges
POST   /api/v1/gamification/donate                // Convert points to donation
GET    /api/v1/gamification/impact-dashboard      // Charity impact stats
POST   /api/v1/gamification/unlock-achievement    // Unlock badge
```

**Database Schema**:
```sql
CREATE TABLE user_game_profiles (
  user_id UUID PRIMARY KEY REFERENCES users(id),
  total_points INT DEFAULT 0,
  level VARCHAR(20) DEFAULT 'bronze', -- bronze, silver, gold, platinum, diamond
  streak_days INT DEFAULT 0,
  last_activity_date DATE,
  badges JSONB DEFAULT '[]',
  lifetime_donations DECIMAL(10,2) DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE point_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  points INT NOT NULL,
  action VARCHAR(100) NOT NULL, -- 'report_submission', 'follow_up', etc.
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE donations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  amount DECIMAL(10,2) NOT NULL,
  points_spent INT NOT NULL,
  charity_id UUID REFERENCES charities(id),
  stripe_payment_intent_id VARCHAR(255),
  status VARCHAR(50) DEFAULT 'pending', -- pending, completed, failed
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE charities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  ein VARCHAR(20), -- US Tax ID for 501(c)(3)
  guidestar_verified BOOLEAN DEFAULT false,
  logo_url TEXT,
  mission TEXT,
  website_url TEXT,
  total_received DECIMAL(12,2) DEFAULT 0
);
```

**Point Calculation Rules**:
```typescript
const POINT_RULES = {
  REPORT_SUBMISSION: 100,
  COMPLETE_REPORT_DETAILS: 50,
  FOLLOW_UP_RESPONSE: 25,
  HELP_ANOTHER_USER: 15,
  STREAK_BONUS: 10, // per day
  FIRST_REPORT: 200, // one-time bonus
};

const LEVEL_THRESHOLDS = {
  BRONZE: 0,
  SILVER: 500,
  GOLD: 2000,
  PLATINUM: 5000,
  DIAMOND: 10000,
};

const DONATION_CONVERSION_RATE = 100; // 100 points = $1 donation
```

**Key Features**:
- Real-time point tracking with Redis
- Streak calculation and bonuses
- Badge/achievement system with unlockable rewards
- Stripe Connect integration for charitable donations
- Tax receipt generation (for US users)
- Impact transparency dashboard
- Anonymous leaderboards (opt-in)

---

### 6. User & Authentication Service

**Responsibility**: Identity management, authentication, authorization

**Technology**: Keycloak (open-source IAM)

**Endpoints** (Keycloak standard):
```
POST   /realms/{realm}/protocol/openid-connect/token      // Login
POST   /realms/{realm}/protocol/openid-connect/logout     // Logout
GET    /realms/{realm}/protocol/openid-connect/userinfo   // User info
POST   /admin/realms/{realm}/users                        // Create user (admin)
```

**Roles & Permissions**:
```yaml
Roles:
  - patient: Can submit own AE reports, view own data
  - reporter: Can submit reports on behalf of patients (HCP)
  - pharmacovigilance_specialist: Can review and approve reports
  - regulator: Read-only access to all reports
  - admin: Full system access

RBAC Policy:
  - ae_reports.create: [patient, reporter]
  - ae_reports.read_own: [patient, reporter]
  - ae_reports.read_all: [pharmacovigilance_specialist, regulator, admin]
  - ae_reports.approve: [pharmacovigilance_specialist, admin]
  - ae_reports.fda_submit: [pharmacovigilance_specialist, admin]
```

**OAuth 2.0 Flows**:
- **Web/Mobile Apps**: Authorization Code Flow with PKCE
- **EHR Integration**: SMART on FHIR (special OAuth profile)
- **Service-to-Service**: Client Credentials Flow
- **MFA**: TOTP (Google Authenticator) + SMS backup

---

### 7. Notification Service

**Responsibility**: Multi-channel notification delivery

**Endpoints**:
```typescript
POST   /api/v1/notifications/send           // Send notification
GET    /api/v1/notifications/{userId}       // Get user notifications
PUT    /api/v1/notifications/{id}/read      // Mark as read
POST   /api/v1/notifications/preferences    // Update preferences
```

**Channels**:
1. **Email** (via SendGrid with BIMI)
2. **WhatsApp** (via Meta Business API)
3. **Push Notifications** (via Firebase Cloud Messaging)
4. **SMS** (via Twilio - for critical alerts)

**Notification Types**:
```typescript
enum NotificationType {
  AE_REPORT_SUBMITTED = 'ae_report_submitted',
  AE_REPORT_APPROVED = 'ae_report_approved',
  AE_REPORT_NEEDS_REVISION = 'ae_report_needs_revision',
  FDA_SUBMISSION_SUCCESS = 'fda_submission_success',
  FOLLOW_UP_REQUIRED = 'follow_up_required',
  GAMIFICATION_LEVEL_UP = 'gamification_level_up',
  GAMIFICATION_BADGE_UNLOCKED = 'gamification_badge_unlocked',
  DONATION_CONFIRMED = 'donation_confirmed',
}
```

**Queue-based Architecture**:
```
User Action → API → Publish to RabbitMQ → Notification Worker → Channel Delivery
                           ↓
                    (Retry logic, DLQ)
```

---

## Data Architecture

### Entity Relationship Diagram

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Users     │         │  Patients   │         │ AE Reports  │
├─────────────┤         ├─────────────┤         ├─────────────┤
│ id (PK)     │────────▶│ id (PK)     │◀───────│ id (PK)     │
│ email       │         │ user_id (FK)│         │ patient_id  │
│ name        │         │ mrn         │         │ reporter_id │
│ role        │         │ dob         │         │ status      │
│ verified    │         │ gender      │         │ severity    │
└─────────────┘         └─────────────┘         │ event_date  │
                                                 │ meddra_codes│
                                                 └─────────────┘
                                                        │
                                                        │
                        ┌───────────────────────────────┤
                        │                               │
                        ▼                               ▼
                ┌─────────────┐                 ┌─────────────┐
                │ AE Drugs    │                 │ AE Workflows│
                ├─────────────┤                 ├─────────────┤
                │ id (PK)     │                 │ id (PK)     │
                │ ae_report_id│                 │ ae_report_id│
                │ drug_name   │                 │ from_status │
                │ dose        │                 │ to_status   │
                │ route       │                 │ actor_id    │
                │ role (S/C)  │                 │ timestamp   │
                └─────────────┘                 └─────────────┘


┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Users     │         │Game Profiles│         │ Donations   │
│             │────────▶│             │────────▶│             │
│             │         │ total_points│         │ amount      │
│             │         │ level       │         │ charity_id  │
│             │         │ streak_days │         │ points_spent│
└─────────────┘         └─────────────┘         └─────────────┘
                                                        │
                                                        ▼
                                                 ┌─────────────┐
                                                 │  Charities  │
                                                 ├─────────────┤
                                                 │ id (PK)     │
                                                 │ name        │
                                                 │ ein         │
                                                 │ verified    │
                                                 └─────────────┘
```

### Data Partitioning Strategy

**PostgreSQL Partitioning** (for scalability):
```sql
-- Partition ae_reports by created_at (yearly)
CREATE TABLE ae_reports_2026 PARTITION OF ae_reports
  FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

CREATE TABLE ae_reports_2027 PARTITION OF ae_reports
  FOR VALUES FROM ('2027-01-01') TO ('2028-01-01');
```

**Caching Strategy** (Redis):
```
Key Pattern: {service}:{resource}:{id}

Examples:
- trust:vmc:cert123                 → VMC certificate (TTL: 24h)
- fhir:patient:pt-456               → Patient FHIR resource (TTL: 1h)
- gamification:leaderboard:global   → Leaderboard data (TTL: 5m)
- genai:embeddings:emb-789          → Vector embeddings (TTL: 7d)
```

### Data Retention Policy

| Data Type | Retention Period | Rationale |
|-----------|------------------|-----------|
| AE Reports | 7 years (active) | FDA/regulatory requirement |
| Audit Logs | 7 years | HIPAA/compliance |
| User Sessions | 24 hours | Security best practice |
| LLM Conversations | 90 days | Training/improvement |
| Analytics Events | 2 years | Business intelligence |
| Backups | 30 days (rolling) | Disaster recovery |

---

## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Perimeter Security                                │
│  • WAF (AWS WAF) - Block common attacks (SQLi, XSS)         │
│  • DDoS Protection (AWS Shield)                             │
│  • CDN (CloudFront) - Caching and edge protection          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: Network Security                                  │
│  • VPC with private subnets                                 │
│  • Security Groups (least privilege)                        │
│  • NACLs (Network ACLs)                                     │
│  • TLS 1.3 for all traffic                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Application Security                              │
│  • API Gateway with rate limiting                           │
│  • OAuth 2.0 + OIDC authentication                          │
│  • JWT tokens with short expiry (15 min)                    │
│  • RBAC/ABAC authorization                                  │
│  • Input validation and sanitization                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: Data Security                                     │
│  • Encryption at rest (AES-256-GCM)                         │
│  • Encryption in transit (TLS 1.3)                          │
│  • PII tokenization (HashiCorp Vault)                       │
│  • Database access controls                                 │
│  • Automated backup encryption                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 5: Monitoring & Response                             │
│  • SIEM (Splunk/DataDog Security)                           │
│  • Intrusion Detection (AWS GuardDuty)                      │
│  • Audit logging (CloudTrail, ELK)                          │
│  • Incident response playbooks                              │
└─────────────────────────────────────────────────────────────┘
```

### Cryptographic Standards

**Encryption**:
- At Rest: AES-256-GCM
- In Transit: TLS 1.3 (minimum 1.2)
- Key Management: AWS KMS / HashiCorp Vault
- Certificate Authority: Let's Encrypt (with auto-renewal)

**Hashing**:
- Passwords: Argon2id (OWASP recommended)
- API Signatures: HMAC-SHA256
- File Integrity: SHA-256

**Digital Signatures**:
- Email (DKIM): RSA-2048 or Ed25519
- JWT: RS256 (RSA + SHA-256)
- FDA Submissions: X.509 certificates

---

## Integration Architecture

### Integration Patterns

#### 1. Synchronous (REST API)
**Use Case**: Real-time operations (CRUD, authentication)
```
Client → API Gateway → Microservice → Database → Response
```

#### 2. Asynchronous (Message Queue)
**Use Case**: Background processing (notifications, analytics)
```
Producer → RabbitMQ → Consumer Workers → Process
```

#### 3. Event-Driven (Pub/Sub)
**Use Case**: Cross-service communication (AE report status changes)
```
Service A → Publish Event → Message Broker → Subscribe → Service B/C/D
```

#### 4. Batch Processing (Scheduled Jobs)
**Use Case**: Nightly FDA submissions, report generation
```
Cron Job → Trigger Lambda → Batch Process → Store Results
```

### External API Integration

**OpenAI API** (GenAI RAG):
```typescript
// Resilience pattern: Retry with exponential backoff
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  maxRetries: 3,
  timeout: 60000, // 60 seconds
});

// Cost optimization: Caching
const cacheKey = `openai:${promptHash}`;
let response = await redis.get(cacheKey);
if (!response) {
  response = await openai.chat.completions.create({...});
  await redis.setex(cacheKey, 3600, JSON.stringify(response));
}
```

**FHIR Servers** (EHR Integration):
```typescript
// Circuit breaker pattern to prevent cascade failures
const circuitBreaker = new CircuitBreaker(fetchFhirResource, {
  timeout: 5000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000,
});

try {
  const patient = await circuitBreaker.fire('Patient/123');
} catch (err) {
  // Fallback: Prompt manual entry
  return { prefillAvailable: false };
}
```

---

## Deployment Architecture

### Kubernetes Cluster Topology

```
┌────────────────────────────────────────────────────────────────┐
│                    AWS EKS Cluster (Production)                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    Namespace: ingress                     │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │  • NGINX Ingress Controller (2 replicas)                │ │
│  │  • AWS ALB integration                                  │ │
│  │  • TLS termination                                      │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                 Namespace: tfnv-services                 │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │                                                          │ │
│  │  Deployments (with HPA):                                │ │
│  │  • trust-service (3 replicas)                           │ │
│  │  • genai-rag-service (5 replicas, GPU nodes)            │ │
│  │  • fhir-service (3 replicas)                            │ │
│  │  • ae-report-service (4 replicas)                       │ │
│  │  • gamification-service (2 replicas)                    │ │
│  │  • notification-service (2 replicas)                    │ │
│  │                                                          │ │
│  │  StatefulSets:                                          │ │
│  │  • redis-cluster (3 nodes)                              │ │
│  │  • rabbitmq-cluster (3 nodes)                           │ │
│  │                                                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                  Namespace: keycloak                     │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │  • keycloak (3 replicas)                                │ │
│  │  • External DB: AWS RDS PostgreSQL                      │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                 Namespace: observability                 │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │  • Prometheus (metrics)                                 │ │
│  │  • Grafana (dashboards)                                 │ │
│  │  • Loki (logs aggregation)                              │ │
│  │  • Jaeger (distributed tracing)                         │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Multi-Region Setup (Disaster Recovery)

**Primary Region**: us-east-1 (North Virginia)
**Secondary Region**: us-west-2 (Oregon)

**Active-Active Configuration**:
- DNS: Route53 with latency-based routing
- Database: RDS with cross-region read replicas
- Files: S3 with cross-region replication
- Kubernetes: Independent EKS clusters in each region
- Traffic Split: 80% primary, 20% secondary (for testing)

**Failover Process**:
1. Health check failure detected (3 consecutive checks)
2. Route53 automatically routes traffic to secondary region
3. Secondary region promotes read replica to primary (RTO: 5 minutes)
4. Alerts sent to on-call team via PagerDuty

**Recovery Time Objective (RTO)**: 5 minutes  
**Recovery Point Objective (RPO)**: 15 minutes (database replication lag)

### CI/CD Pipeline

```
┌──────────────┐
│  Developer   │
│  Git Push    │
└──────┬───────┘
       │
       ▼
┌────────────────────────────────────────┐
│        GitHub Actions (CI)             │
├────────────────────────────────────────┤
│ 1. Checkout code                       │
│ 2. Run linters (ESLint, Pylint)        │
│ 3. Run unit tests                      │
│ 4. Run integration tests               │
│ 5. Security scan (Snyk, CodeQL)        │
│ 6. Build Docker images                 │
│ 7. Push to ECR (AWS Container Registry)│
│ 8. Update image tags in GitOps repo    │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│         GitOps Repo (CD)               │
│     (Kubernetes manifests)             │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│          ArgoCD (CD)                   │
├────────────────────────────────────────┤
│ 1. Detect manifest changes             │
│ 2. Validate manifests                  │
│ 3. Sync to Kubernetes (dev → staging)  │
│ 4. Run smoke tests                     │
│ 5. Manual approval for production      │
│ 6. Blue-green deployment to prod       │
│ 7. Health checks                       │
│ 8. Rollback if failure                 │
└────────────────────────────────────────┘
```

---

## Performance & Scalability

### Target SLAs

| Metric | Target | Measurement |
|--------|--------|-------------|
| API Latency (p95) | <200ms | Prometheus |
| API Latency (p99) | <500ms | Prometheus |
| System Uptime | 99.9% | DataDog |
| Error Rate | <0.1% | DataDog |
| GenAI Response Time | <3s | Custom metric |
| FHIR Sync Time | <2s | Custom metric |

### Horizontal Pod Autoscaling (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ae-report-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ae-report-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Database Optimization

**Read Replicas**:
- Primary: Write operations
- 2 Read Replicas: Read-heavy operations (analytics, reports)
- Connection pooling: PgBouncer (max 100 connections)

**Query Optimization**:
- Indexing on frequently queried columns
- Materialized views for complex aggregations
- Query result caching (Redis)

**Database Sharding** (future):
- Shard key: `patient_id` or `tenant_id` (for multi-tenancy)
- Shard distribution: Consistent hashing

---

## Conclusion

The TFNV-Protocol architecture is designed for:
- **Scalability**: Microservices + Kubernetes + HPA
- **Reliability**: Multi-region, circuit breakers, retries
- **Security**: Defense in depth, encryption, RBAC
- **Compliance**: HIPAA, FDA 21 CFR Part 11, SOC 2
- **Observability**: Comprehensive logging, metrics, tracing

This architecture supports the four core pillars (Trust, GenAI, FHIR, Gamification) while maintaining flexibility for future enhancements.

---

*Document Version: 1.0*  
*Last Updated: 2026-01-08*  
*Owner: HealthTech Architecture Team*
