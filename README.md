# NCSU Advising Chatbot — TuffyBot

An AI-powered academic advising assistant for NC State University.
Combines a **hybrid Retrieval-Augmented Generation (RAG)** pipeline, a **Neo4j-backed degree audit graph (DAG)**, and a **tool-calling LLM agent** behind a Shibboleth-authenticated web app.

> **Note:** This repository is a public showcase. The full source code is private.
> If you'd like to know more about the implementation, design choices, or technical decisions, I'm open to discussion — feel free to reach out.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Scrapers](#scrapers)
- [RAG Pipeline](#rag-pipeline)
- [DAG — Degree Audit Graph](#dag--degree-audit-graph)
- [Authentication — Shibboleth SSO](#authentication--shibboleth-sso)
- [Agent / LLM Layer](#agent--llm-layer)
- [Contact](#contact)

---

## Overview

- Designed for NC State's **40,000+ student** population, with a modular, swappable component architecture for scaling.
- Lets a student upload a degree audit PDF, ask natural-language advising questions, and receive grounded answers with citations.
- Combines **semantic + keyword search** over scraped advising pages with a **graph-modeled course catalog** for prerequisite-aware recommendations.
- Self-hosted LLM (Llama 3.1) via Ollama on an NVIDIA GPU VM — no third-party API calls, FERPA-friendly.

---

## Architecture

![System Architecture](readme_assets/architecture.png)

- **Frontend (React + Docker):** PDF upload, major/minor selection, chat UI.
- **Backend (Node.js + Express):** routes student requests to the DAG and Agent services.
- **Agentic RAG (Python + Ollama + Llama 3.1):** orchestrates retrieval and tool calls.
- **DAG API (Python + FastAPI + Neo4j):** prerequisite graph and degree audit engine.
- **Vector store (Qdrant):** stores 5,000+ embedded chunks for hybrid search.
- **Authentication (Shibboleth SAML SP):** gates access via NC State's identity provider.

---

## Tech Stack

| Layer | Tools |
|---|---|
| **Frontend** | React, JavaScript, Docker |
| **Backend** | Node.js, Express, Multer |
| **Agent / LLM** | Python, Ollama, Llama 3.1, LangChain (text splitter) |
| **RAG** | Qdrant (vector + BM25 hybrid), nomic-embed-text |
| **DAG** | Neo4j, Cypher, FastAPI, Pydantic |
| **PDF Parsing** | pdfplumber |
| **Scraping** | Scrapy (multi-spider, multi-domain) |
| **Auth** | Shibboleth SAML 2.0, Apache reverse proxy |
| **DevOps** | Docker Compose (8 services), NVIDIA Container Toolkit |

---

## Scrapers

Two Scrapy projects feed the system: one for the **course catalog** (DAG) and one for **advising content** (RAG).

- **DAG Scraper:** crawls `catalog.ncsu.edu` for courses and program requirements, normalizing prerequisite groupings (AND/OR semantics, slash codes like `EC/ARE 301`).
- **RAG Scraper:** crawls **7,000+ pages** across **6 NC State domains** (advising, policies, student services, careers, getinvolved, course catalog).
- Handles **static HTML and JS-rendered API endpoints** — falls back to public APIs when pages are JavaScript-only.
- Outputs JSON for the loader/embedder; idempotent and rate-limited per domain.

![Scraper API approach](readme_assets/scraper_api_visual.png)

---

## RAG Pipeline

Hybrid retrieval over the embedded knowledge base feeds grounded context to the LLM.

- **5,120 chunks** embedded with `nomic-embed-text` (768 dimensions) and stored in **Qdrant**.
- **Hybrid search:** dense vector (semantic) + BM25 (exact term) merged via **Reciprocal Rank Fusion** — catches both natural-language questions and acronyms / course codes.
- **Top-8 retrieval** with title-prefixed embeddings for stronger context.
- Async ingest pipeline: scraping, chunking, and embedding run as a background job decoupled from query-time latency.

![RAG flow](readme_assets/rag_flow.png)

#### Embedding space

A semantic similarity network of the 5,120 chunks — clusters correspond to topical neighborhoods (advising vs. policies vs. course catalog).

![RAG network](readme_assets/rag_network.png)

#### Domain heatmap

Cross-domain similarity, showing how each scraped source maps onto the embedding space.

![RAG heatmap](readme_assets/rag_heatmap.png)

---

## DAG — Degree Audit Graph

A **Neo4j directed acyclic graph** modeling the NC State course catalog and degree requirements.

- **1,000+ courses** as nodes; **89,000+ prerequisite + curriculum edges**.
- Encodes AND/OR prerequisite groupings, slash-code expansions, wildcard prefixes (e.g., `BIO ` matches any BIO course), and pool-based requirements (GEP, free electives).
- **4-pass allocation engine** matches a student's transcript against requirements: Required → Elective Groups → Pool Requirements → Free Electives. Prevents double-counting and validates prerequisite chains for planned courses.
- Exposed via a FastAPI service with endpoints for `/audit`, `/full-audit`, `/missing`, `/requirements`, `/programs`, `/courses`.

![DAG example](readme_assets/dag_example.png)

#### PDF degree-audit parser

Uploaded transcripts are parsed via `pdfplumber` and normalized into the schema the DAG engine consumes.

![Parser pipeline](readme_assets/parser_pipeline_visual.png)

---

## Authentication — Shibboleth SSO

NC State requires SAML-based authentication for any service touching student data.

- **Shibboleth 2.0 Service Provider** behind an **Apache reverse proxy**, federated with NC State's identity provider.
- Forwards trusted headers (`x-shib-uid`, `x-shib-eppn`, `x-shib-mail`, `x-shib-2fauthed`) to the Express backend.
- HTTPS termination at the proxy; backend services stay on the internal Docker network.
- Supports the campus 2FA requirement for faculty/staff accounts.

---

## Agent / LLM Layer

A custom tool-calling agent loop drives the chat experience.

- **Llama 3.1 8B** served via **Ollama** on an NVIDIA GPU VM — fully self-hosted, no third-party API.
- Manual `TOOL_CALL: name(arg)` parsing via regex — works around quantized-model limitations without sacrificing tool use.
- **6 tools** orchestrated by the agent:
  - `search_advising_info(query)` — hybrid RAG search
  - `search_programs(query)` — list majors/minors
  - `get_courses(prefix)` — list by department prefix
  - `get_course_info(code)` — single-course details
  - `get_requirements()` — full audit (completed + missing)
  - `get_missing()` — incomplete requirements only
- Multi-round agent loop (capped at 5 tool calls) feeds tool output back into context with explicit instructions to ground answers in retrieved data.
- Returns Markdown-formatted answers with **inline source links** to the original advising pages.

![Agent flow](readme_assets/agent_visual.png)

---

## Contact

**Brandon Wroblewski**
CS @ NC State · Graduating Aug 2026
[bnwroble@ncsu.edu](mailto:bnwroble@ncsu.edu) · [LinkedIn](https://www.linkedin.com/in/brandon-wroblewski/)

If you want to dig into any specific component — the agent loop, the audit engine, the hybrid search, or the deployment — I'm happy to walk through it.
