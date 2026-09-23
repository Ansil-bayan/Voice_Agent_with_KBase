# 🎙️ Voice Agent with Knowledge Base

An AI-powered voice agent for **business loan qualification**, built with [Vapi](https://vapi.ai), [ChromaDB](https://www.trychroma.com/), and [FastAPI](https://fastapi.tiangolo.com/). The agent conducts real-time phone conversations to qualify leads, answer policy questions from a curated knowledge base, and escalate to human specialists when needed.

---

## ✨ Features

- **Automated Lead Qualification** — Collects revenue, time in business, credit score, and desired loan amount through natural voice conversation
- **Knowledge-Grounded Answers** — Retrieves accurate policy, eligibility, and rate information from a vector database instead of hallucinating
- **Multi-Source Data Pipeline** — Scrapes web pages and extracts PDF content (with table support) to build the knowledge base
- **PII Redaction** — Automatically detects and redacts SSNs, credit cards, emails, phone numbers, names, and addresses using regex + spaCy NER
- **Near-Deduplication** — Removes duplicate content using MinHash LSH (Jaccard similarity ≥ 0.85)
- **Human Escalation** — Seamlessly transfers callers to a human loan specialist via Vapi's native call transfer
- **CRM Integration** — Submits qualified lead data to a webhook endpoint for downstream processing

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA PIPELINE                                │
│                                                                     │
│  Web Pages ──┐                                                      │
│              ├──▶ parse.py ──▶ boilerplate_removal.py ──▶ ingest.py │
│  PDF Docs ───┘    (Extract,     (Clean noise,              (Embed & │
│                    PII Redact,   headers,                   index   │
│                    Deduplicate)  footers)                   into    │
│                                                            ChromaDB)│
└────────────────────────────────────┬────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       RUNTIME SYSTEM                                │
│                                                                     │
│  Caller ◄──▶ Vapi Voice Agent ◄──▶ vapi_server.py (FastAPI)         │
│              (GPT-4o-mini +          │                               │
│               Deepgram STT +         ▼                              │
│               Cartesia TTS)     ChromaDB Vector Search              │
│                                 (all-MiniLM-L6-v2)                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| **Voice Platform** | [Vapi](https://vapi.ai) |
| **LLM** | OpenAI GPT-4o-mini (via Vapi) |
| **Speech-to-Text** | Deepgram Nova 2 (via Vapi) |
| **Text-to-Speech** | Cartesia (via Vapi) |
| **Vector Database** | ChromaDB (persistent, local) |
| **Embedding Model** | `all-MiniLM-L6-v2` (Sentence Transformers) |
| **API Framework** | FastAPI + Uvicorn |
| **NLP / NER** | spaCy (`en_core_web_sm`) |
| **Deduplication** | MinHash LSH via `datasketch` |
| **PDF Extraction** | pdfplumber + pypdf (fallback) |
| **Web Scraping** | `unstructured` (HTML partitioning) |

---

## 📁 Project Structure

```
.
├── parse.py                                  # Data extraction pipeline (web + PDF → JSON)
├── boilerplate_removal.py                    # Post-extraction noise cleanup
├── ingest.py                                 # Embeds cleaned JSON into ChromaDB
├── setup_vapi.py                             # Registers tools & creates Vapi assistant
├── vapi_server.py                            # FastAPI server handling Vapi tool calls
├── test_kbase_retrieval.py                   # Evaluates vector search retrieval quality
├── check_chromadb_content.py                 # Utility to inspect ChromaDB contents
├── verify_db.py                              # Verifies ChromaDB is populated correctly
├── business_loans_knowledge_base_parsed.json # Raw extracted knowledge base
├── business_loans_knowledge_base_cleaned.json# Cleaned knowledge base (boilerplate removed)
├── test_query&responses.txt                  # Sample retrieval test results
├── Mock_conversations/                       # Recorded test call audio (.wav)
│   ├── Test 1.wav
│   ├── Test 2.wav
│   └── Test 3.wav
├── requirements.txt                          # Python dependencies
├── .env.example                              # Environment variable template
└── .gitignore
```

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- A [Vapi](https://vapi.ai) account and API key
- A publicly accessible server URL (e.g., via [ngrok](https://ngrok.com)) for Vapi webhooks

### 1. Clone the Repository

```bash
git clone https://github.com/Ansil-bayan/Voice_Agent_with_KBase.git
cd Voice_Agent_with_KBase
```

### 2. Create a Virtual Environment

```bash
python -m venv venv
source venv/bin/activate    # Linux/macOS
venv\Scripts\activate       # Windows
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

> [!NOTE]
> Additional dependencies for the data pipeline (not in `requirements.txt`) include:
> `pdfplumber`, `pypdf`, `pandas`, `spacy`, `datasketch`, `unstructured`, and `tabulate`.
> Install them with:
> ```bash
> pip install pdfplumber pypdf pandas spacy datasketch unstructured tabulate
> python -m spacy download en_core_web_sm
> ```

### 4. Configure Environment Variables

```bash
cp .env.example .env
```

Edit `.env` with your credentials:

```env
VAPI_API_KEY=your_vapi_api_key
SERVER_URL=https://your-server.ngrok.io/query-kb
BUSINESS_ACTION_URL=https://your-server.ngrok.io/record-lead
HUMAN_AGENT_PHONE=+1234567890
```

### 5. Build the Knowledge Base

Run the full data pipeline:

```bash
# Step 1: Extract content from web pages and PDFs
python parse.py

# Step 2: Remove boilerplate noise
python boilerplate_removal.py

# Step 3: Embed and index into ChromaDB
python ingest.py
```

### 6. Verify the Database

```bash
python verify_db.py
```

You should see output like:
```
SUCCESS: 'knowledge_base' collection found with N indexed records.
```

### 7. Start the Server

```bash
python vapi_server.py
```

The FastAPI server starts on `http://0.0.0.0:8000`. Expose it publicly using ngrok:

```bash
ngrok http 8000
```

### 8. Register the Vapi Assistant

```bash
python setup_vapi.py
```

This registers the knowledge base search tool, lead qualification tool, and human transfer tool with Vapi, then creates the voice assistant.

---

## 📡 API Reference

### `POST /query-kb`

Handles knowledge base search requests from Vapi tool calls.

**Request Body** (Vapi webhook format):
```json
{
  "message": {
    "toolCalls": [{
      "id": "call_abc123",
      "function": {
        "name": "search_knowledge_base",
        "arguments": { "query": "What are the eligibility requirements?" }
      }
    }]
  }
}
```

**Response**:
```json
{
  "results": [{
    "toolCallId": "call_abc123",
    "result": "Relevant knowledge base content..."
  }]
}
```

---

## 🧪 Testing

### Retrieval Quality Test

```bash
python test_kbase_retrieval.py
```

Runs 5 predefined test queries across categories (Product, Policy, Qualification, FAQ, Objection) and displays the top retrieved chunk with distance scores.

### Database Inspection

```bash
python check_chromadb_content.py   # Peek at raw ChromaDB records
python verify_db.py                # Verify collection exists & test a sample query
```

### Voice Call Tests

The `Mock_conversations/` directory contains 3 recorded test calls (`Test 1.wav`, `Test 2.wav`, `Test 3.wav`) demonstrating live agent interactions.

---

## 🔄 Data Pipeline Details

The knowledge base is built through a 3-stage pipeline:

| Stage | Script | Input | Output |
|---|---|---|---|
| **1. Extract** | `parse.py` | Web URLs + PDF URLs | `business_loans_knowledge_base.json` |
| **2. Clean** | `boilerplate_removal.py` | Raw JSON | `business_loans_knowledge_base_cleaned.json` |
| **3. Index** | `ingest.py` | Cleaned JSON | ChromaDB (`./chroma_db/`) |

### Extraction Features (parse.py)

- **Web scraping** with `unstructured` HTML partitioning and cookie/nav filtering
- **PDF extraction** with pdfplumber (primary) + pypdf (fallback), including table extraction
- **PII redaction** via regex patterns (SSN, credit card, email, phone, EIN) and spaCy NER (names, addresses)
- **Near-deduplication** using MinHash LSH with Jaccard similarity threshold of 0.85
- **Content validation** rejecting chunks under 40 characters or with >30% non-ASCII noise

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Commit your changes (`git commit -m 'Add your feature'`)
4. Push to the branch (`git push origin feature/your-feature`)
5. Open a Pull Request

---

## 📄 License

This project is open source. See the [LICENSE](LICENSE) file for details.
