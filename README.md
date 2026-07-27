# Autonomous AI Sales Lead Parsing & CRM Data Pipeline

An enterprise RevOps automation workflow built to ingest unstructured sales call transcripts, process them through Anthropic Claude, sanitize non-standard LLM markdown outputs using Regular Expressions, and structure key deal indicators into a CRM data store.

## 📌 Problem Statement
LLMs like Claude often wrap JSON outputs in markdown code blocks (```json ... ```) despite strict system instructions, breaking down-stream API integrations and spreadsheet parsers. This project implements a fault-tolerant extraction layer to guarantee schema consistency.

## 🏗️ Architecture & Data Flow
```mermaid
flowchart LR
    A[Google Form Submission] --> B[Anthropic Claude API]
    B -->|Raw Text Output| C[Text Parser Module]
    C -->|Regex Regex \{[\s\S]*\}| D[JSON Parse Module]
    D -->|Structured Schema| E[Google Sheets / CRM Store]

    style A fill:#4285F4,color:#fff
    style B fill:#7057ff,color:#fff
    style C fill:#ff6b6b,color:#fff
    style D fill:#22c55e,color:#fff
    style E fill:#0F9D58,color:#fff
1. **Trigger:** Webhook/Form submission capturing lead transcript & discovery notes.
2. **AI Inference:** Anthropic Claude analyzes transcript for `lead_score`, `decision_maker`, `pain_points`, `budget_mentioned`, and `routing_action`.
3. **Regex Extraction Layer:** Custom Regular Expression (`\{([\s\S]*)\}`) isolates raw JSON from markdown wrappers across multi-line payloads.
4. **Data Transformation:** Native JSON parsing and dynamic schema validation (`GTM Structure`).
5. **Destination:** Auto-populates enriched deal metrics into CRM / Google Sheets.

## 🛠️ Tech Stack & Key Concepts
- **Platform:** Make.com (iPaaS)
- **AI/LLM:** Anthropic Claude API
- **Data Handling:** Regex (Regular Expressions), JSON Schema Definition, Data Mapping
- **Integrations:** Google Workspace / CRM Webhooks

## 📁 Repository Contents
- `ai-lead-parsing-pipeline.json`: Complete Make.com scenario blueprint export ready for import.
