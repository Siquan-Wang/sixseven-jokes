# SixSeven Jokes 🎭

> **AI-powered humor platform for parents and teachers** — serving age-appropriate, personalized jokes with voice delivery across iOS and Android.

[![Product Hunt](https://img.shields.io/badge/Product%20Hunt-Live-orange)](https://www.producthunt.com/products/sixseven-jokes)
[![Website](https://img.shields.io/badge/Website-67jokes.com-blue)](https://67jokes.com/)

*Why is it called SixSeven? Because seven ate nine! 🤣*

---

## ⚠️ Disclaimer

> **This repository contains selected code demos and architecture showcases from the SixSeven Jokes project. The full production codebase (backend services, frontend apps, deployment configs, proprietary data) is not open-sourced.** 
---

## Overview

SixSeven Jokes is a **production AI product** that delivers personalized, age-appropriate jokes to kids through a complete system covering data ingestion, retrieval, generation, voice synthesis, and user feedback loops. The product is live on iOS, Android, and Web with several hundred daily active users.

This repository showcases the **core AI and data systems** I designed and built:

| Module | Description |
|--------|-------------|
| [**Data Pipeline**](data_pipeline/) | Multi-source joke extraction (PDF, images, text), LLM-based tagging, and two-stage deduplication |
| [**RAG Pipeline**](rag/) | FAISS-based semantic retrieval, preference-aware filtering, Gemini fallback generation |
| [**Fine-tuning**](fine_tuning/) | LoRA/QLoRA fine-tuning pipeline for custom joke generation models |
| [**Multimodal**](multimodal/) | Dual-voice audio synthesis with ElevenLabs, two-tier caching, async processing |
| [**Guardrail**](guardrail/) | Lightweight child-appropriateness safety filter (rule-based + LLM) |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER REQUEST                                  │
│              (age_range, scenario, preferences)                       │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    SCENARIO NORMALIZATION                             │
│         SentenceTransformer + FAISS semantic matching                 │
│         "classroom jokes" → "school" | "pet stories" → "animals"     │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    RETRIEVAL LAYER                                    │
│    ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐       │
│    │ Age + Theme  │→ │ FAISS Vector │→ │ Preference Filter   │       │
│    │ Filtering    │  │ Search       │  │ (like/dislike/view) │       │
│    └─────────────┘  └──────────────┘  └─────────────────────┘       │
└─────────────────────┬──────────────┬────────────────────────────────┘
                      │              │
              Sufficient?     Shortfall?
                      │              │
                      ▼              ▼
              ┌──────────┐  ┌─────────────────────────────────┐
              │  Return   │  │   GEMINI FALLBACK GENERATION    │
              │  Results  │  │  Preference-aware prompting      │
              └──────────┘  │  Robust structured output parsing │
                            │  Async write-back to content pool │
                            └──────────┬──────────────────────┘
                                       │
                                       ▼
                            ┌──────────────────────┐
                            │   SAFETY GUARDRAIL   │
                            │  Rule-based + LLM    │
                            └──────────┬───────────┘
                                       │
                                       ▼
                            ┌──────────────────────┐
                            │  MULTIMODAL DELIVERY  │
                            │  Dual-voice synthesis  │
                            │  Two-tier audio cache  │
                            └──────────────────────┘
```

---

## Key Technical Highlights

### 1. Retrieval-First, Generation-as-Fallback

The system **does not generate every joke from scratch**. Instead:
- Most requests are served from a curated, deduplicated joke pool via FAISS retrieval
- Gemini is only called when the pool lacks sufficient matches
- Generated jokes are written back to the pool, creating a **virtuous expansion cycle**

**Why this matters:** Lower cost, lower latency, higher content quality, better consistency.

### 2. Multi-Source Data Pipeline

```
PDF Joke Books ──┐
                  │     ┌──────────┐     ┌───────────┐     ┌──────────┐
Image Content  ───┼───▶ │Extractors│───▶ │LLM Tagger │───▶ │  Dedup   │───▶ Dataset
                  │     │(per type)│     │ (batched) │     │(2-stage) │
Text Files ──────┘     └──────────┘     └───────────┘     └──────────┘
```

- **Extractors:** Factory pattern — PDF uses PyMuPDF + Gemini, images use Gemini Vision, text uses regex + LLM fallback
- **Tagger:** Batched LLM tagging with constrained vocabularies for age groups, themes, joke types
- **Dedup:** Exact hash dedup (4,268 → 3,997) + semantic embedding dedup (3,997 → 3,303)

### 3. Preference-Aware Personalization

User feedback signals drive both retrieval and generation:
- **Retrieval:** Liked jokes boost score, disliked jokes are hard-excluded, viewed jokes are deprioritized
- **Generation:** User's liked/disliked jokes are injected into Gemini prompts as in-context conditioning
- **No cold start problem:** Works with zero history (random from pool) and improves with each interaction

### 4. Robust Structured Output Parsing

LLMs don't reliably output valid JSON. Our parser handles:
- Markdown code block wrappers (`\`\`\`json ... \`\`\``)
- Trailing commas
- Partial JSON arrays
- Non-JSON preamble text
- Regex fallback for severely malformed output

### 5. Dual-Voice Multimodal Delivery

Jokes are delivered as **audio experiences** with two different voices:
- Setup voice (warm, storytelling tone)
- Punchline voice (energetic, comedic tone)

Audio is generated via ElevenLabs and cached in a **two-tier system**:
- **Tier 1:** Local disk cache (~0ms lookup)
- **Tier 2:** Firebase Storage (persistent, shared across instances)

### 6. LoRA Fine-tuning Pipeline

Custom joke generation model trained with:
- **QLoRA** (4-bit quantization) for memory-efficient training
- Instruction-tuning format with **preference-conditioned examples**
- Stratified train/val/test splits by theme
- Multi-dimensional evaluation: format compliance, diversity, training overlap, LLM-as-judge

---

## Project Structure

```
sixseven-jokes/
├── config.py                           # Centralized configuration
├── requirements.txt                    # Dependencies
│
├── data_pipeline/                      # Data ingestion & processing
│   ├── extractors/
│   │   ├── base.py                     # Abstract extractor interface
│   │   ├── pdf_extractor.py            # PyMuPDF + Gemini extraction
│   │   ├── image_extractor.py          # Gemini Vision extraction
│   │   └── text_extractor.py           # Regex + LLM fallback extraction
│   ├── tagger.py                       # LLM-based age/theme/type tagging
│   ├── dedup.py                        # Exact + embedding-based dedup
│   └── pipeline.py                     # End-to-end pipeline orchestrator
│
├── rag/                                # Retrieval-Augmented Generation
│   ├── embeddings.py                   # FAISS index management
│   ├── scenario_matcher.py             # Semantic scenario normalization
│   ├── retrieval.py                    # Preference-aware joke retrieval
│   ├── generation.py                   # Gemini fallback generation
│   ├── structured_output.py            # Robust JSON parser for LLM output
│   └── pipeline.py                     # Full RAG serving pipeline
│
├── fine_tuning/                        # Model fine-tuning
│   ├── data_preparation.py             # Instruction-tuning dataset builder
│   ├── train.py                        # LoRA/QLoRA training script
│   └── evaluate.py                     # Multi-metric evaluation
│
├── multimodal/                         # Audio content delivery
│   ├── voice_synthesis.py              # ElevenLabs dual-voice synthesis
│   ├── audio_cache.py                  # Two-tier caching (local + Firebase)
│   └── delivery.py                     # Multimodal delivery pipeline
│
├── guardrail/                          # Content safety
│   └── safety_filter.py                # Rule-based + LLM safety filter
│
└── tests/                              # Test suite
    ├── test_data_pipeline.py
    ├── test_rag.py
    └── test_multimodal.py
```

---

## Quick Start

### 1. Setup

```bash
git clone https://github.com/Siquan-Wang/sixseven-jokes.git
cd sixseven-jokes
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your API keys
```

### 2. Run the Data Pipeline

```python
from data_pipeline import JokeDataPipeline

pipeline = JokeDataPipeline(gemini_api_key="your_key")
dataset, stats = pipeline.run("data/raw/")
pipeline.save_dataset(dataset, "data/jokes.json")
```

### 3. Serve Jokes via RAG

```python
from rag import JokeRAGPipeline
from rag.pipeline import JokeRequest

# Initialize pipeline
rag = JokeRAGPipeline()
rag.load_from_dataset("data/jokes.json")

# Serve a request
response = rag.serve(JokeRequest(
    age_range="5-7",
    scenario="animals",
    num_jokes=3,
    liked_joke_ids=["joke_123"],
))

for joke in response.jokes:
    print(f"Q: {joke['question']}")
    print(f"A: {joke['answer']}\n")
```

### 4. Generate Audio

```python
from multimodal import MultimodalDeliveryPipeline

delivery = MultimodalDeliveryPipeline()
results = delivery.deliver(response.jokes, include_audio=True)
```

### 5. Fine-tune a Custom Model

```python
from fine_tuning import JokeDatasetBuilder, JokeFineTuner

# Prepare dataset
builder = JokeDatasetBuilder()
splits = builder.build_dataset(jokes)
builder.save_dataset(splits, "data/fine_tuning/")

# Train
tuner = JokeFineTuner(use_qlora=True)
tuner.setup()
metrics = tuner.train("data/fine_tuning/train.jsonl", "data/fine_tuning/val.jsonl")
```

### 6. Run Tests

```bash
pytest tests/ -v
```

---

## Tech Stack

| Category | Technologies |
|----------|-------------|
| **LLM** | Google Gemini 1.5 Flash, Fine-tuned LoRA models |
| **Embeddings** | SentenceTransformers (all-MiniLM-L6-v2), FAISS |
| **Backend** | FastAPI, Firebase Auth, Firestore, Cloud Run |
| **Voice** | ElevenLabs TTS (multilingual v2) |
| **Fine-tuning** | HuggingFace Transformers, PEFT, TRL, bitsandbytes |
| **Frontend** | Google AI Studio, iOS/Android native apps |
| **Infra** | Firebase Storage, Cloudflare, GCP |

---

## Product Metrics

- 📱 **Platforms:** iOS + Android + Web
- 👥 **DAU:** Several hundred daily active users
- 🎯 **Content Pool:** 3,300+ curated and tagged jokes
- 🗣️ **Audio:** Dual-voice joke delivery with caching
- 🚀 **Launched on:** [Product Hunt](https://www.producthunt.com/products/sixseven-jokes)

---

---

## About

This project was co-founded and built with a team of friends. As the **Founding Engineer and AI Lead**, my primary contributions include:

- **Project vision & iteration** — Continuously refined the product direction and led development planning
- **Backend architecture** — Designed API interfaces, latency optimization strategies, and the retrieval-generation pipeline
- **AI/ML systems** — Built the embedding algorithms, data pipeline, RAG system, and preference-aware generation
- **Product analytics** — Analyzed user behavior patterns post-launch to drive data-informed iteration

For more information about the product, visit [67jokes.com](https://67jokes.com/) or our [Product Hunt page](https://www.producthunt.com/products/sixseven-jokes).

📧 Contact: info@sixsevengroup.com

## License

MIT License — This demo code is provided for portfolio and educational purposes.
