# assignment-dua

Assignment 2 — AI Product Engineering with TypeScript

## Overview

This repository implements PRD Generator using LLM orchestration patterns — moving beyond single-call patterns into multi-stage pipelines with quality gates and cross-validation

## Implementation

### Stage 1: Brief Refinement (Fan-out/Fan-in)

Given a raw product brief from a user, the AI must refine it before generating PRD sections to ensure clarity and scope control.

Implementation: three specialist personas (product manager, engineer, business analyst) review the brief in parallel, each focusing on their domain — user value, technical risks, and business constraints. Their reviews are synthesized into one refined brief with explicit assumptions and scope boundaries.

**Pattern:** Fan-out/Fan-in  
**Concepts:** Multi-perspective analysis, conflict resolution, explicit assumption tracking

---

### Stage 2: Section Generation (Evaluator-Optimizer)

Each PRD section must meet quality standards before being accepted. Low-quality drafts are automatically revised.

Implementation: section drafted → evaluated against rubric (score 1-5) → if score < 4, revised with evaluator feedback → final accepted. This loop ensures output consistency without human intervention.

**Pattern:** Evaluator-Optimizer  
**Concepts:** Self-evaluation, revision loop, quality threshold, structured rubric

---

### Stage 3: Cross-Section Consistency Check

Data Model and API Contract are generated independently — they can drift and become inconsistent.

Implementation: LLM checks consistency between Data Model entities and API endpoints. If inconsistent (missing endpoints, undefined fields), API Contract is reconciled against Data Model as source of truth.

**Pattern:** Document-level Consistency  
**Concepts:** Cross-validation, reconciliation, source-of-truth hierarchy

---

## Architecture

```
User Input → API → Queue → Worker → [Stage 1 → Stage 2 → Stage 3] → PRD Output
```

- **API**: Hono
- **Queue**: BullMQ + Redis (background processing)
- **Worker**: orchestrates all 3 stages sequentially
- **Output**: Markdown and PDF file

## PRD Sections Generated

1. Problem Statement — context, target user, MVP scope
2. User Stories — implementable stories with ID, priority, trigger/input/output
3. Acceptance Criteria — testable Given/When/Then criteria
4. Data Model — entities, fields, relations, constraints
5. API Contract — REST endpoints with request/response spec
6. Tech Stack — recommended stack proportional to MVP scope

## Tech Stack

- TypeScript
- Hono
- Prisma
- BullMQ + Redis
- Anvia
- Zod
- PDFKit + Marked

## Setup

```bash
pnpm install

# Start PostgreSQL
docker-compose up -d

# Run migrations
pnpm prisma migrate dev
```

Create a `.env` file:

```
OPENAI_API_KEY=your_key
DATABASE_URL=postgresql://prdgenerator:prdgenerator@localhost:5432/prdgenerator
REDIS_HOST=localhost
REDIS_PORT=6379
```

## Run

```bash
# Start API server
pnpm dev

# Start background worker (separate terminal)
pnpm worker:dev
```

API runs at `http://localhost:8000`

## Test

```bash
# Create PRD request
curl -X POST http://localhost:8000/prd \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Task Manager App",
    "description": "A simple task management app for personal use. Users can create, edit, delete tasks, and mark them as complete.",
    "additionalInfo": "MVP only, no authentication required"
  }'

# Check status
curl http://localhost:8000/prd/{projectId}

# Download result (when status = "done")
curl http://localhost:8000/prd/{projectId}/download --output prd.pdf
```

## Key Files

```
src/
├── modules/prd/
│   ├── services.ts    — LLM orchestration (fan-out, evaluator-optimizer, consistency)
│   ├── prompts.ts     — System prompts for each PRD section
│   ├── router.ts      — API routes
│   └── schema.ts      — Zod validation schemas
├── worker.ts          — BullMQ worker pipeline
├── utils/             — Prisma, queue, PDF helpers
```

## Patterns Demonstrated

| Pattern             | Application                       | Benefit                                   |
| ------------------- | --------------------------------- | ----------------------------------------- |
| Fan-out/Fan-in      | Brief refinement by 3 specialists | Balanced perspective, conflict resolution |
| Evaluator-Optimizer | Section quality gate              | Auto-revision, consistent output quality  |
| Structured Output   | All LLM calls use Zod schema      | Type-safe, predictable format             |
| Background Queue    | Long-running generation           | Non-blocking API, retry support           |
| Cross-validation    | Data Model vs API Contract        | Document coherence, reconciliation        |
