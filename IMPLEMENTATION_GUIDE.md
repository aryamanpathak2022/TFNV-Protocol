# TFNV-Protocol: Phase-by-Phase Implementation Guide

## Table of Contents
1. [Phase 1: Trust Layer Implementation](#phase-1-trust-layer-implementation)
2. [Phase 2: GenAI RAG Core Implementation](#phase-2-genai-rag-core-implementation)
3. [Phase 3: SMART on FHIR Implementation](#phase-3-smart-on-fhir-implementation)
4. [Phase 4: Gamification for Good Implementation](#phase-4-gamification-for-good-implementation)

---

## Phase 1: Trust Layer Implementation

### Overview
Build verified communication channels to eliminate "scam fear" through BIMI/VMC and WhatsApp Green Tick.

### Prerequisites
- Domain ownership with DNS control
- Email sending infrastructure decision (SendGrid vs AWS SES)
- Meta Business account
- Budget for VMC certificate (~$1,500/year)

---

### Step 1.1: Email Authentication Stack (BIMI/VMC)

#### A. Configure SPF (Sender Policy Framework)

**DNS TXT Record**:
```dns
Name: @
Type: TXT
Value: v=spf1 include:sendgrid.net include:_spf.google.com ~all
```

**Verification**:
```bash
dig TXT tfnv-protocol.com
# Should return the SPF record
```

#### B. Configure DKIM (DomainKeys Identified Mail)

**Generate DKIM Keys** (via SendGrid):
```bash
# SendGrid will provide DKIM CNAME records
# Example:
s1._domainkey.tfnv-protocol.com CNAME s1.domainkey.u12345.wl.sendgrid.net
s2._domainkey.tfnv-protocol.com CNAME s2.domainkey.u12345.wl.sendgrid.net
```

Add CNAME records to DNS.

**Verification**:
```bash
dig CNAME s1._domainkey.tfnv-protocol.com
```

#### C. Configure DMARC (Domain-based Message Authentication)

**DNS TXT Record**:
```dns
Name: _dmarc
Type: TXT
Value: v=DMARC1; p=quarantine; rua=mailto:dmarc@tfnv-protocol.com; ruf=mailto:dmarc@tfnv-protocol.com; fo=1; adkim=s; aspf=s; pct=100
```

**Explanation**:
- `p=quarantine`: Policy to quarantine unauthenticated emails
- `rua`: Aggregate reports sent here
- `adkim=s`: Strict DKIM alignment
- `aspf=s`: Strict SPF alignment

**Verification**:
```bash
dig TXT _dmarc.tfnv-protocol.com
```

#### D. Obtain VMC (Verified Mark Certificate)

**Process**:
1. **Trademark Registration**: Ensure brand logo is registered with USPTO/EUIPO
2. **Logo Preparation**: 
   - Format: SVG (Scalable Vector Graphics)
   - Size: Square aspect ratio (recommended 1:1)
   - Requirements: Tiny SVG specification
   
3. **Apply for VMC** (via DigiCert, Entrust, or Sectigo):
   - Provide trademark documentation
   - Submit logo SVG file
   - Domain validation
   - Cost: ~$1,500/year

4. **Receive VMC Certificate**: Contains logo and cryptographic proof

#### E. Configure BIMI DNS Record

**DNS TXT Record**:
```dns
Name: default._bimi
Type: TXT
Value: v=BIMI1; l=https://tfnv-protocol.com/logo.svg; a=https://tfnv-protocol.com/vmc.pem
```

**Explanation**:
- `l=`: URL to brand logo (SVG)
- `a=`: URL to VMC certificate (PEM format)

**Host Logo and VMC**:
```bash
# Host on CDN (CloudFront) for high availability
aws s3 cp logo.svg s3://tfnv-assets/logo.svg --acl public-read
aws s3 cp vmc.pem s3://tfnv-assets/vmc.pem --acl public-read
```

#### F. Implementation Code (NestJS)

**trust.service.ts**:
```typescript
import { Injectable } from '@nestjs/common';
import * as nodemailer from 'nodemailer';

@Injectable()
export class TrustService {
  private transporter: nodemailer.Transporter;

  constructor() {
    this.transporter = nodemailer.createTransport({
      host: 'smtp.sendgrid.net',
      port: 587,
      secure: false,
      auth: {
        user: 'apikey',
        pass: process.env.SENDGRID_API_KEY,
      },
    });
  }

  async sendVerifiedEmail(to: string, subject: string, html: string) {
    const info = await this.transporter.sendMail({
      from: '"TFNV Protocol" <noreply@tfnv-protocol.com>',
      to,
      subject,
      html,
      headers: {
        // Custom headers for tracking
        'X-TFNV-Message-ID': this.generateMessageId(),
        'X-TFNV-Signature': await this.signMessage(html),
      },
    });

    return {
      messageId: info.messageId,
      accepted: info.accepted,
      rejected: info.rejected,
    };
  }

  private generateMessageId(): string {
    return `tfnv-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  private async signMessage(content: string): Promise<string> {
    // HMAC-SHA256 signature for message integrity
    const crypto = require('crypto');
    const secret = process.env.MESSAGE_SIGNING_SECRET;
    return crypto
      .createHmac('sha256', secret)
      .update(content)
      .digest('hex');
  }

  async verifyEmail(email: string): Promise<boolean> {
    // Query DNS for SPF, DKIM, DMARC, BIMI
    const dns = require('dns').promises;
    
    try {
      const domain = email.split('@')[1];
      
      // Check SPF
      const spfRecords = await dns.resolveTxt(domain);
      const hasSpf = spfRecords.some(record => 
        record.join('').startsWith('v=spf1')
      );
      
      // Check DMARC
      const dmarcRecords = await dns.resolveTxt(`_dmarc.${domain}`);
      const hasDmarc = dmarcRecords.some(record => 
        record.join('').startsWith('v=DMARC1')
      );
      
      // Check BIMI
      const bimiRecords = await dns.resolveTxt(`default._bimi.${domain}`);
      const hasBimi = bimiRecords.some(record => 
        record.join('').startsWith('v=BIMI1')
      );
      
      return hasSpf && hasDmarc && hasBimi;
    } catch (error) {
      console.error('Email verification failed:', error);
      return false;
    }
  }
}
```

**Testing BIMI**:
```bash
# Use BIMI Inspector (online tool)
# https://bimigroup.org/bimi-generator/

# Or command line:
dig TXT default._bimi.tfnv-protocol.com

# Test email delivery to Gmail:
# Send test email and check for logo display next to sender name
```

---

### Step 1.2: WhatsApp Business Green Tick

#### A. Create Meta Business Account

1. Go to https://business.facebook.com/
2. Create Business Account
3. Add business details:
   - Business name: TFNV Protocol
   - Business address
   - Tax ID / EIN
   - Website: https://tfnv-protocol.com

#### B. Apply for WhatsApp Business API

1. Go to https://business.facebook.com/wa/manage/home/
2. Create WhatsApp Business Account
3. Apply for API access:
   - Select "Business Solution Provider" or "Direct Client"
   - Provide use case: Healthcare adverse event reporting
   - Estimated monthly message volume

#### C. Get Green Tick Verification

**Requirements**:
- Official business verification (Meta will review)
- Business website with clear information
- Consistent branding across platforms
- High-quality use case (healthcare)
- No policy violations

**Process**:
1. Submit verification request in WhatsApp Business Manager
2. Provide business documentation:
   - Certificate of incorporation
   - Business license
   - Utility bill or bank statement (proof of address)
3. Wait for approval (1-2 weeks)

#### D. Configure WhatsApp Business API

**Install WhatsApp Business API Client** (On-Premise or Cloud):

**Cloud API (Recommended)**:
```bash
# No installation needed, use Meta's Cloud API
# Base URL: https://graph.facebook.com/v18.0/{phone-number-id}/messages
```

**Webhook Setup**:
```typescript
// webhooks.controller.ts
import { Controller, Post, Body, Get, Query } from '@nestjs/common';

@Controller('webhooks/whatsapp')
export class WhatsAppWebhookController {
  @Get()
  verify(@Query() query: any) {
    // Verify webhook (Meta requirement)
    const mode = query['hub.mode'];
    const token = query['hub.verify_token'];
    const challenge = query['hub.challenge'];

    if (mode === 'subscribe' && token === process.env.WHATSAPP_VERIFY_TOKEN) {
      return parseInt(challenge, 10);
    }
    return 'Verification failed';
  }

  @Post()
  async handleMessage(@Body() body: any) {
    // Handle incoming messages
    if (body.object === 'whatsapp_business_account') {
      for (const entry of body.entry) {
        for (const change of entry.changes) {
          if (change.field === 'messages') {
            const message = change.value.messages[0];
            await this.processMessage(message);
          }
        }
      }
    }
    return { status: 'ok' };
  }

  private async processMessage(message: any) {
    // Process incoming message (user response to AE follow-up)
    console.log('Received message:', message);
    // Business logic here...
  }
}
```

**Send Message via WhatsApp**:
```typescript
// whatsapp.service.ts
import { Injectable } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class WhatsAppService {
  private readonly apiUrl = `https://graph.facebook.com/v18.0/${process.env.WHATSAPP_PHONE_NUMBER_ID}/messages`;
  private readonly accessToken = process.env.WHATSAPP_ACCESS_TOKEN;

  async sendMessage(to: string, message: string) {
    try {
      const response = await axios.post(
        this.apiUrl,
        {
          messaging_product: 'whatsapp',
          to: to, // Phone number in format: 1234567890
          type: 'text',
          text: {
            body: message,
          },
        },
        {
          headers: {
            Authorization: `Bearer ${this.accessToken}`,
            'Content-Type': 'application/json',
          },
        }
      );

      return {
        success: true,
        messageId: response.data.messages[0].id,
      };
    } catch (error) {
      console.error('WhatsApp send failed:', error.response?.data);
      throw error;
    }
  }

  async sendTemplate(to: string, templateName: string, params: string[]) {
    // Send pre-approved template message (for initial contact)
    const response = await axios.post(
      this.apiUrl,
      {
        messaging_product: 'whatsapp',
        to: to,
        type: 'template',
        template: {
          name: templateName,
          language: { code: 'en_US' },
          components: [
            {
              type: 'body',
              parameters: params.map(p => ({ type: 'text', text: p })),
            },
          ],
        },
      },
      {
        headers: {
          Authorization: `Bearer ${this.accessToken}`,
          'Content-Type': 'application/json',
        },
      }
    );

    return response.data;
  }

  async verifyGreenTick(phoneNumber: string): Promise<boolean> {
    // Check if phone number has green tick verification
    try {
      const response = await axios.get(
        `https://graph.facebook.com/v18.0/${process.env.WHATSAPP_BUSINESS_ACCOUNT_ID}`,
        {
          params: {
            fields: 'verified_name,quality_rating',
          },
          headers: {
            Authorization: `Bearer ${this.accessToken}`,
          },
        }
      );

      return response.data.verified_name !== undefined;
    } catch (error) {
      console.error('Green tick verification failed:', error);
      return false;
    }
  }
}
```

**Create Message Templates** (in Meta Business Manager):

Template Name: `ae_follow_up`
```
Hello {{1}},

Thank you for reporting the adverse event. We need a few more details to complete your report.

Please reply with:
1. When did the symptoms start?
2. Are you still experiencing them?

Your safety is our priority.

- TFNV Protocol Team
```

#### E. Integration Testing

**Test Checklist**:
```bash
# 1. Send test message to your phone
curl -X POST "https://graph.facebook.com/v18.0/$PHONE_NUMBER_ID/messages" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "to": "1234567890",
    "type": "text",
    "text": {
      "body": "Test message from TFNV Protocol"
    }
  }'

# 2. Verify green tick is visible in WhatsApp
# Open WhatsApp, check sender name has green checkmark

# 3. Test webhook by sending reply
# Check logs for received webhook
```

---

### Step 1.3: Trust Verification Dashboard

**dashboard.tsx** (React):
```typescript
import React, { useEffect, useState } from 'react';

interface TrustStatus {
  email: {
    spf: boolean;
    dkim: boolean;
    dmarc: boolean;
    bimi: boolean;
    vmcValid: boolean;
  };
  whatsapp: {
    greenTick: boolean;
    qualityRating: string;
    messageLimit: number;
  };
}

export const TrustDashboard: React.FC = () => {
  const [status, setStatus] = useState<TrustStatus | null>(null);

  useEffect(() => {
    fetch('/api/v1/trust/status')
      .then(res => res.json())
      .then(setStatus);
  }, []);

  if (!status) return <div>Loading...</div>;

  return (
    <div className="trust-dashboard">
      <h1>Trust Layer Status</h1>
      
      <div className="email-verification">
        <h2>Email Authentication</h2>
        <StatusItem label="SPF" value={status.email.spf} />
        <StatusItem label="DKIM" value={status.email.dkim} />
        <StatusItem label="DMARC" value={status.email.dmarc} />
        <StatusItem label="BIMI" value={status.email.bimi} />
        <StatusItem label="VMC Certificate" value={status.email.vmcValid} />
      </div>

      <div className="whatsapp-verification">
        <h2>WhatsApp Business</h2>
        <StatusItem label="Green Tick" value={status.whatsapp.greenTick} />
        <div>Quality Rating: {status.whatsapp.qualityRating}</div>
        <div>Message Limit: {status.whatsapp.messageLimit}/day</div>
      </div>
    </div>
  );
};

const StatusItem: React.FC<{ label: string; value: boolean }> = ({ label, value }) => (
  <div className="status-item">
    <span>{label}:</span>
    <span className={value ? 'success' : 'failure'}>
      {value ? '✅ Verified' : '❌ Not Verified'}
    </span>
  </div>
);
```

---

### Phase 1 Deliverables Checklist

- [ ] SPF record configured and verified
- [ ] DKIM keys generated and DNS records added
- [ ] DMARC policy configured with reporting
- [ ] VMC certificate obtained from DigiCert/Entrust
- [ ] BIMI DNS record published
- [ ] Email sends with BIMI logo visible in Gmail/Outlook
- [ ] Meta Business Account created
- [ ] WhatsApp Business API access approved
- [ ] WhatsApp Green Tick verification obtained
- [ ] WhatsApp webhook configured and tested
- [ ] Trust verification API implemented
- [ ] Trust dashboard deployed
- [ ] End-to-end testing completed
- [ ] Documentation created

---

## Phase 2: GenAI RAG Core Implementation

### Overview
Build FDA-compliant conversational triage and auto-narrative generation using RAG architecture.

---

### Step 2.1: Vector Database Setup (Pinecone)

#### A. Create Pinecone Index

```python
import pinecone

# Initialize Pinecone
pinecone.init(
    api_key=os.getenv("PINECONE_API_KEY"),
    environment=os.getenv("PINECONE_ENVIRONMENT")  # e.g., us-west1-gcp
)

# Create index with dimensions matching OpenAI embeddings
index_name = "tfnv-knowledge"
if index_name not in pinecone.list_indexes():
    pinecone.create_index(
        name=index_name,
        dimension=3072,  # text-embedding-3-large has 3072 dimensions
        metric="cosine",
        pods=1,
        pod_type="p1.x1"  # Performance tier
    )

index = pinecone.Index(index_name)
```

#### B. Ingest Knowledge Base

**Data Sources**:
1. FDA MedWatch Guidelines (PDF)
2. ICH E2B(R3) Standards (PDF)
3. Drug Monographs from DrugBank (JSON)
4. Historical Adverse Event Reports (Anonymized)
5. MedDRA Terminology (CSV)

**Ingestion Script**:
```python
from langchain.document_loaders import PyPDFLoader, JSONLoader, CSVLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Pinecone

# Load documents
pdf_loader = PyPDFLoader("fda_medwatch_guidelines.pdf")
pdf_docs = pdf_loader.load()

json_loader = JSONLoader("drugbank_monographs.json", jq_schema=".drugs[]")
json_docs = json_loader.load()

# Split into chunks
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    length_function=len,
)

chunks = text_splitter.split_documents(pdf_docs + json_docs)

# Generate embeddings and upload to Pinecone
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vectorstore = Pinecone.from_documents(
    chunks,
    embeddings,
    index_name="tfnv-knowledge"
)

print(f"Ingested {len(chunks)} chunks into Pinecone")
```

---

### Step 2.2: Conversational Triage System

**rag_service.py**:
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from langchain.chat_models import ChatOpenAI
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferMemory
from langchain.vectorstores import Pinecone
from langchain.embeddings import OpenAIEmbeddings
import pinecone

app = FastAPI()

# Initialize components
pinecone.init(api_key=os.getenv("PINECONE_API_KEY"), environment=os.getenv("PINECONE_ENVIRONMENT"))
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vectorstore = Pinecone.from_existing_index("tfnv-knowledge", embeddings)
llm = ChatOpenAI(model="gpt-4o", temperature=0)

class ChatRequest(BaseModel):
    session_id: str
    message: str
    history: list = []

class ChatResponse(BaseModel):
    response: str
    severity: str  # mild, moderate, severe, life_threatening
    next_questions: list
    confidence: float

@app.post("/api/v1/genai/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    # Build conversational retrieval chain
    memory = ConversationBufferMemory(memory_key="chat_history", return_messages=True)
    
    # Pre-populate memory with history
    for msg in request.history:
        if msg['role'] == 'user':
            memory.chat_memory.add_user_message(msg['content'])
        else:
            memory.chat_memory.add_ai_message(msg['content'])
    
    qa_chain = ConversationalRetrievalChain.from_llm(
        llm=llm,
        retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
        memory=memory,
        return_source_documents=True,
    )
    
    # Custom prompt template
    from langchain.prompts import PromptTemplate
    
    template = """You are an FDA-compliant AI assistant for pharmacovigilance.
    
    Context from knowledge base:
    {context}
    
    Conversation history:
    {chat_history}
    
    Patient's latest message:
    {question}
    
    Your task:
    1. Extract key medical information (drug names, symptoms, timing)
    2. Assess severity using ICH E2B criteria
    3. Ask targeted follow-up questions to gather complete information
    4. Be empathetic and clear
    5. NEVER diagnose or provide medical advice
    
    Respond with a helpful message that guides the patient through reporting.
    """
    
    result = qa_chain({"question": request.message})
    
    # Extract severity and next questions using a second LLM call
    severity_prompt = f"""
    Based on this patient input: "{request.message}"
    And this AI response: "{result['answer']}"
    
    Classify severity as: mild, moderate, severe, or life_threatening
    Also suggest 2-3 follow-up questions.
    
    Return JSON:
    {{
      "severity": "...",
      "next_questions": ["...", "...", "..."],
      "confidence": 0.0-1.0
    }}
    """
    
    severity_result = await llm.apredict(severity_prompt)
    severity_data = json.loads(severity_result)
    
    return ChatResponse(
        response=result['answer'],
        severity=severity_data['severity'],
        next_questions=severity_data['next_questions'],
        confidence=severity_data['confidence']
    )
```

---

### Step 2.3: Auto-Narrative Generation

**narrative_generator.py**:
```python
from langchain.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class AdverseEventNarrative(BaseModel):
    narrative: str = Field(description="ICH E2B compliant narrative")
    meddra_codes: dict = Field(description="MedDRA coding (PT, LLT, SOC)")
    naranjo_score: int = Field(description="Naranjo causality score (0-13)")
    causality: str = Field(description="Causality assessment")

@app.post("/api/v1/genai/generate-narrative")
async def generate_narrative(ae_data: dict):
    """
    Generate ICH E2B(R3) compliant narrative from structured AE data
    """
    
    # Build prompt with all collected information
    prompt = f"""
    Generate an ICH E2B(R3) compliant adverse event narrative.
    
    Patient Information:
    - Age: {ae_data['patient_age']}, Gender: {ae_data['patient_gender']}
    - Weight: {ae_data['patient_weight']} kg
    
    Suspect Drug(s):
    {json.dumps(ae_data['suspect_drugs'], indent=2)}
    
    Concomitant Medications:
    {json.dumps(ae_data['concomitant_drugs'], indent=2)}
    
    Event Description:
    {ae_data['event_description']}
    
    Event Date: {ae_data['event_date']}
    Outcome: {ae_data['event_outcome']}
    
    Tasks:
    1. Write a concise, factual narrative (ICH E2B format)
    2. Assign MedDRA codes (PT, LLT, SOC)
    3. Calculate Naranjo causality score
    4. Provide causality assessment (definite, probable, possible, unlikely, unrelated)
    
    Narrative Guidelines:
    - Start with patient demographics
    - Describe temporal relationship between drug and event
    - Include relevant medical history and concomitant medications
    - Describe event onset, progression, and outcome
    - Note any dechallenge/rechallenge information
    - Use past tense, third person
    - Be objective and factual
    
    Return structured JSON following the AdverseEventNarrative schema.
    """
    
    parser = PydanticOutputParser(pydantic_object=AdverseEventNarrative)
    
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    result = await llm.apredict(prompt + "\n\n" + parser.get_format_instructions())
    
    narrative_obj = parser.parse(result)
    
    return narrative_obj.dict()
```

**Example Generated Narrative**:
```
A 45-year-old male patient weighing 80 kg initiated treatment with ibuprofen 
400 mg orally three times daily for chronic back pain. Concomitant medications 
included lisinopril 10 mg daily for hypertension. Approximately 3 days after 
starting ibuprofen, the patient developed a generalized erythematous maculopapular 
rash on the trunk and extremities, accompanied by mild pruritus. The patient 
denied fever, mucosal involvement, or systemic symptoms. Upon discontinuation 
of ibuprofen (dechallenge), the rash resolved within 5 days without medical 
intervention. The patient did not resume ibuprofen (no rechallenge). 
The event is considered probably related to ibuprofen based on temporal relationship 
and dechallenge response. MedDRA coding: PT: Rash maculo-papular (10037868), 
SOC: Skin and subcutaneous tissue disorders. Naranjo score: 6 (probable).
```

---

### Step 2.4: Medical Entity Recognition

**ner_service.py**:
```python
import spacy
from scispacy.linking import EntityLinker

# Load scispaCy model with entity linker
nlp = spacy.load("en_core_sci_lg")
nlp.add_pipe("scispacy_linker", config={"resolve_abbreviations": True, "linker_name": "umls"})

def extract_medical_entities(text: str) -> dict:
    """
    Extract drugs, symptoms, and diseases from free text
    """
    doc = nlp(text)
    
    entities = {
        "drugs": [],
        "symptoms": [],
        "diseases": [],
        "procedures": []
    }
    
    for ent in doc.ents:
        entity_data = {
            "text": ent.text,
            "label": ent.label_,
            "start": ent.start_char,
            "end": ent.end_char,
        }
        
        # Get UMLS concept IDs
        if ent._.kb_ents:
            cui = ent._.kb_ents[0][0]  # Top UMLS CUI
            entity_data["cui"] = cui
            entity_data["definition"] = linker.kb.cui_to_entity[cui].definition
        
        # Categorize by semantic type
        if ent.label_ in ["DRUG", "CHEMICAL"]:
            entities["drugs"].append(entity_data)
        elif ent.label_ in ["SYMPTOM", "SIGN"]:
            entities["symptoms"].append(entity_data)
        elif ent.label_ in ["DISEASE", "DISORDER"]:
            entities["diseases"].append(entity_data)
        elif ent.label_ == "PROCEDURE":
            entities["procedures"].append(entity_data)
    
    return entities

# Example usage
text = "Patient took ibuprofen and developed a severe rash and facial swelling"
entities = extract_medical_entities(text)
# Output:
# {
#   "drugs": [{"text": "ibuprofen", "label": "DRUG", "cui": "C0020740", ...}],
#   "symptoms": [{"text": "rash", ...}, {"text": "facial swelling", ...}]
# }
```

---

### Step 2.5: Guardrails & Safety

**guardrails_config.py**:
```python
from nemoguardrails import RailsConfig, LLMRails

# Define rails configuration
rails_config = """
define user ask medical diagnosis
  "What disease do I have?"
  "Can you diagnose me?"
  "What's wrong with me medically?"

define bot refuse diagnosis
  "I cannot provide medical diagnoses. I can only help you report adverse events to the FDA. Please consult a healthcare provider for medical advice."

define flow
  user ask medical diagnosis
  bot refuse diagnosis

define bot response quality
  - Must not include medical advice or diagnoses
  - Must not recommend specific treatments
  - Must reference FDA/ICH guidelines when applicable
  - Must be empathetic and supportive
  - Must encourage users to seek medical care if serious

define hallucination check
  - Cross-reference all drug names with DrugBank
  - Verify MedDRA codes against official terminology
  - Do not invent regulatory requirements
"""

config = RailsConfig.from_content(rails_config)
rails = LLMRails(config, llm=llm)

# Use rails-protected LLM
response = await rails.generate(messages=[{"role": "user", "content": user_input}])
```

---

### Phase 2 Deliverables Checklist

- [ ] Pinecone vector database configured
- [ ] Knowledge base ingested (FDA guidelines, drug DB, etc.)
- [ ] Conversational triage API implemented
- [ ] Auto-narrative generation working
- [ ] Medical NER integrated (scispaCy)
- [ ] MedDRA coding automated
- [ ] Naranjo score calculator implemented
- [ ] Guardrails configured (NeMo/Guardrails AI)
- [ ] Human-in-the-loop review queue built
- [ ] LLM ops monitoring (LangSmith/Helicone)
- [ ] Cost tracking dashboard
- [ ] FDA compliance validation
- [ ] End-to-end testing with sample cases

---

## Phase 3: SMART on FHIR Implementation

### Overview
Enable EHR data pre-filling through SMART on FHIR integration.

---

### Step 3.1: HAPI FHIR Server Setup

**docker-compose.yml**:
```yaml
version: '3.8'
services:
  fhir-server:
    image: hapiproject/hapi:latest
    ports:
      - "8080:8080"
    environment:
      - spring.datasource.url=jdbc:postgresql://postgres:5432/hapi
      - spring.datasource.username=admin
      - spring.datasource.password=admin
      - hapi.fhir.fhir_version=R4
      - hapi.fhir.subscription.resthook_enabled=true
    depends_on:
      - postgres
  
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=hapi
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=admin
    volumes:
      - fhir-data:/var/lib/postgresql/data

volumes:
  fhir-data:
```

```bash
docker-compose up -d
# FHIR server available at http://localhost:8080/fhir
```

---

### Step 3.2: SMART Configuration

**smart-configuration.json** (served at `/.well-known/smart-configuration`):
```json
{
  "authorization_endpoint": "https://auth.tfnv-protocol.com/realms/tfnv/protocol/openid-connect/auth",
  "token_endpoint": "https://auth.tfnv-protocol.com/realms/tfnv/protocol/openid-connect/token",
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post"],
  "registration_endpoint": "https://auth.tfnv-protocol.com/realms/tfnv/clients-registrations/openid-connect",
  "scopes_supported": [
    "openid",
    "profile",
    "launch",
    "launch/patient",
    "patient/*.read",
    "patient/Patient.read",
    "patient/MedicationStatement.read",
    "patient/AllergyIntolerance.read",
    "patient/Condition.read",
    "patient/Observation.read",
    "user/*.read"
  ],
  "response_types_supported": ["code"],
  "capabilities": [
    "launch-ehr",
    "launch-standalone",
    "client-public",
    "client-confidential-symmetric",
    "context-ehr-patient",
    "context-standalone-patient",
    "sso-openid-connect"
  ]
}
```

---

### Step 3.3: SMART App Launch

**fhir.controller.ts**:
```typescript
import { Controller, Get, Query, Res } from '@nestjs/common';
import { Response } from 'express';
import axios from 'axios';

@Controller('fhir')
export class FhirController {
  @Get('launch')
  async launch(@Query('iss') iss: string, @Query('launch') launch: string, @Res() res: Response) {
    // Step 1: Fetch SMART configuration from EHR
    const smartConfig = await axios.get(`${iss}/.well-known/smart-configuration`);
    
    // Step 2: Build authorization URL
    const authUrl = new URL(smartConfig.data.authorization_endpoint);
    authUrl.searchParams.append('response_type', 'code');
    authUrl.searchParams.append('client_id', process.env.SMART_CLIENT_ID);
    authUrl.searchParams.append('redirect_uri', process.env.SMART_REDIRECT_URI);
    authUrl.searchParams.append('scope', 'launch patient/*.read');
    authUrl.searchParams.append('state', this.generateState());
    authUrl.searchParams.append('aud', iss);
    authUrl.searchParams.append('launch', launch);
    
    // Step 3: Redirect to EHR authorization page
    res.redirect(authUrl.toString());
  }

  @Get('callback')
  async callback(@Query('code') code: string, @Query('state') state: string) {
    // Step 4: Exchange authorization code for access token
    const tokenResponse = await axios.post(
      process.env.TOKEN_ENDPOINT,
      new URLSearchParams({
        grant_type: 'authorization_code',
        code: code,
        redirect_uri: process.env.SMART_REDIRECT_URI,
        client_id: process.env.SMART_CLIENT_ID,
        client_secret: process.env.SMART_CLIENT_SECRET,
      }),
      {
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      }
    );

    const { access_token, patient, expires_in } = tokenResponse.data;

    // Step 5: Fetch patient data from FHIR server
    const patientData = await this.fetchPatientContext(access_token, patient);

    // Step 6: Pre-fill AE report form
    return {
      prefillData: patientData,
      accessToken: access_token,
      expiresIn: expires_in,
    };
  }

  private async fetchPatientContext(accessToken: string, patientId: string) {
    const baseUrl = process.env.FHIR_BASE_URL;

    // Fetch Patient resource
    const patient = await this.fetchFhirResource(baseUrl, 'Patient', patientId, accessToken);

    // Fetch MedicationStatement resources
    const medications = await this.fetchFhirResource(
      baseUrl,
      'MedicationStatement',
      null,
      accessToken,
      { patient: patientId }
    );

    // Fetch AllergyIntolerance resources
    const allergies = await this.fetchFhirResource(
      baseUrl,
      'AllergyIntolerance',
      null,
      accessToken,
      { patient: patientId }
    );

    // Fetch Condition resources (medical history)
    const conditions = await this.fetchFhirResource(
      baseUrl,
      'Condition',
      null,
      accessToken,
      { patient: patientId }
    );

    return {
      patient: this.mapPatientResource(patient),
      medications: this.mapMedicationResources(medications.entry || []),
      allergies: this.mapAllergyResources(allergies.entry || []),
      conditions: this.mapConditionResources(conditions.entry || []),
    };
  }

  private async fetchFhirResource(
    baseUrl: string,
    resourceType: string,
    id: string | null,
    accessToken: string,
    params: any = {}
  ) {
    const url = id ? `${baseUrl}/${resourceType}/${id}` : `${baseUrl}/${resourceType}`;
    
    const response = await axios.get(url, {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        Accept: 'application/fhir+json',
      },
      params,
    });

    return response.data;
  }

  private mapPatientResource(patient: any) {
    return {
      id: patient.id,
      mrn: patient.identifier?.[0]?.value,
      name: `${patient.name?.[0]?.given?.[0]} ${patient.name?.[0]?.family}`,
      dob: patient.birthDate,
      gender: patient.gender,
      phone: patient.telecom?.find(t => t.system === 'phone')?.value,
      email: patient.telecom?.find(t => t.system === 'email')?.value,
    };
  }

  private mapMedicationResources(entries: any[]) {
    return entries.map(entry => {
      const med = entry.resource;
      return {
        name: med.medicationCodeableConcept?.text || med.medicationReference?.display,
        code: med.medicationCodeableConcept?.coding?.[0]?.code,
        dosage: med.dosage?.[0]?.text,
        startDate: med.effectivePeriod?.start,
        status: med.status,
      };
    });
  }

  private mapAllergyResources(entries: any[]) {
    return entries.map(entry => {
      const allergy = entry.resource;
      return {
        substance: allergy.code?.text,
        severity: allergy.criticality,
        reaction: allergy.reaction?.[0]?.manifestation?.[0]?.text,
      };
    });
  }

  private mapConditionResources(entries: any[]) {
    return entries.map(entry => {
      const condition = entry.resource;
      return {
        name: condition.code?.text,
        onsetDate: condition.onsetDateTime,
        status: condition.clinicalStatus?.coding?.[0]?.code,
      };
    });
  }

  private generateState(): string {
    return Math.random().toString(36).substring(2, 15);
  }
}
```

---

### Step 3.4: Write Back to EHR (AdverseEvent Resource)

**create-adverse-event.ts**:
```typescript
async createAdverseEventInEhr(aeReport: any, accessToken: string) {
  const adverseEvent = {
    resourceType: 'AdverseEvent',
    identifier: [
      {
        system: 'https://tfnv-protocol.com/ae-reports',
        value: aeReport.id,
      },
    ],
    status: 'completed',
    actuality: 'actual',
    category: [
      {
        coding: [
          {
            system: 'http://terminology.hl7.org/CodeSystem/adverse-event-category',
            code: 'medication-mishap',
            display: 'Medication Mishap',
          },
        ],
      },
    ],
    subject: {
      reference: `Patient/${aeReport.patient_id}`,
    },
    date: aeReport.event_date,
    detected: aeReport.created_at,
    recordedDate: aeReport.created_at,
    seriousness: {
      coding: [
        {
          system: 'http://terminology.hl7.org/CodeSystem/adverse-event-seriousness',
          code: this.mapSeverityToSeriousness(aeReport.severity),
        },
      ],
    },
    outcome: {
      coding: [
        {
          system: 'http://terminology.hl7.org/CodeSystem/adverse-event-outcome',
          code: aeReport.event_outcome,
        },
      ],
    },
    suspectEntity: aeReport.suspect_drugs.map(drug => ({
      instance: {
        reference: `Medication/${drug.code}`,
        display: drug.name,
      },
      causality: {
        assessment: {
          text: aeReport.causality_assessment,
        },
      },
    })),
  };

  // POST to FHIR server
  const response = await axios.post(
    `${process.env.FHIR_BASE_URL}/AdverseEvent`,
    adverseEvent,
    {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        'Content-Type': 'application/fhir+json',
      },
    }
  );

  return response.data;
}
```

---

### Phase 3 Deliverables Checklist

- [ ] HAPI FHIR server deployed
- [ ] SMART configuration endpoint published
- [ ] OAuth 2.0 authorization flow implemented
- [ ] SMART App Launch (standalone & embedded) working
- [ ] Patient data pre-filling functional
- [ ] Medication, allergy, condition fetching
- [ ] FHIR AdverseEvent resource write-back
- [ ] Multi-EHR vendor testing (Epic Sandbox, Cerner Sandbox)
- [ ] HIPAA compliance audit
- [ ] Error handling and fallbacks
- [ ] Documentation for EHR integration partners

---

## Phase 4: Gamification for Good Implementation

### Overview
Build engagement through gamification mechanics and charitable micro-donations.

---

### Step 4.1: Point System & Levels

**gamification.service.ts**:
```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UserGameProfile, PointTransaction } from './entities';
import { Redis } from 'ioredis';

@Injectable()
export class GamificationService {
  constructor(
    @InjectRepository(UserGameProfile)
    private profileRepo: Repository<UserGameProfile>,
    @InjectRepository(PointTransaction)
    private transactionRepo: Repository<PointTransaction>,
    private redis: Redis,
  ) {}

  private readonly POINT_RULES = {
    REPORT_SUBMISSION: 100,
    COMPLETE_DETAILS: 50,
    FOLLOW_UP_RESPONSE: 25,
    HELP_ANOTHER_USER: 15,
    STREAK_BONUS: 10,
    FIRST_REPORT_BONUS: 200,
  };

  private readonly LEVEL_THRESHOLDS = {
    BRONZE: 0,
    SILVER: 500,
    GOLD: 2000,
    PLATINUM: 5000,
    DIAMOND: 10000,
  };

  async awardPoints(
    userId: string,
    action: string,
    metadata: any = {}
  ): Promise<{ newPoints: number; levelUp: boolean; newLevel?: string }> {
    const points = this.POINT_RULES[action] || 0;
    
    if (points === 0) {
      throw new Error(`Unknown action: ${action}`);
    }

    // Record transaction
    await this.transactionRepo.save({
      user_id: userId,
      points,
      action,
      metadata,
    });

    // Update profile
    let profile = await this.profileRepo.findOne({ where: { user_id: userId } });
    
    if (!profile) {
      profile = this.profileRepo.create({
        user_id: userId,
        total_points: 0,
        level: 'BRONZE',
      });
    }

    const oldLevel = profile.level;
    profile.total_points += points;
    profile.last_activity_date = new Date();

    // Check for level up
    const newLevel = this.calculateLevel(profile.total_points);
    const levelUp = newLevel !== oldLevel;
    if (levelUp) {
      profile.level = newLevel;
    }

    // Update streak
    profile.streak_days = this.calculateStreak(profile);

    await this.profileRepo.save(profile);

    // Update Redis leaderboard
    await this.redis.zadd('leaderboard:global', profile.total_points, userId);

    return {
      newPoints: profile.total_points,
      levelUp,
      newLevel: levelUp ? newLevel : undefined,
    };
  }

  private calculateLevel(points: number): string {
    if (points >= this.LEVEL_THRESHOLDS.DIAMOND) return 'DIAMOND';
    if (points >= this.LEVEL_THRESHOLDS.PLATINUM) return 'PLATINUM';
    if (points >= this.LEVEL_THRESHOLDS.GOLD) return 'GOLD';
    if (points >= this.LEVEL_THRESHOLDS.SILVER) return 'SILVER';
    return 'BRONZE';
  }

  private calculateStreak(profile: UserGameProfile): number {
    const today = new Date();
    const lastActivity = profile.last_activity_date;
    
    if (!lastActivity) return 1;

    const diffDays = Math.floor((today.getTime() - lastActivity.getTime()) / (1000 * 60 * 60 * 24));
    
    if (diffDays === 0) {
      // Same day
      return profile.streak_days;
    } else if (diffDays === 1) {
      // Consecutive day
      return profile.streak_days + 1;
    } else {
      // Streak broken
      return 1;
    }
  }

  async getLeaderboard(limit: number = 10): Promise<any[]> {
    // Fetch top users from Redis sorted set
    const userIds = await this.redis.zrevrange('leaderboard:global', 0, limit - 1, 'WITHSCORES');
    
    const leaderboard = [];
    for (let i = 0; i < userIds.length; i += 2) {
      const userId = userIds[i];
      const points = parseInt(userIds[i + 1], 10);
      
      const profile = await this.profileRepo.findOne({ where: { user_id: userId } });
      
      leaderboard.push({
        rank: i / 2 + 1,
        userId: userId,
        username: profile?.user?.username || 'Anonymous',
        points,
        level: profile?.level,
      });
    }

    return leaderboard;
  }
}
```

---

### Step 4.2: Badge System

**badges.config.ts**:
```typescript
export const BADGES = [
  {
    id: 'first-reporter',
    name: 'First Reporter',
    description: 'Submitted your first adverse event report',
    icon: '🏅',
    criteria: { reports_count: 1 },
  },
  {
    id: 'detail-detective',
    name: 'Detail Detective',
    description: 'Submitted a report with complete medical details',
    icon: '🔍',
    criteria: { complete_details: true },
  },
  {
    id: 'safety-champion',
    name: 'Safety Champion',
    description: 'Submitted 10 adverse event reports',
    icon: '🏆',
    criteria: { reports_count: 10 },
  },
  {
    id: 'streak-master',
    name: 'Streak Master',
    description: 'Maintained a 7-day activity streak',
    icon: '🔥',
    criteria: { streak_days: 7 },
  },
  {
    id: 'generous-giver',
    name: 'Generous Giver',
    description: 'Made your first charitable donation',
    icon: '💝',
    criteria: { donations_count: 1 },
  },
  {
    id: 'community-helper',
    name: 'Community Helper',
    description: 'Helped 5 other patients in the forum',
    icon: '🤝',
    criteria: { forum_helps: 5 },
  },
];

@Injectable()
export class BadgeService {
  async checkAndAwardBadges(userId: string): Promise<string[]> {
    const profile = await this.getProfile(userId);
    const awardedBadges = [];

    for (const badge of BADGES) {
      // Check if already earned
      if (profile.badges.includes(badge.id)) {
        continue;
      }

      // Check criteria
      if (this.meetsCriteria(profile, badge.criteria)) {
        // Award badge
        profile.badges.push(badge.id);
        awardedBadges.push(badge.id);
        
        // Send notification
        await this.notificationService.send(userId, {
          type: 'BADGE_UNLOCKED',
          badge: badge,
        });
      }
    }

    if (awardedBadges.length > 0) {
      await this.profileRepo.save(profile);
    }

    return awardedBadges;
  }

  private meetsCriteria(profile: any, criteria: any): boolean {
    return Object.entries(criteria).every(([key, value]) => {
      return profile[key] >= value;
    });
  }
}
```

---

### Step 4.3: Micro-Donation System

**donation.service.ts**:
```typescript
import { Injectable } from '@nestjs/common';
import Stripe from 'stripe';

@Injectable()
export class DonationService {
  private stripe: Stripe;
  private readonly CONVERSION_RATE = 100; // 100 points = $1

  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
      apiVersion: '2023-10-16',
    });
  }

  async convertPointsToDonation(
    userId: string,
    pointsToSpend: number,
    charityId: string
  ): Promise<{ donationId: string; amount: number }> {
    const profile = await this.profileRepo.findOne({ where: { user_id: userId } });

    if (profile.total_points < pointsToSpend) {
      throw new Error('Insufficient points');
    }

    const donationAmount = pointsToSpend / this.CONVERSION_RATE; // in dollars

    // Deduct points
    profile.total_points -= pointsToSpend;
    await this.profileRepo.save(profile);

    // Create donation record
    const donation = await this.donationRepo.save({
      user_id: userId,
      charity_id: charityId,
      amount: donationAmount,
      points_spent: pointsToSpend,
      status: 'pending',
    });

    // Process payment via Stripe (TFNV pays the charity)
    const charity = await this.charityRepo.findOne({ where: { id: charityId } });

    const paymentIntent = await this.stripe.paymentIntents.create({
      amount: Math.round(donationAmount * 100), // Convert to cents
      currency: 'usd',
      description: `Donation to ${charity.name} from TFNV user`,
      metadata: {
        user_id: userId,
        charity_id: charityId,
        donation_id: donation.id,
      },
      transfer_data: {
        destination: charity.stripe_account_id, // Charity's Stripe Connect account
      },
    });

    // Update donation status
    donation.stripe_payment_intent_id = paymentIntent.id;
    donation.status = 'completed';
    await this.donationRepo.save(donation);

    // Update charity total
    charity.total_received += donationAmount;
    await this.charityRepo.save(charity);

    // Update user's lifetime donations
    profile.lifetime_donations += donationAmount;
    await this.profileRepo.save(profile);

    // Generate tax receipt (for US users)
    if (donationAmount >= 250) {
      await this.generateTaxReceipt(userId, donation);
    }

    return {
      donationId: donation.id,
      amount: donationAmount,
    };
  }

  private async generateTaxReceipt(userId: string, donation: any) {
    // Generate PDF tax receipt (IRS compliant)
    const PDFDocument = require('pdfkit');
    const doc = new PDFDocument();

    doc.fontSize(20).text('Tax-Deductible Donation Receipt', { align: 'center' });
    doc.moveDown();
    doc.fontSize(12).text(`Date: ${new Date().toLocaleDateString()}`);
    doc.text(`Donation Amount: $${donation.amount.toFixed(2)}`);
    doc.text(`Charity: ${donation.charity.name}`);
    doc.text(`EIN: ${donation.charity.ein}`);
    doc.text(`Donor ID: ${userId.substring(0, 8)}...`);
    doc.moveDown();
    doc.text('No goods or services were provided in exchange for this donation.');

    // Save to S3
    const buffer = await this.generatePdfBuffer(doc);
    const key = `tax-receipts/${donation.id}.pdf`;
    await this.s3.upload({ Bucket: 'tfnv-receipts', Key: key, Body: buffer }).promise();

    // Email to user
    await this.emailService.send(userId, {
      subject: 'Tax Receipt for Your Donation',
      body: 'Attached is your tax-deductible donation receipt.',
      attachments: [{ filename: 'receipt.pdf', path: key }],
    });
  }
}
```

---

### Step 4.4: Impact Dashboard

**impact-dashboard.tsx**:
```typescript
import React, { useEffect, useState } from 'react';
import { Card, Progress, Statistic } from 'antd';

interface ImpactStats {
  totalDonations: number;
  totalDonors: number;
  charities: Array<{
    name: string;
    amount: number;
    impact: string; // e.g., "Funded 100 vaccine doses"
  }>;
  userContribution: number;
  userRank: number;
}

export const ImpactDashboard: React.FC = () => {
  const [stats, setStats] = useState<ImpactStats | null>(null);

  useEffect(() => {
    fetch('/api/v1/gamification/impact-dashboard')
      .then(res => res.json())
      .then(setStats);
  }, []);

  if (!stats) return <div>Loading...</div>;

  return (
    <div className="impact-dashboard">
      <h1>💝 Impact Dashboard</h1>
      <p>See the real-world impact of your adverse event reports!</p>

      <div className="stats-grid">
        <Card>
          <Statistic
            title="Total Donations"
            value={stats.totalDonations}
            prefix="$"
            precision={2}
          />
        </Card>
        
        <Card>
          <Statistic
            title="Community Donors"
            value={stats.totalDonors}
          />
        </Card>

        <Card>
          <Statistic
            title="Your Contribution"
            value={stats.userContribution}
            prefix="$"
            precision={2}
          />
        </Card>

        <Card>
          <Statistic
            title="Your Rank"
            value={stats.userRank}
            suffix={`/ ${stats.totalDonors}`}
          />
        </Card>
      </div>

      <h2>Charity Breakdown</h2>
      {stats.charities.map(charity => (
        <Card key={charity.name} style={{ marginBottom: 16 }}>
          <h3>{charity.name}</h3>
          <Progress
            percent={(charity.amount / stats.totalDonations) * 100}
            format={() => `$${charity.amount.toFixed(2)}`}
          />
          <p className="impact-description">{charity.impact}</p>
        </Card>
      ))}

      <div className="testimonial">
        <h3>🌟 Success Story</h3>
        <blockquote>
          "Thanks to TFNV community donations, we were able to provide emergency medical supplies
          to 500 families affected by the recent disaster. Your adverse event reports are making
          a real difference!" - Doctors Without Borders
        </blockquote>
      </div>
    </div>
  );
};
```

---

### Phase 4 Deliverables Checklist

- [ ] Point system implemented with all rules
- [ ] Level calculation and progression
- [ ] Badge system with 6+ badges
- [ ] Streak tracking and bonuses
- [ ] Leaderboard (Redis-backed)
- [ ] Stripe Connect integration
- [ ] Charity verification (GuideStar API)
- [ ] Donation conversion (points to $)
- [ ] Tax receipt generation (PDF)
- [ ] Impact dashboard with visualizations
- [ ] Real-time notifications (Socket.io)
- [ ] Anonymous forum/community
- [ ] Moderation tools
- [ ] Testing with real charities
- [ ] Legal compliance (charity regulations)

---

## Cross-Phase Integration

### End-to-End Flow

1. **User starts AE report** → Trust layer verifies identity (BIMI/WhatsApp)
2. **EHR data fetch** → SMART on FHIR pre-fills form
3. **Conversational triage** → GenAI RAG guides through questions
4. **Auto-narrative** → LLM generates ICH E2B narrative
5. **Review & submit** → Human validation, FDA submission
6. **Gamification** → Award points, check badges, offer donation
7. **Impact tracking** → Update charity dashboard

### Deployment Order

1. **Infrastructure** (Kubernetes, databases, Redis)
2. **Phase 1** (Trust Layer)
3. **Phase 3** (FHIR - needed for pre-filling)
4. **Phase 2** (GenAI RAG - uses pre-filled data)
5. **Phase 4** (Gamification - rewards for completed reports)
6. **Observability** (Monitoring, logging, alerting)

---

## Testing Strategy

### Unit Tests
- Each service has >80% code coverage
- Mock external APIs (OpenAI, FHIR, Stripe)

### Integration Tests
- Test service-to-service communication
- Test database transactions
- Test message queue processing

### E2E Tests
- Simulate full AE report submission
- Test EHR integration with Epic Sandbox
- Test donation flow end-to-end

### Security Tests
- Penetration testing (annual)
- Vulnerability scanning (Snyk, CodeQL)
- HIPAA compliance audit

### Performance Tests
- Load testing (10,000 concurrent users)
- Stress testing (API rate limits)
- LLM cost optimization testing

---

*Document Version: 1.0*  
*Last Updated: 2026-01-08*  
*Owner: HealthTech Implementation Team*
