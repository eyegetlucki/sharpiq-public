# SharpIQ ⚡
**AI-powered NBA, MLB & Soccer prop betting analysis platform**

> 🔗 Live at [sharpiq.online](https://sharpiq.online)

---

## Overview

SharpIQ is a production AI analytics platform that uses a multi-agent RAG pipeline to analyze sports prop bets across NBA, MLB, and Soccer (including World Cup 2026). Every output is grounded in semantically retrieved context with cited source embeddings — nothing hallucinated.

Built and operated solo. This is a live commercial product — source code is proprietary.

---

## Architecture

### Multi-Agent Pipeline (/analyze/v2)

SharpIQ's core analysis runs on a 5-node LangGraph StateGraph:

```
Research Agent → Fatigue Agent → Analyst Agent → Critic Agent → Synthesizer
```

| Agent | Type | Responsibility |
|---|---|---|
| **Research Agent** | Pure retrieval — no LLM | Semantic search against pgvector, sport-aware routing, writes game logs and context to typed AgentState |
| **Fatigue Agent** | Pure computation — no LLM | Composite fatigue score (rolling minutes, Haversine travel miles, timezone shift, rest days, back-to-back detection) |
| **Analyst Agent** | Claude Sonnet | Structured JSON verdict with confidence score, signals, reasoning, and cited source embeddings |
| **Critic Agent** | Claude Sonnet | Pressure-tests Analyst reasoning, identifies overconfidence and missing context, returns revised confidence |
| **Synthesizer** | No LLM | Applies sport-specific confidence caps, fires human-in-the-loop review_flag when confidence < 60%, assembles final output |

Full LangSmith tracing per node with token counts, latency, and cost monitoring.

---

### RAG Pipeline

```
Multi-source nightly ETL
        ↓
Enrichment layer (fatigue · travel · weather · matchup signals)
        ↓
OpenAI text-embedding-3-small (1536-dim vectors)
        ↓
Supabase pgvector (semantic storage)
        ↓
At inference: semantic retrieval → LangGraph multi-agent pipeline → structured JSON output
```

**Data Sources:**
- NBA: nba_api (game logs, splits, shot quality metrics, TS%, possessions)
- MLB: statsapi (game logs, pitcher matchups, batter splits, batting order), Baseball Savant (barrel%, exit velocity, hard-hit%)
- Soccer: ESPN API (World Cup 2026), FBref via soccerdata (11 leagues)
- Odds: The Odds API (NBA/MLB/Soccer prop lines)
- Travel: Haversine distance between venues, timezone shift calculation
- Weather: NWS API (US venues), Open-Meteo (international venues)

---

### AWS Infrastructure

```
Internet → ALB (443) → ECS Fargate (8000) → FastAPI Backend
                ↓
           CloudWatch Logs
```

| Component | Details |
|---|---|
| **Container** | Docker (python:3.11-slim), stored in ECR |
| **Runtime** | AWS ECS Fargate (1 vCPU / 2GB RAM) — no EC2 management |
| **Load Balancer** | Application Load Balancer, internet-facing |
| **SSL** | AWS Certificate Manager, DNS-validated via Namecheap |
| **IAM** | Least-privilege task execution role |
| **Security Groups** | ECS accessible from ALB only — no public container access |
| **Monitoring** | CloudWatch log streaming to /ecs/sharpiq-api |
| **Health Checks** | ALB polls GET /health every 30s |

---

## Full Stack

| Layer | Technology |
|---|---|
| **Frontend** | Next.js App Router · TypeScript · Tailwind CSS · Framer Motion · Vercel |
| **Backend** | FastAPI · Python 3.11 · AWS ECS Fargate |
| **Auth** | Clerk (JWT · email · Google · Apple SSO) |
| **Payments** | Stripe Checkout + webhooks ($15/mo subscription) |
| **AI / LLM** | Claude Sonnet (Analyst + Critic) · OpenAI Embeddings |
| **Orchestration** | LangGraph StateGraph · LangSmith tracing |
| **Vector DB** | Supabase pgvector (1536-dim embeddings) |
| **Data Pipeline** | Railway cron jobs (5 services — nightly ingest, props refresh, settlement) |
| **DNS / SSL** | Namecheap + AWS ACM + ALB |

---

## Key Engineering Decisions

**Why separate Research and Fatigue agents with no LLM?**
Deterministic computation doesn't need a language model. Keeping retrieval and scoring outside the LLM eliminates a whole class of variability and makes those nodes fast, cheap, and perfectly reproducible.

**Why a Critic Agent?**
Single-model RAG systems have no mechanism for catching their own overconfidence. The Critic Agent reads the Analyst's reasoning and actively challenges it — flagging missing context, logical gaps, and inflated confidence scores before the output reaches the user.

**Why human-in-the-loop at 60% confidence?**
Financial decision-making requires a safety layer. Picks below 60% confidence get flagged for human review rather than auto-published — keeping the product trustworthy even when the model is uncertain.

**Why ECS Fargate over serverless?**
The nightly ETL pipeline and multi-agent inference require consistent compute rather than cold-start tolerance. Fargate gives container-level control without EC2 management overhead.

---

## What This Project Demonstrates

- Production RAG pipeline design with multi-source ETL, embedding, and semantic retrieval
- LangGraph multi-agent architecture with typed state, conditional edges, and failure recovery
- Human-in-the-loop design patterns for financial-stakes AI outputs
- AWS ECS Fargate production deployment with ALB, IAM, Security Groups, and CloudWatch
- LangSmith observability across a multi-node agentic pipeline
- Full-stack SaaS with auth, payments, and subscription management

---

1. Terraform Infrastructure as Code

All AWS infrastructure (ECS Fargate, ECR, ALB, ACM, IAM, CloudWatch, Security Groups) is now version-controlled in infrastructure/. Anyone can see exactly what's deployed, reproduce it, or audit changes via terraform plan. State is stored in S3 with locking. No resources were recreated — everything was imported from the existing live setup.

2. RAGAS Eval Harness

An offline evaluation system that scores SharpIQ's AI pipeline using the RAGAS framework. It reads settled predictions from the database, retrieves the same context the AI used, and scores each prediction across three metrics: faithfulness (did the reasoning match the data?), answer relevance (did the verdict address the prop?), and context precision (was the retrieved data useful?). Runs as a local CLI — completely separate from production. Lays the groundwork for fine-tuning the Critic agent once enough data is collected.

3. Natural Language SQL Agent — POST /query/natural

A new API endpoint that lets you query SharpIQ's entire database in plain English. Ask things like "which players had the highest fatigue score last week?" or "what's the win rate on soccer props?" — Claude generates the SQL, executes it safely (SELECT-only, 100-row cap, read-only session), and returns both the raw results and a plain English explanation of the findings. Requires authentication. No extra infrastructure — runs over the existing Supabase connection.

## Developer

**Laitrell Uy-Xayachak** — AI systems developer and solo product builder
- 🌐 [sharpiq.online](https://sharpiq.online)
- 🐙 [github.com/eyegetlucki](https://github.com/eyegetlucki)
- 📧 laitrell.company@gmail.com

> Source code is proprietary. This repository serves as a public architecture overview for portfolio purposes.
