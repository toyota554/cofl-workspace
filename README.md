# COFL: Continuous Operational Feedback Loop for D2C Commerce

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docker](https://img.shields.io/badge/Docker-24.0.5+-blue.svg)](https://www.docker.com/)
[![Python](https://img.shields.io/badge/Python-3.11-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110.0-green.svg)](https://fastapi.tiangolo.com/)

> **Reference implementation for:**  
> *"Designing a Continuous Operational Feedback Loop for Direct-to-Consumer Commerce: Integrating Event-Driven Automation and On-Premise Generative AI"*  
> Der-Fa Chen, Yung-Hsing Chen, Bo-Siang Chen — *MDPI Information*, 2026

---

## Overview

COFL is a fully localized, event-driven, closed-loop intelligence system for Direct-to-Consumer (D2C) e-commerce complaint triage and automated response generation. The architecture eliminates PII transmission to cloud endpoints by running a 4-bit quantized LLM entirely on commodity CPU-only edge hardware (~USD 1,640).

### Key Empirical Results

| Metric | COFL (Local) | Cloud GPT-4o | Ratio |
|---|---|---|---|
| E2E Latency | 3.22 s | 2.77 s | 1.16× |
| Macro-F1 (Sentiment) | 0.836 | 0.864 | 96.8% |
| ROUGE-L (Generation) | 0.421 | 0.448 | 94.0% |
| Cost per 1k requests (NT$) | ~0.9 | ~180 | **1/200** |
| 36-month TCO (NT$) | ~133,000 | ~6,490,000 | **48×** |

### Four COFL Stages (Closed-Loop Control Mapping)

```
[External Event] → Sense (n8n Webhook)
                 → Analyze (FastAPI /analyze → Ollama LLM)
                 → Act    (FastAPI /generate → Notification)
                 → Monitor (PostgreSQL event log + daily metrics)
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Bridge Network                     │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │     n8n      │───▶│  FastAPI GW  │───▶│    Ollama    │  │
│  │  Port 5678   │    │  Port 8000   │    │  Port 11434  │  │
│  │ (Workflow    │◀───│ (4 async     │    │ (LLaMA 3 8B  │  │
│  │  Engine)     │    │  workers)    │    │  Q4_K_M)     │  │
│  └──────┬───────┘    └──────────────┘    └──────────────┘  │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────┐                                          │
│  │  PostgreSQL  │                                          │
│  │  Port 5432   │                                          │
│  │ (Event Log + │                                          │
│  │  Metrics)    │                                          │
│  └──────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

### Hardware Specification (Validated Test Bed)

| Component | Specification |
|---|---|
| CPU | Intel Core i5-12400F (6C/12T, 2.5–4.4 GHz, 65 W TDP) |
| RAM | 32 GB DDR4-3200 (2 × 16 GB dual-channel) |
| Storage | 512 GB PCIe 3.0 NVMe SSD |
| GPU | **None** — CPU-only inference |
| OS | Ubuntu 22.04 LTS (kernel 5.15.0-105) |
| LLM | LLaMA 3 8B Q4_K_M (GGUF, 4.65 GB) |

---

## Prerequisites

- **OS:** Ubuntu 22.04 LTS
- **Docker:** ≥ 24.0.5
- **Docker Compose:** ≥ v2.24.6
- **RAM:** ≥ 32 GB (model occupies ~4.65 GB; system steady-state ~13.8 GB)
- **Storage:** ≥ 20 GB free

---

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/<your-username>/cofl-workspace.git
cd cofl-workspace
```

### 2. Configure Environment

```bash
cp .env.example .env
# Edit .env to set your passwords and tokens
nano .env
```

### 3. Start All Services

```bash
docker compose up -d
```

### 4. Pull and Load the LLM Model

```bash
# Pull LLaMA 3 8B Q4_K_M (Pareto-optimal for CPU inference)
docker exec -it cofl-ollama ollama pull llama3:8b-instruct-q4_K_M

# Verify the model is loaded
docker exec -it cofl-ollama ollama list
```

Or use the setup script:

```bash
chmod +x scripts/setup_ollama.sh
./scripts/setup_ollama.sh
```

### 5. Verify All Services

```bash
# Check container health
docker compose ps

# Test FastAPI gateway
curl http://localhost:8000/health

# Test sentiment analysis endpoint
curl -X POST http://localhost:8000/analyze \
  -H "Content-Type: application/json" \
  -d '{"text": "My order arrived damaged and customer support was unhelpful.", "order_id": "ORD-001"}'

# Access n8n UI
open http://localhost:5678
```

---

## Service Endpoints

| Service | URL | Description |
|---|---|---|
| FastAPI Gateway | `http://localhost:8000` | Main API + interactive docs at `/docs` |
| n8n Workflow UI | `http://localhost:5678` | Visual workflow editor |
| Ollama API | `http://localhost:11434` | LLM inference backend |
| PostgreSQL | `localhost:5432` | Event log and metrics DB |

---

## API Reference

### `POST /analyze`

Accepts incoming complaint text, runs zero-shot sentiment classification via Ollama LLM (`temperature=0.1`), and returns a structured sentiment result.

**Request:**
```json
{
  "text": "My order arrived damaged and customer support was unhelpful.",
  "order_id": "ORD-12345",
  "star_rating": 1
}
```

**Response:**
```json
{
  "order_id": "ORD-12345",
  "sentiment_label": "negative",
  "sentiment_score": 0.12,
  "category": "product_defect",
  "requires_response": true,
  "latency_ms": 2130
}
```

> If `sentiment_score < 0.35`, the n8n workflow automatically triggers `/generate`.

---

### `POST /generate`

Generates a professional customer service reply with a role-conditioned prompt. Response is capped at 150 words and must include a concrete remediation offer.

**Request:**
```json
{
  "order_id": "ORD-12345",
  "original_text": "My order arrived damaged...",
  "sentiment_label": "negative",
  "category": "product_defect"
}
```

**Response:**
```json
{
  "order_id": "ORD-12345",
  "generated_reply": "Dear valued customer, we sincerely apologize...",
  "word_count": 87,
  "latency_ms": 1090
}
```

---

## n8n Workflow Setup

1. Open n8n at `http://localhost:5678`
2. Navigate to **Workflows → Import from File**
3. Import `n8n/workflows/complaint_triage_workflow.json`
4. Activate the workflow
5. The Webhook trigger URL will be: `http://localhost:5678/webhook/cofl-complaint`

### Workflow Pipeline

```
[Webhook Trigger]
      │
      ▼
[HTTP Request → POST /analyze]
      │
      ▼
[Switch: sentiment_score < 0.35?]
      │                    │
      ▼ Yes                ▼ No
[HTTP Request          [Log to PostgreSQL]
 → POST /generate]
      │
      ▼
[Log to PostgreSQL]
      │
      ▼
[Send Notification]
```

---

## Quantization Ablation Results

| Model | Quant | Size (GB) | Tok/s | F1 Macro | ROUGE-L |
|---|---|---|---|---|---|
| **LLaMA 3 8B** | **Q4_K_M ✓** | **4.65** | **17.2** | **0.836** | **0.421** |
| LLaMA 3 8B | Q5_K_M | 5.33 | 13.8 | 0.841 | 0.427 |
| LLaMA 3 8B | Q8_0 | 8.54 | 8.4 | 0.847 | 0.433 |
| LLaMA 3 8B | F16 (full) | 15.3 | 3.1 | 0.851 | 0.439 |
| Mistral 7B | Q4_K_M | 4.11 | 19.6 | 0.821 | 0.412 |
| Phi-3 3.8B | Q4_K_M | 2.30 | 24.1 | 0.798 | 0.388 |

Q4_K_M achieves **98.2% of full-precision F1** at **5.5× throughput** — Pareto-optimal for CPU-only edge deployment.

---

## FMEA: Failure Modes and Mitigations

| Failure Mode | Est. Frequency | Automated Mitigation |
|---|---|---|
| LLM inference timeout (>30s) | <0.5% | FastAPI timeout handler; n8n routes to PostgreSQL escalation queue |
| Webhook queue overflow | <0.1% | n8n queue-depth monitoring; PostgreSQL event log enables replay |
| PostgreSQL connection exhaustion | <0.1% | pgBouncer pool (max 20); inference continues without logging |
| LLM output parse failure | 2.3% (measured) | Regex fallback re-prompts; 97.7% first-pass success |
| Container OOM crash | Not observed | Docker restart policy: always; per-service health-check endpoints |

---

## Project Structure

```
cofl-workspace/
├── README.md
├── docker-compose.yml          # Four-service containerized stack
├── .env.example                # Environment variable template
├── fastapi/
│   ├── Dockerfile              # Python 3.11, 4 async workers
│   ├── requirements.txt        # FastAPI, Uvicorn, httpx, psycopg2
│   └── main.py                 # /analyze and /generate endpoints
├── scripts/
│   ├── setup_ollama.sh         # Pull LLaMA 3 8B Q4_K_M model
│   └── init_db.sql             # PostgreSQL schema initialization
├── n8n/
│   └── workflows/
│       └── complaint_triage_workflow.json   # Importable n8n workflow
├── postgres/
│   └── init.sql                # DB initialization on first run
└── docs/
    └── architecture.md         # Detailed architecture documentation
```

---

## Citation

If you use this codebase in your research, please cite:

```bibtex
@article{chen2026cofl,
  title={Designing a Continuous Operational Feedback Loop for Direct-to-Consumer Commerce: Integrating Event-Driven Automation and On-Premise Generative AI},
  author={Chen, Der-Fa and Chen, Yung-Hsing and Chen, Bo-Siang},
  journal={Information},
  volume={17},
  year={2026},
  publisher={MDPI}
}
```

---

## License

This project is released under the [MIT License](LICENSE).

---

## Contact

- Bo-Siang Chen — toyota554@gmail.com  
- Department of Vehicle Engineering, Nan Kai University of Technology, Nantou, Taiwan
