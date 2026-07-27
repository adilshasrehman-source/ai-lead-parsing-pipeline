# Autonomous AI Sales Lead Parsing & CRM Data Pipeline

An enterprise RevOps automation workflow built to ingest unstructured sales discovery inputs, process them using the Anthropic Claude API, sanitize non-standard LLM outputs via custom Regular Expression matching, and structure key deal indicators into a CRM data store.

---

## 📌 Problem Statement
Large Language Models (LLMs) often return structured data wrapped in markdown code blocks (e.g., ` ```json ... ``` `) or surrounded by conversational text, despite strict system prompt instructions. This inconsistency breaks down-stream API parsers and automation modules in production environments. 

This project implements a fault-tolerant extraction layer using Regular Expressions to isolate and sanitize the raw JSON payload before sending it to schema validators and destination databases.

---

## 🏗️ System Architecture & Data Flow

```mermaid
graph LR
    A[Google Form Submission] --> B[Anthropic Claude API]
    B --> C[Text Parser - Regex Filter]
    C --> D[JSON Parsing Module]
    D --> E[Google Sheets / CRM Store]
