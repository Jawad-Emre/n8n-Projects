# DevTriage AI: Autonomous Incident Triage Copilot

<p align="center">
  <img src="https://img.shields.io/badge/n8n-Workflow%20Automation-FF6D5A?style=flat-square&logo=n8n&logoColor=white" />
  <img src="https://img.shields.io/badge/Pinecone-Vector%20Database-000000?style=flat-square&logo=pinecone&logoColor=white" />
  <img src="https://img.shields.io/badge/Google%20Gemini-3.5%20Flash%20Lite-4285F4?style=flat-square&logo=google&logoColor=white" />
  <img src="https://img.shields.io/badge/Embeddings-3072%20Dimensions-8B5CF6?style=flat-square" />
  <img src="https://img.shields.io/badge/Safety%20Gate-Zero--LLM%20Regex-10B981?style=flat-square" />
</p>

---

## Overview

DevTriage AI is an autonomous, confidence-gated incident triage copilot built inside n8n. It ingests raw developer error logs and stack traces, sanitizes credentials deterministically, cross-checks incidents against internal company runbooks via high-dimensional vector search, and either provides a verified code fix with citations or safely escalates to senior human engineers.

The system incorporates an Active Learning Flywheel: when a senior engineer resolves an escalated incident, the postmortem is automatically vectorized and indexed into Pinecone in real time. This enables the system to autonomously resolve future occurrences of the same failure with 95%+ confidence without model retraining.

---

## Architecture

<p align="center">
  <img src="architecture.svg" alt="DevTriage AI Architecture" width="100%" />
</p>

---

## Included Workflows

1. `Workflow_1_Knowledge_Ingestion.json`
   * Chunks developer documentation and company runbooks.
   * Generates 3,072-dimensional vector embeddings with Google Gemini.
   * Ingests vectors into the `devtriage-docs` Pinecone index.

2. `Workflow_2_DevTriage_Engine.json`
   * Deterministic Safety Gate: Intercepts leaked credentials and P1 outages with zero-LLM regex.
   * AI Agent: Google Gemini 3.5 Flash Lite with tool-calling integration to Pinecone.
   * Confidence Gate (Score >= 0.70): Verifies citations before outputting code; escalates ambiguous errors safely.

3. `Workflow_3_Active_Learning_Flywheel.json`
   * Incident resolution interface for senior on-call engineers.
   * Formats the postmortem and auto-indexes it into Pinecone.
   * Closes the self-learning loop (boosts confidence on recurring errors from 25% to 95%).
