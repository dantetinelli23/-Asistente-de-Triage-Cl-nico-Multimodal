[README.md](https://github.com/user-attachments/files/28841743/README.md)
# 🏥 Multimodal Clinical Triage Assistant

> An AI automation pipeline built on **self-hosted n8n** that triages patient symptoms from **text, voice, or image** — grounded in real clinical protocols (RAG), gated by **human medical review**, and shipped with **self-evaluation** and **production-grade observability**.

![n8n](https://img.shields.io/badge/n8n-self--hosted-EA4B71)
![RAG](https://img.shields.io/badge/architecture-RAG-blue)
![Pinecone](https://img.shields.io/badge/vector%20db-Pinecone-000000)
![Cohere](https://img.shields.io/badge/embeddings-Cohere-39594D)
![Groq](https://img.shields.io/badge/LLM-Groq%20Llama%203.3-F55036)
![Anthropic](https://img.shields.io/badge/judge-Claude%20Sonnet-D97757)
![LangSmith](https://img.shields.io/badge/tracing-LangSmith-1C3C3C)
![Airtable](https://img.shields.io/badge/metrics-Airtable-18BFFF)

This is **not a chatbot that answers questions**. It is a system designed to be **reliable, auditable, safe, and measurable** — the four properties that separate a toy automation from one a real institution could adopt.

> ⚠️ **The system does not diagnose.** It returns a preliminary, evidence-based orientation and says so explicitly in every response. This single constraint shapes every design decision below.

<!--
  📸 SCREENSHOT — add a hero image here (highest-impact change you can make).
  Recommended: a Telegram conversation showing a real triage end to end.
  1. Create a folder /docs/img in the repo
  2. Drop the image as /docs/img/demo-telegram.png
  3. Uncomment the line below
-->
<!-- ![Demo: triage over Telegram](docs/img/demo-telegram.png) -->

---

## ✨ What it does

- 🎙️ **Multimodal intake** — text, voice (speech-to-text), and image, all unified into a single pipeline.
- 🔍 **RAG-grounded answers** — every triage is backed by real clinical protocols; no invented medical advice.
- 👨‍⚕️ **Human-in-the-loop** — high-urgency cases never reach the patient without a doctor's approval.
- ⚖️ **Self-evaluation** — an independent LLM judge audits every response and escalates when it disagrees.
- 🔁 **Resilient by design** — urgency-based timeouts, a second reviewer, a safe fallback protocol, model fallbacks, and retries on critical calls.
- 📊 **Two-layer observability** — technical metrics (LangSmith) + business/ops metrics (Airtable), tied together by an end-to-end correlation ID.
- 🤖 **Self-monitoring** — a scheduled daily loop watches its own quality KPIs and alerts the team on degradation.

---

## 🎯 The problem

Patients can't judge how serious their own symptoms are, and that uncertainty almost always resolves the wrong way: someone with a minor issue floods the ER "just in case", while someone with a serious one downplays it and loses critical time.

The system acts like the **triage nurse at an ER**: it doesn't diagnose or treat — it **sorts and prioritizes** before the person arrives. The patient describes their symptoms (typing, sending a voice note, or taking a photo) and receives an **urgency level + affected area + initial recommendation**, grounded in real protocols. High-urgency cases are reviewed by a doctor before going out.

---

## 🛠 Tech stack

| Layer | Tool | Role |
|------|------|------|
| Orchestration | **n8n** (self-hosted) | Workflow engine |
| Channel | **Telegram** | Multimodal patient intake (webhook) |
| Primary LLM | **Groq** (`llama-3.3-70b-versatile`) | Triage agent |
| Fallback LLM | **OpenAI / OpenRouter** | Backup if the primary fails |
| Judge LLM | **Anthropic** (`claude-sonnet-4.6`) | Quality evaluator |
| Speech-to-text | **Groq Whisper** (`whisper-large-v3-turbo`) | Audio transcription |
| Embeddings | **Cohere** (`embed-multilingual-v3.0`) | Vectorization (1024-dim) |
| Reranking | **Cohere Rerank** | Second relevance filter |
| Vector DB | **Pinecone** | Semantic search |
| Document parsing | **LlamaCloud / LlamaParse** | Protocol ingestion (PDF/DOCX) |
| Business metrics | **Airtable** | `CONSULTAS` & `LOGS_SISTEMA` tables |
| Tracing | **LangSmith** | Latency, tokens, cost per run |
| Analysis | **NotebookLM** | Insights over exported data |

---

## 🏗 Architecture

The system is split into **three independent workflows** that share one knowledge base. The separation is intentional — they solve different problems and run at different times.

```
FLOW A — INGESTION   (one-shot, per document)
  Form → LlamaParse → polling → chunking (1000/200) → Cohere → Pinecone

FLOW B — CONSULTATION   (real-time, per patient)
  Telegram ─► Switch (audio / image / text)
                  │
                  ├─ audio  → Whisper STT → quality gate ┐
                  ├─ image  → multimodal agent ──────────┤
                  └─ text   → direct ─────────────────────┤
                                          (triage agent + RAG)
                                                 │
                                            Normalize  ← merges the 3 branches
                                                 │
                                            LLM Judge (evaluation)
                                                 │
                                            Discrepancy? ── escalate? ──┐
                                                 │                       │
                                          NO ────┘            YES → HITL (doctor)
                                                 │                       │
                                                 └────────► Reply to patient
                  ▸ every checkpoint logs to Airtable + LangSmith

FLOW C — SELF-IMPROVEMENT   (scheduled, daily)
  Schedule → read CONSULTAS metrics → quality dropped? → alert team
```

**The key design move:** all three modalities converge in a single `Normalize` node that gives them a common shape. From there on, **the rest of the pipeline is channel-agnostic** — which is why adding the third modality (text) was nearly trivial.

<!--
  📸 SCREENSHOT — add a picture of the n8n canvas here.
  It visually communicates the scale of the workflow at a glance.
  Save it as /docs/img/n8n-canvas.png and uncomment:
-->
<!-- ![n8n workflow canvas](docs/img/n8n-canvas.png) -->

---

## 📂 What's in this repo

```
.
├── Proyecto_Final.json          # Main n8n workflow (ingestion + consultation + self-improvement)
├── Logger_Proyecto_Final.json   # Reusable Logger sub-workflow (DRY logging to Airtable)
└── README.md                    # You are here
```

> 💡 **Suggested additions** (see TODO comments in this file): a `/docs` folder with the full design document (PDF) and a `/docs/img` folder with screenshots of the bot, the n8n canvas, and the Airtable metrics tables.

---

## ⚙️ How it works (in 60 seconds)

**Ingestion** turns clinical protocols (PDF/DOCX) into searchable knowledge: LlamaParse cleans the document, it's split into 1000-char chunks with 200-char overlap, embedded with Cohere, and stored in Pinecone.

**Consultation** is the heart. A Telegram webhook receives the message; a `Switch` routes it by type; each branch converts its format to text the agent understands (Whisper for audio, a multimodal model for images, direct for text). The triage agent retrieves relevant protocols from Pinecone (re-ranked by Cohere) and produces a structured triage. An independent **LLM judge** scores it and emits its own urgency — if the judge sees something more serious, the case **escalates to a doctor** via Telegram's *send-and-wait*. Everything is logged.

**Self-improvement** runs daily: it reads the latest cases, computes quality KPIs (avg. safety score + discrepancy rate), and **alerts the team** if either crosses a threshold — it detects and warns, it never silently changes behavior.

📄 **Full design rationale** (every parameter and decision explained from scratch) lives in the [project deliverable](#-live-artifacts).

---

## 💡 Key design decisions

| Decision | Why |
|----------|-----|
| RAG instead of a bare LLM | An LLM hallucinates; in healthcare that's unacceptable. Force grounding in real protocols. |
| `temperature: 0.2` on triage | Fidelity to the source and consistency, not creativity. |
| Audio quality gate | Don't triage on a doubtful transcription — ask the patient to re-record. |
| Unify branches in `Normalize` | The downstream pipeline is channel-agnostic → easy to extend. |
| Independent judge LLM | An automatic second pair of eyes that catches the agent's mistakes. |
| Urgency-based dynamic timeouts | Waiting time should scale with risk. |
| Correlation ID in every log | End-to-end traceability — reconstruct any case from its ID. |
| Logger as a sub-workflow | DRY: configure logging once, call it from everywhere. |
| Self-improvement that *alerts* (doesn't auto-tune) | Prudence: a human approves changes too, not just edge cases. |

---

## 🚀 Running it locally

This is a self-hosted n8n project that depends on several external services. You'll need accounts/API keys for: **Telegram** (Bot via @BotFather), **Groq**, **OpenAI or OpenRouter**, **Anthropic**, **Cohere**, **Pinecone**, **LlamaCloud**, **Airtable**, and **LangSmith**.

**1. Start n8n (self-hosted, via Docker):**
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```
n8n will be available at `http://localhost:5678`.

**2. Create the Pinecone index** — dimension `1024`, metric `cosine` (matches Cohere `embed-multilingual-v3.0`).

**3. Set up the Airtable base** with two tables:
- `CONSULTAS` — one row per completed consultation (urgencies, eval scores, doctor decision, audio signals…).
- `LOGS_SISTEMA` — one row per system event, with a severity level (`INFO` / `WARNING` / `ERROR` / `CRITICO`).

**4. Import the workflows** into n8n (Workflows → Import from File):
- `Proyecto_Final.json`
- `Logger_Proyecto_Final.json`

**5. Configure credentials** in n8n's credential manager for every service above. Credentials live in n8n, never in the workflow files.

**6. Ingest your protocols** — run the ingestion flow and upload your clinical protocol PDFs/DOCX.

**7. Activate** the main workflow so it listens on the Telegram webhook, then message your bot.

> 🔐 No secrets are committed to this repo — all keys are managed through n8n's credential store.

---

## ⚠️ Limitations & next steps

Documented honestly — knowing a system's limits is part of understanding it.

- The ingestion polling loop has no maximum retry cap (risk of an infinite loop if LlamaParse stalls).
- The text-branch agent showed a recurring `Bad request` error on some runs (visible in LangSmith) — pending diagnosis.
- The image evaluator judges the agent's *description* of the photo, not the original image. A truly multimodal evaluator would be more robust.
- The self-improvement thresholds (safety `< 3.5`, discrepancy `> 30%`) are initial estimates and must be recalibrated against real production data.
- **Before any real-world use:** institutional legal/clinical validation, compliance with personal-data law (Argentina's Ley 25.326), explicit informed consent, and a permanent human backup channel.

---

## 🔗 Live artifacts

- 📊 **Metrics dashboard (Airtable)** — live `CONSULTAS` & `LOGS_SISTEMA` tables: <!-- replace with your share link --> `[add link]`
- 🧩 **Workflow JSON** — see `Proyecto_Final.json` and `Logger_Proyecto_Final.json` in this repo.

---

## 👤 About

Built by **Dante Tinelli** as the final project for **IA Automation Avanzado (Coderhouse)**.

A production-minded exploration of AI automation: end-to-end RAG, real multimodal integration, human-in-the-loop design, automatic evaluation, and two-layer observability — built no-code/low-code on n8n.

<!-- Add your links so employers can reach you: -->
- 🔗 LinkedIn: `[add link]`
- 💻 GitHub: `[add link]`
- ✉️ Email: `[add email]`
