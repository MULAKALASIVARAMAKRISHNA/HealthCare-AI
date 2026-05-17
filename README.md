# 🏥 Healthcare AI Assistant

A production-ready, RAG-powered healthcare AI assistant that answers clinical and operational questions from a curated knowledge base — with zero hallucinations, structured JSON responses, and a live public API.

---

## 📐 Architecture Overview

```
User Question
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│              Agentic Intent Router                      │
│   Appointment query? → mock_check_available_slots()     │
│   Knowledge query?   → RAG Pipeline                    │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  Hybrid Retriever                       │
│   BM25 (lexical) + Dense Semantic → RRF Fusion          │
│   Embedding model: all-MiniLM-L6-v2 (SentenceTransform)│
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│               Multi-Provider LLM Chain                  │
│   Gemini 2.5 Flash → Flash-Lite → Groq Llama-3.3 70B   │
│                    → Llama-3.1 8B → Sarvam AI           │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│          Pydantic ClinicalResponse Schema               │
│   { analysis, questions[2], confidence_score }          │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
              FastAPI + Cloudflare Tunnel
              (Public HTTPS endpoint)
```

| Layer | Technology | Purpose |
|:---|:---|:---|
| **Document Ingestion** | PyMuPDF + pytesseract OCR | Parse PDF, TXT, CSV, MD files |
| **Chunking** | 800-word windows, 100-word overlap | Preserve context across boundaries |
| **Embeddings** | `all-MiniLM-L6-v2` (SentenceTransformers) | Dense semantic vectors |
| **Retrieval** | BM25 lexical + Dense semantic → RRF Fusion | Hybrid best-of-both retrieval |
| **LLM Chain** | Gemini 2.5 Flash → Flash-Lite → Groq Llama-3.3 70B → Llama-3.1 8B → Sarvam AI | Multi-provider fallback |
| **Token Budget** | Dynamic classifier: `simple=300`, `complex=600`, `outscope=200` | Cost-efficient inference |
| **Output Schema** | Pydantic `ClinicalResponse` (JSON enforced) | Structured, auditable responses |
| **Agentic Tool** | `mock_check_available_slots(dept, date)` router | Intent-based workflow branching |
| **API** | FastAPI + Cloudflare Tunnel (public HTTPS) | Production-ready endpoints |

---

## 📁 Project Structure

```
healthcare-ai-assistant/
│
├── Health_Assistant.ipynb      # Main notebook — all cells run top-to-bottom
│
├── data/                       # Synthetic healthcare documents (no real PHI)
│   ├── medication_refill_policy.txt
│   ├── telehealth_consultation_guidelines.txt
│   ├── insurance_eligibility_faq.txt
│   ├── appointment_scheduling_policy.txt
│   ├── hipaa_privacy_guidelines.txt
│   ├── discharge_instructions.txt
│   └── global_health_data.txt
│
├── requirements.txt            # Python dependencies
├── Dockerfile                  # Container build (optional local run)
├── docker-compose.yml          # Multi-service orchestration (optional)
└── README.md                   # This file
```

---

## 🔑 API Keys Required

| Secret Name | Provider | Get It At |
|:---|:---|:---|
| `GEMINI_API_KEY` | Google AI Studio | [aistudio.google.com](https://aistudio.google.com/app/apikey) |
| `GROQ_API_KEY` | Groq Console | [console.groq.com](https://console.groq.com/keys) |
| `SARVAM_API_KEY` | Sarvam AI *(optional fallback)* | [dashboard.sarvam.ai](https://dashboard.sarvam.ai) |

> **Minimum required:** at least one of `GEMINI_API_KEY` or `GROQ_API_KEY`.

---

## 🚀 Quickstart — Google Colab (Recommended)

### Step 1 — Open the Notebook

Upload `Health_Assistant.ipynb` to [Google Colab](https://colab.research.google.com) or open it directly from GitHub.

### Step 2 — Add API Keys

1. Click the 🔑 **Secrets** icon in the left sidebar.
2. Add the following secrets (toggle "Notebook access" ON for each):

| Secret Name | Value |
|:---|:---|
| `GEMINI_API_KEY` | `AIza...` |
| `GROQ_API_KEY` | `gsk_...` |
| `SARVAM_API_KEY` | `sk-...` *(optional)* |

### Step 3 — Run All Cells

```
Runtime → Run all  (Ctrl + F9)
```

Cells must be run **in order** (top to bottom) on first launch. After Cell 11 initialises the assistant, Cells 12–15 can be re-run independently.

### Step 4 — Get the Public API URL

After Cell 15 completes, you will see output like:

```
===========================================================
  MINDBOWSER HACKATHON — LIVE PUBLIC API
===========================================================
  BASE URL : https://abc-def-ghi.trycloudflare.com
  HEALTH   : https://abc-def-ghi.trycloudflare.com/health
  ASK      : https://abc-def-ghi.trycloudflare.com/ask
  INGEST   : https://abc-def-ghi.trycloudflare.com/ingest
  SWAGGER  : https://abc-def-ghi.trycloudflare.com/docs
===========================================================
```

Copy the `BASE URL` into your browser's Swagger UI (`/docs`) to test all endpoints interactively.

---

## 🐳 Docker Setup (Optional Local Run)

### Build and Run

```bash
docker build -t healthcare-ai-assistant .
docker run -p 8000:8000 \
  -e GOOGLE_API_KEY=AIza... \
  -e GROQ_API_KEY=gsk_... \
  healthcare-ai-assistant
```

### With Docker Compose

```bash
# Copy and fill in your keys
cp .env.example .env

docker-compose up --build
```

The API will be available at `http://localhost:8000`.

---

## 🔌 API Reference

### `GET /health`

Returns server health status and current corpus chunk count.

```bash
curl https://<your-tunnel-url>/health
```

**Response:**
```json
{
  "status": "healthy",
  "version": "4.2",
  "providers_indexed": 5,
  "corpus_chunks": 7
}
```

---

### `POST /ask`

Submit a natural-language question and receive a structured JSON answer grounded in the knowledge base.

```bash
curl -X POST https://<your-tunnel-url>/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Can a patient request a medication refill through telehealth?"}'
```

**Response:**
```json
{
  "answer": "Yes, patients may request refills for routine medications during telehealth follow-up visits if the prescribing physician determines it is clinically appropriate. However, controlled substances (Schedule II–V) cannot be refilled via telehealth — an in-person visit with a DEA-registered provider is required.",
  "sources": [
    {
      "document": "medication_refill_policy.txt",
      "chunk": "Telehealth follow-up: routine medications may be renewed if the prescribing physician determines it is clinically appropriate..."
    }
  ],
  "confidence": "high",
  "follow_up_questions": [
    "Is your medication classified as a controlled substance (Schedule II–V)?",
    "Have you already established care with a prescribing physician at this practice?"
  ],
  "metadata": {
    "llm_used": "gemini-2.5-flash",
    "token_tier": "simple",
    "token_budget_used": 300
  }
}
```

**Out-of-scope response (no hallucination):**
```json
{
  "answer": "I could not find this information in the provided documents.",
  "sources": [],
  "confidence": "low"
}
```

---

### `POST /ingest`

Upload a new document to extend the knowledge base at runtime. Supported formats: `.pdf`, `.txt`, `.csv`, `.md`.

```bash
curl -X POST https://<your-tunnel-url>/ingest \
  -F "file=@/path/to/your_document.pdf"
```

**Response:**
```json
{
  "message": "Successfully ingested 'your_document.pdf'. Extracted 4,231 characters → 6 chunks.",
  "chunks_created": 6,
  "total_corpus_chunks": 13
}
```

---

## 🧠 Prompt Engineering Strategy

The system prompt enforces strict grounding rules to ensure safe, auditable responses:

```
You are a professional healthcare information assistant.

RULES:
1. Answer ONLY from the retrieved context provided below.
2. If the context does not contain sufficient information, respond:
   "I could not find this information in the provided documents."
3. Never guess, infer, or hallucinate facts not present in the context.
4. Never provide a direct medical diagnosis or prescribe treatments.
5. Keep responses clear, professional, and concise.
6. If asked "who are you?", introduce yourself as the Healthcare AI Assistant.

RESPONSE FORMAT (strict JSON):
{
  "analysis": "<2-sentence answer grounded in context>",
  "questions": ["<Follow-up question 1>", "<Follow-up question 2>"],
  "confidence_score": <0.0 to 1.0>
}

CONTEXT:
{retrieved_chunks}

QUESTION:
{user_question}
```

**Key design decisions:**
- `confidence_score ≤ 0.2` signals an out-of-scope or unanswerable query.
- Exactly 2 follow-up questions are always generated to guide the patient.
- Pydantic schema validation catches any malformed LLM output before it reaches the API response.

---

## 🤖 Agentic Workflow

The assistant uses a lightweight intent router before invoking the RAG pipeline:

```python
if "book" or "appointment" or "schedule" or "slot" in query:
    → mock_check_available_slots(department, date)
    → Returns: "Available slots for Cardiology on Monday: 10:00 AM, 2:30 PM, 4:00 PM"
else:
    → RAG pipeline
```

**Example:**

> **User:** "Can I book a cardiology appointment for Monday?"
>
> **Assistant:** "I can check mock appointment availability. Available slots for Cardiology on Monday are: 10:00 AM, 2:30 PM, and 4:00 PM. Would you like me to confirm one of these?"

This does not connect to a real scheduling system — it is a mock tool demonstrating intent-based workflow branching.

---

## 📁 Synthetic Dataset

All documents are **fully synthetic** — no real patient data or PHI is used at any point.

| File | Topic |
|:---|:---|
| `medication_refill_policy.txt` | Controlled substance rules, online portal workflow, early refill denial |
| `telehealth_consultation_guidelines.txt` | Eligible services, technology requirements, insurance parity |
| `insurance_eligibility_faq.txt` | Accepted plans, copays, prior authorisation, sliding-scale fees |
| `appointment_scheduling_policy.txt` | Booking channels, specialist referrals, cancellation/no-show policy |
| `hipaa_privacy_guidelines.txt` | Patient rights, PHI access, HHS complaint process, civil penalties |
| `discharge_instructions.txt` | Activity restrictions, ER warning signs, medication adherence, follow-up |
| `global_health_data.txt` | WHO malaria statistics (public data, synthetic format) |

---

## 🔁 Notebook Cell Execution Order

| # | Cell Name | Purpose |
|:-:|:---|:---|
| 1 | Install Dependencies | pip install all packages + OCR system deps |
| 2 | API Key Setup | Load keys from Colab Secrets |
| 3 | Imports & Logging | All library imports + logger config |
| 4 | Token Budget & Query Classifier | Dynamic per-query tier classification |
| 5 | Pydantic Output Schema | Strict `ClinicalResponse` JSON enforcement |
| 6 | Synthetic Document Corpus | 7 base healthcare policy documents |
| 7 | Hybrid Retriever | BM25 + Dense → RRF fusion retriever |
| 8 | Multi-Provider LLM Chain | Fallback chain across 5 providers |
| 9 | Prompt Engineering & Message Builder | System prompt + context assembly |
| 10 | HealthcareAssistant Orchestrator | Main class: RAG + agentic routing |
| 11 | Initialise Assistant | Boot assistant (downloads model once, ~90MB) |
| 12 | Interactive Single Query | Test any single question |
| 13 | Batch Test — All Queries | Run all 11 test queries |
| 14 | Token Budget Audit | Pre-flight TPM cost check |
| 15 | FastAPI Server + Cloudflare Tunnel | Live public API via HTTPS |

> **Always run top-to-bottom on first launch.** After Cell 11, Cells 12–15 can be re-run independently.

---

## 🧪 Sample Questions & Expected Responses

| Query | Expected Behaviour |
|:---|:---|
| "Can a patient request a medication refill through telehealth?" | ✅ Answers from `medication_refill_policy.txt` |
| "What are my rights under HIPAA for accessing medical records?" | ✅ Lists 6 rights from `hipaa_privacy_guidelines.txt` |
| "What should I do if I have a fever after discharge?" | ✅ References ER warning signs from `discharge_instructions.txt` |
| "Can I book a cardiology appointment for Monday?" | ✅ Routes to mock scheduling tool |
| "How many malaria deaths occurred globally in 2022?" | ✅ Returns "608,000" from `global_health_data.txt` |
| "What is the stock price of Apple?" | ✅ Returns out-of-scope: "I could not find this information..." |
| "Diagnose my symptoms." | ✅ Refuses to diagnose; redirects to a clinician |

---

## 🔧 Technical Decisions & Trade-offs

### LLM Used
**Primary:** Gemini 2.5 Flash (Google AI Studio — free tier)
**Fallback chain:** Gemini Flash-Lite → Groq Llama-3.3 70B → Groq Llama-3.1 8B → Sarvam AI

*Rationale:* Multi-provider fallback guarantees response availability even if one provider is rate-limited or unavailable. Gemini Flash offers the best cost-quality ratio for healthcare QA tasks.

### Embedding Model
**`all-MiniLM-L6-v2`** (SentenceTransformers, ~90MB)

*Rationale:* Strong semantic retrieval performance with minimal memory footprint. No API key required — runs locally.

### Vector Database / Retrieval
**Hybrid: BM25 + Dense vectors → Reciprocal Rank Fusion (RRF)**

*Rationale:*
- BM25 excels at exact keyword matches (drug names, policy codes, statistics).
- Dense retrieval excels at semantic similarity ("chest pain after surgery" → discharge instructions).
- RRF fusion improves recall without needing per-score calibration.
- In-memory for Colab simplicity; designed to swap to ChromaDB/Pinecone for persistence.

### Prompting Strategy
Strict grounding with Pydantic-enforced output schema. Out-of-scope queries are detected via `confidence_score ≤ 0.2` without a secondary LLM call.

### Agentic Workflow
Lightweight keyword-based intent router (no LLM call overhead). Routes appointment queries to a mock tool and all other queries to the RAG pipeline.

---

## ⚖️ Limitations & Future Improvements

| Limitation | Future Improvement |
|:---|:---|
| In-memory corpus — resets on notebook restart | Persist to ChromaDB / Pinecone for durable storage |
| Mock appointment scheduling tool | Integrate real EHR scheduling API (Epic SMART on FHIR) |
| English-only responses | Multilingual support via Sarvam AI (Hindi, Tamil, Telugu, etc.) |
| No API authentication | Add OAuth2 / API key middleware for production |
| Colab + Cloudflare tunnel (ephemeral) | Deploy to Cloud Run / Railway with a custom domain |
| No unit tests | Add pytest suite covering retrieval, schema validation, and API endpoints |
| No PHI de-identification pipeline | Add Presidio or AWS Comprehend Medical for PII scrubbing |

---

## 🛡️ Healthcare Data Privacy

- **No real patient data or PHI** is used anywhere in this project.
- All documents in `/data` are fully synthetic.
- The `/ingest` endpoint accepts only text-based files; no patient identifiers are stored.
- In a production deployment, all API traffic should be encrypted (HTTPS), access-controlled (OAuth2/JWT), and logged for audit trails in compliance with HIPAA's Technical Safeguard requirements.
- PHI de-identification (per HIPAA Safe Harbor or Expert Determination methods) would be required before ingesting any real clinical documents.

---

## 📦 Dependencies

```txt
langchain
langchain-google-genai
langchain-groq
langchain-openai
rank_bm25
sentence-transformers
pydantic
python-dotenv
pymupdf
python-multipart
pytesseract
pdf2image
uvicorn
fastapi
numpy
```

Install all dependencies:

```bash
pip install -r requirements.txt
```

System dependencies (for OCR):

```bash
apt-get install -y tesseract-ocr poppler-utils
```

---

## 🙋 Author

**SIva Ramakrishna**


---

## 📄 License

This project is submitted as a hackathon assignment. All synthetic documents and source code are original work created for evaluation purposes only.
