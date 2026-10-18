# DevTriage AI: Autonomous Incident Triage Copilot

<p align="center">
  <img src="https://img.shields.io/badge/n8n-Workflow%20Automation-FF6D5A?style=flat-square&logo=n8n&logoColor=white" />
  <img src="https://img.shields.io/badge/Pinecone-Vector%20Database-000000?style=flat-square&logo=pinecone&logoColor=white" />
  <img src="https://img.shields.io/badge/Google%20Gemini-3.5%20Flash%20Lite-4285F4?style=flat-square&logo=google&logoColor=white" />
  <img src="https://img.shields.io/badge/Embeddings-3072%20Dimensions-8B5CF6?style=flat-square" />
  <img src="https://img.shields.io/badge/Safety%20Gate-Zero--LLM%20Regex-10B981?style=flat-square" />
  <img src="https://img.shields.io/badge/Status-Production%20Ready-success?style=flat-square" />
</p>

---

## Executive Summary

DevTriage AI is an autonomous, confidence-gated incident triage copilot built natively inside n8n. It ingests raw developer error logs and stack traces, sanitizes credentials deterministically, cross-checks incidents against internal company runbooks via high-dimensional vector search, and either provides a verified code fix with citations or safely escalates to senior human engineers.

The system features an Active Learning Flywheel: when a senior engineer resolves an escalated incident, the postmortem is automatically vectorized and indexed into Pinecone in real time. This enables the system to autonomously resolve recurring occurrences of the same error with 95%+ confidence without model retraining.

---

## System Architecture

<p align="center">
  <img src="DevTriage/architecture.svg" alt="DevTriage AI Architecture" width="100%" />
</p>

### End-to-End Workflow Diagram

```mermaid
flowchart TD
    %% Workflow 1
    subgraph WF1 [Workflow 1: Knowledge Ingestion Pipeline]
        A1["Company Docs / Runbooks<br/>(Markdown / Notion / Drive)"] --> A2["Text Splitter Node<br/>(500 chars / 50 overlap)"]
        A2 --> A3["Embeddings Google Gemini<br/>(3,072 Dims Dense)"]
        A3 --> A4["Pinecone Vector Store<br/>(Index: devtriage-docs)"]
    end

    %% Workflow 2
    subgraph WF2 [Workflow 2: Autonomous DevTriage Copilot]
        B1["Incident Trigger<br/>(Form / Webhook / Sentry)"] --> B2["Step 1: Deterministic Safety Gate<br/>(Regex Secret Redaction & P1 Filter)"]
        
        B2 --> B3{"Safety Check:<br/>P1 Outage or Secret Leak?"}
        
        %% Safety Gate Branches
        B3 -- "YES (Critical)" --> B4["Immediate Slack / PagerDuty Alert<br/>(Zero-LLM Bypass)"]
        B4 --> B5["Escalate to Senior On-Call"]
        
        B3 -- "NO (Safe Log)" --> B6["Clean Log & Extract Signature<br/>(Exception Name + Message)"]
        
        %% Retrieval
        B6 --> B7["Step 2 & 3: AI Agent Node<br/>(Gemini 3.5 Flash Lite)"]
        B8["Pinecone Vector Store Tool<br/>(search_company_runbooks)"] -.->|"Tool Query"| B7
        
        %% Confidence Gate
        B7 --> B9{"Confidence Gate:<br/>Score >= 0.70?"}
        
        %% High Confidence
        B9 -- "YES (>= 0.70)" --> B10["Instant Verified Fix<br/>(Root Cause + Fix + Citations)"]
        
        %% Low Confidence / Escalation
        B9 -- "NO (< 0.70)" --> B11["Low Confidence Escalation Queue<br/>(Anti-Hallucination Safe Failover)"]
    end

    %% Connection to Workflow 3
    B11 -.->|"Routes Unresolved Incident"| C1

    %% Workflow 3
    subgraph WF3 [Workflow 3: Active Learning Flywheel]
        C1["Senior Engineer Review Form<br/>(Inputs Verified Fix & Cause)"] --> C2["Format Postmortem Code Node<br/>(Standardized Runbook Template)"]
        C2 --> C3["Embeddings Google Gemini<br/>(3,072 Dims)"]
        C3 --> C4["Pinecone Upsert Node<br/>(Auto-Indexed Solution)"]
    end

    %% Flywheel Feedback
    C4 -.->|"Re-indexes into Vector Store"| A4

    classDef primary fill:#2563eb,stroke:#1d4ed8,stroke-width:2px,color:#fff;
    classDef success fill:#16a34a,stroke:#15803d,stroke-width:2px,color:#fff;
    classDef warning fill:#ea580c,stroke:#c2410c,stroke-width:2px,color:#fff;
    classDef danger fill:#dc2626,stroke:#b91c1c,stroke-width:2px,color:#fff;
    classDef purple fill:#7c3aed,stroke:#6d28d9,stroke-width:2px,color:#fff;

    class B1,A1,C1 primary;
    class B10,C4 success;
    class B3,B9 warning;
    class B4 danger;
    class B7,A3,C3 purple;
```

---

## Core Architectural Pillars

### 1. Deterministic Safety Gate (Zero-LLM Risk)
* Problem: Sending raw developer error logs directly to an LLM risks leaking customer API keys, passwords, database URIs, and PII, violating SOC2, HIPAA, and GDPR compliance standards. Furthermore, model latency during critical production outages delays immediate incident response.
* Implementation: A compiled JavaScript regex filter intercepts:
  * Secrets & Credentials: Detects OpenAI/Anthropic keys (`sk-[a-zA-Z0-9_\-]{20,}`), AWS Access Keys (`AKIA[0-9A-Z]{16}`), Bearer Tokens, and Database Connection URIs, replacing matches with `[REDACTED_SECRET]`.
  * P1 Outage Detection: Detects fatal failure keywords (`OutOfMemoryError`, `DatabaseConnectionPoolExhausted`, `FATAL_CRASH`, `SIGSEGV`) and immediately alerts on-call engineers via Slack or PagerDuty, completely bypassing LLM inference.
  * Noise Reduction: Strips ISO timestamps, hex memory addresses, and normalizes stack frames to extract the canonical error signature.

### 2. High-Dimensional Hybrid Retrieval (Pinecone + Gemini)
* Embeds company runbooks and incident documentation using Google's 3,072-dimensional dense embeddings (`gemini-embedding-001`).
* Stored in a serverless Pinecone index (`devtriage-docs`) using cosine distance similarity.
* Allows the AI Agent to perform semantic lookup and map noisy stack traces to authoritative internal runbooks.

### 3. Anti-Hallucination & Mathematical Confidence Gate (Score >= 0.70)
* Powered by Google Gemini 3.5 Flash Lite with strict grounding prompts and function-calling integration.
* Anti-Hallucination Protocol: The model is restricted from inferring unverified fixes. If no internal documentation resolves the incident, the model must output a confidence score below 0.50.
* Routing Logic:
  * Confidence >= 0.70: Automated resolution delivered with root cause analysis, verified code fix, and cited runbook titles.
  * Confidence < 0.70: Code generation is suppressed; incident metadata is routed directly to the Senior Human Engineer Queue.

### 4. Active Learning Flywheel (Human-in-the-Loop)
* When an incident is escalated, an on-call engineer reviews the log and submits a verified postmortem.
* n8n automatically transforms the postmortem into a structured runbook, generates 3,072-dim embeddings, and upserts it directly into Pinecone.
* Outcome: On future occurrences of the same failure, triage confidence transitions from 25% to 95%, achieving autonomous resolution.

---

## Live Verification & Benchmark Results

The system was verified across four real-world production incident test cases:

| Benchmark Scenario | Ingested Error Payload | Confidence Score | System Route & Outcome |
| :--- | :--- | :---: | :--- |
| Test 1: Credential Leak & P1 Crash | `FATAL: DatabaseConnectionPoolExhausted - Secret: Bearer sk-998234871239...` | Bypassed | Zero-LLM Escalation: Credential sanitized, P1 flagged, bypassed AI straight to Senior On-Call. |
| Test 2: Known Documented Error | `stripe.error.SignatureVerificationError: Invalid signature in webhooks.py:42` | 92% | Instant Verified Fix: Cited Stripe Runbook, provided raw request body buffer fix. |
| Test 3: Unseen Error (Pre-Flywheel) | `grpc._channel._InactiveRpcError: StatusCode.RESOURCE_EXHAUSTED Quota exceeded` | 25% | Anti-Hallucination Lock: Refused to guess, score locked at 25%, escalated to Human Queue. |
| Test 4: Recurring Error (Post-Flywheel) | Same gRPC error re-tested after engineer postmortem was indexed via Workflow 3 | 95% | Autonomous Fix: Retrieved human-verified postmortem, cited fix, and resolved autonomously. |

---

## Repository Structure

```
.
├── README.md                                      # Project documentation
└── DevTriage/
    ├── architecture.svg                           # Vector architecture diagram
    ├── architecture.md                            # Architecture design blueprint & Mermaid charts
    ├── README.md                                  # Module documentation
    ├── Workflow_1_Knowledge_Ingestion.json        # n8n Pipeline: Runbook chunking & Pinecone ingestion
    ├── Workflow_2_DevTriage_Engine.json           # n8n Pipeline: Safety Gate, Gemini Agent & Confidence Gate
    └── Workflow_3_Active_Learning_Flywheel.json   # n8n Pipeline: Engineer resolution & Pinecone auto-indexing
```

---

## Quickstart & Setup Guide

### Prerequisites
1. An active instance of n8n (Cloud or self-hosted via Docker).
2. A Pinecone account (Serverless Starter Tier).
3. A Google AI Studio API Key.

### Step 1: Create the Pinecone Index
In your Pinecone Dashboard, click Create Index:
* Name: `devtriage-docs`
* Dimensions: `3072` (matches Gemini's embedding output)
* Metric: `cosine`
* Vector Type: `Dense`

### Step 2: Import Workflows into n8n
1. Open n8n -> Add Workflow -> Import from File.
2. Import `Workflow_1_Knowledge_Ingestion.json`.
3. Import `Workflow_2_DevTriage_Engine.json`.
4. Import `Workflow_3_Active_Learning_Flywheel.json`.

### Step 3: Configure Credentials in n8n
* Pinecone API: Connect using your Pinecone API Key.
* Google Gemini API: Connect using your Google AI Studio API Key.

### Step 4: Run Knowledge Ingestion
1. Open Workflow 1 (`DevTriage - Knowledge Ingestion`).
2. Click Test workflow once.
3. Verify in Pinecone Dashboard that the `devtriage-docs` index shows the indexed runbook records.

### Step 5: Execute Incident Triage
1. Open Workflow 2 (`DevTriage - Triage Engine`).
2. Open the form URL and submit an error log.
3. Observe real-time sanitization, vector retrieval, confidence scoring, and automated routing.

---

## Tech Stack & Integrations

* Orchestration: n8n (Form Triggers, If/Switch Routers, Code Nodes, LangChain Agent Nodes)
* Large Language Model: Google Gemini 3.5 Flash Lite (Tool calling, structured JSON output)
* Embeddings: Google `gemini-embedding-001` (3,072 dimensions)
* Vector Database: Pinecone Serverless
* Deterministic Logic: Custom JavaScript V8 regex engine

---

## Author

JAWAD-EMRE  
GitHub: [@Jawad-Emre](https://github.com/Jawad-Emre)  
Repository: [n8n-Projects](https://github.com/Jawad-Emre/n8n-Projects)
