# Complete A-Z Architecture & Implementation Plan: Steps 1 to 8

This document serves as the complete, authoritative specification for the **AI-Powered Bid/RFP Proposal Generation Pipeline** in **AgentMitra**. It details every step from initial document ingestion (Step 1) to final proposal generation (Step 8) with concrete examples, schemas, and CLI contracts.

---

## 🏗️ High-Level Architecture Flow Diagram

```text
  ┌─────────────────────────────────────────────────────────────┐
  │ PHASE 1: Document Ingestion & Processing (Steps 1 – 4)      │
  ├─────────────────────────────────────────────────────────────┤
  │ Step 1a: Folder Intake (Discovers raw files: PDF, DOCX, etc)│
  │ Step 1b: Markdown Conversion (Outputs page-marked .md)      │
  │ Step 2 : Document Classification (Detects Doc Type & TOC)   │
  │ Step 3 : Section Detection (Extracts section page ranges)   │
  │ Step 4 : Chunking & Azure AI Search Indexing                │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ PHASE 2: Knowledge Understanding & Generation (Steps 6 – 8) │
  ├─────────────────────────────────────────────────────────────┤
  │ Step 6 : Knowledge Understanding Layer (Manifest Creation)  │
  │          - 6.1 Section Summaries (LLM page-window summary)  │
  │          - 6.2 File Summary (Combines section summaries)    │
  │          - 6.3 Master Manifest (step6_manifest.json)        │
  │                                                             │
  │ Step 7 : Bid Brief Planning & Section Mapping               │
  │          - Maps proposal requirements to source files/sections│
  │          - Produces step7_bid_mapping.json                  │
  │                                                             │
  │ Step 8 : Content Generation Agent                           │
  │          - Grounded RAG content drafting per section        │
  │          - Produces final proposal (final_bid_brief.md)     │
  └─────────────────────────────────────────────────────────────┘
```

---

## 📌 STEP 1a: Folder Intake (`pipeline/step1_intake`)

### 📖 Description
Discovers all files (PDF, DOCX, XLSX, PPTX, etc.) in the project input directory, validates extensions/mime types, computes SHA256 hashes, and registers file metadata.

* **CLI Command**:
  ```bash
  python -m pipeline.step1_intake.cli --source-folder ./raw_docs --output-json ./step1_intake_result.json
  ```
* **Input**: Directory of raw files (`./raw_docs/Technical_RFP.pdf`)
* **Output (`step1_intake_result.json`) Example**:
  ```json
  {
    "run_id": "step1-intake-abc123",
    "source_folder": "/path/to/raw_docs",
    "scanned_entries": 1,
    "accepted_files": 1,
    "files": [
      {
        "file_name": "Technical_RFP.pdf",
        "absolute_path": "/path/to/raw_docs/Technical_RFP.pdf",
        "extension": ".pdf",
        "mime_type": "application/pdf",
        "size_bytes": 1048576,
        "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
      }
    ]
  }
  ```

---

## 📌 STEP 1b: Markdown Conversion (`pipeline/step1_markdown`)

### 📖 Description
Converts binary documents into clean Markdown files containing explicit page markers (`<!-- PAGE: N -->`), preserving layout semantics.

* **CLI Command**:
  ```bash
  python -m pipeline.step1_markdown.cli --intake-result ./step1_intake_result.json --output-dir ./markdown
  ```
* **Input**: Raw PDF/DOCX files from Step 1a.
* **Output File (`Technical_RFP.pdf.md`) Example**:
  ```markdown
  <!-- PAGE: 1 -->
  # Request for Proposal: Cloud Migration Project
  ...
  <!-- PAGE: 45 -->
  # 4. Security and Compliance Requirements
  All cloud environments must support ISO 27001, TLS 1.3 encryption, and Role-Based Access Control...
  <!-- PAGE: 62 -->
  ```

---

## 📌 STEP 2: Document Classification (`pipeline/step2_classification`)

### 📖 Description
Uses an Azure AI Foundry agent to analyze the Table of Contents (TOC) or initial pages of each Markdown document and classify its type (e.g., `technical_spec`, `rfp_main`, `commercial`, `legal`, `questionnaire`).

* **CLI Command**:
  ```bash
  python -m pipeline.step2_classification.cli --markdown-folder ./markdown --output-json ./step2_classification_result.json
  ```
* **Input**: Markdown files in `./markdown`
* **Output (`step2_classification_result.json`) Example**:
  ```json
  {
    "run_id": "step2-class-def456",
    "classified_files": 1,
    "files": [
      {
        "source_document_name": "Technical_RFP.pdf",
        "markdown_path": "./markdown/Technical_RFP.pdf.md",
        "document_type": "rfp_main",
        "confidence": 0.98,
        "reasoning": "Document contains formal RFP structure, scope of work, and compliance tables."
      }
    ]
  }
  ```

---

## 📌 STEP 3: Section Detection (`pipeline/step3_section_detection`)

### 📖 Description
Analyzes the page-marked Markdown files using an AI agent to identify logical section boundaries, extracting each section's title, start page, end page, and hierarchy level.

* **CLI Command**:
  ```bash
  python -m pipeline.step3_section_detection.cli --markdown-folder ./markdown --output-json ./step3_section_detection_result.json
  ```
* **Input**: Markdown files in `./markdown`
* **Output (`step3_section_detection_result.json`) Example**:
  ```json
  {
    "run_id": "step3-sec-ghi789",
    "processed_files": 1,
    "files": [
      {
        "source_document_name": "Technical_RFP.pdf",
        "markdown_path": "./markdown/Technical_RFP.pdf.md",
        "status": "processed",
        "page_count": 62,
        "sections": [
          {
            "section_name": "Security and Compliance Requirements",
            "start_page": 45,
            "end_page": 62,
            "level": 1,
            "confidence": 0.95
          }
        ]
      }
    ]
  }
  ```

---

## 📌 STEP 4: Chunking & Indexing (`pipeline/step4_chunking_indexing`)

### 📖 Description
Slices Markdown sections into token-bounded chunks (e.g. ~240 words per chunk), generates vector embeddings via Azure OpenAI, and indexes them into Azure AI Search with metadata (`source_document_name`, `section_name`, `page_start`, `page_end`).

* **CLI Command**:
  ```bash
  python -m pipeline.step4_chunking_indexing.cli --markdown-folder ./markdown --step2-result ./step2_classification_result.json --step3-result ./step3_section_detection_result.json --output-json ./step4_chunking_result.json
  ```
* **Input**: Markdown files + Step 2 JSON + Step 3 JSON.
* **Output (`step4_chunking_result.json`) Example**:
  ```json
  {
    "run_id": "step4-chunks-jkl012",
    "total_chunks": 1,
    "indexed_chunks": 1,
    "chunks": [
      {
        "chunk_id": "ck-a1b2c3d4e5",
        "source_document_name": "Technical_RFP.pdf",
        "section_name": "Security and Compliance Requirements",
        "page_start": 45,
        "page_end": 48,
        "token_estimate": 220,
        "text": "All cloud environments must support ISO 27001, TLS 1.3 encryption, and Role-Based Access Control..."
      }
    ]
  }
  ```

---

## 📌 STEP 6: Knowledge Understanding Layer (`pipeline/step6_file_summary`)

### 📖 Description
Reads Markdown files page-by-page. Using the section boundaries from `step3_result.json`, it:
1. **6.1 Section Summaries**: Summarizes each section independently.
2. **6.2 File Summary**: Summarizes the section summaries into one overall document summary.
3. **6.3 Master Manifest**: Compiles all summaries into `step6_manifest.json`.

* **CLI Command**:
  ```bash
  python -m pipeline.step6_file_summary.cli --markdown-folder ./markdown --step2-result ./step2_classification_result.json --step3-result ./step3_section_detection_result.json --output-json ./step6_manifest.json
  ```
* **Inputs**: `.md` files + `step2_classification_result.json` + `step3_section_detection_result.json`
* **Output (`step6_manifest.json`) Example**:
  ```json
  {
    "run_id": "step6-manifest-mno345",
    "processed_files": 1,
    "files": [
      {
        "source_document_name": "Technical_RFP.pdf",
        "markdown_path": "./markdown/Technical_RFP.pdf.md",
        "document_type": "rfp_main",
        "overall_summary": "This document defines functional scope and mandates strict ISO 27001 cloud security standards.",
        "sections": [
          {
            "section_name": "Security and Compliance Requirements",
            "start_page": 45,
            "end_page": 62,
            "summary": "Mandates ISO 27001 compliance, TLS 1.3 encryption, audit logging, and RBAC authentication."
          }
        ]
      }
    ]
  }
  ```

---

## 📌 STEP 7: Bid Brief Planning & Mapping (`pipeline/step7_bid_mapping`)

### 📖 Description
Evaluates each section requirement of the target proposal template (the "Bid Brief") against the master manifest (`step6_manifest.json`) to determine which source files and sections contain the necessary evidence.

* **CLI Command**:
  ```bash
  python -m pipeline.step7_bid_mapping.cli --manifest-json ./step6_manifest.json --bid-template ./bid_template.json --step4-result ./step4_chunking_result.json --output-json ./step7_bid_mapping.json
  ```
* **Inputs**: `step6_manifest.json` + `bid_template.json` + `step4_chunking_result.json`
* **Output (`step7_bid_mapping.json`) Example**:
  ```json
  {
    "run_id": "step7-mapping-pqr678",
    "bid_sections": [
      {
        "bid_section_id": "sec-3-security",
        "bid_section_title": "3. Security and Risk Management",
        "mapped_sources": [
          {
            "source_document_name": "Technical_RFP.pdf",
            "section_name": "Security and Compliance Requirements",
            "start_page": 45,
            "end_page": 62,
            "relevance_score": 0.95,
            "reasoning": "Contains explicit ISO 27001 encryption and authentication mandates."
          }
        ]
      }
    ]
  }
  ```

---

## 📌 STEP 8: Content Generation Agent (`pipeline/step8_content_generation`)

### 📖 Description
Executes RAG content drafting section-by-section. It pulls the text chunks mapped in Step 7, passes them as context to Azure OpenAI, generates professional proposal section content, and compiles `final_bid_brief.md`.

* **CLI Command**:
  ```bash
  python -m pipeline.step8_content_generation.cli --mapping-json ./step7_bid_mapping.json --step4-result ./step4_chunking_result.json --output-json ./step8_generation_result.json --output-md ./final_bid_brief.md
  ```
* **Inputs**: `step7_bid_mapping.json` + `step4_chunking_result.json`
* **Output (`step8_generation_result.json`) Example**:
  ```json
  {
    "run_id": "step8-gen-stu901",
    "sections": [
      {
        "bid_section_id": "sec-3-security",
        "bid_section_title": "3. Security and Risk Management",
        "content": "### 3. Security and Risk Management\n\nOur proposed platform strictly aligns with the requirements set forth in `Technical_RFP.pdf`. Key security capabilities include:\n- **ISO 27001 Compliance**: Full audit logging and security policy compliance.\n- **Data Protection**: End-to-end TLS 1.3 encryption in transit.\n- **Access Control**: Role-Based Access Control (RBAC) integrated with enterprise SSO."
      }
    ]
  }
  ```
* **Final Output (`final_bid_brief.md`)**:
  ```markdown
  # Final Bid Brief Proposal

  ## 1. Executive Summary
  ...

  ## 3. Security and Risk Management
  Our proposed platform strictly aligns with the requirements set forth in `Technical_RFP.pdf`. Key security capabilities include:
  - **ISO 27001 Compliance**: Full audit logging and security policy compliance.
  - **Data Protection**: End-to-end TLS 1.3 encryption in transit.
  - **Access Control**: Role-Based Access Control (RBAC) integrated with enterprise SSO.
  ```

---

## 🎯 Summary for Team Presentation

| Step | Module Name | Primary Input | Primary Output | Main Function |
|:---|:---|:---|:---|:---|
| **1a** | `step1_intake` | Raw file folder | `step1_intake_result.json` | Scans directory, calculates file hashes, filters extensions |
| **1b** | `step1_markdown` | Raw files | `.md` files with `<!-- PAGE: N -->` | Converts documents into page-marked Markdown |
| **2** | `step2_classification` | `.md` files | `step2_classification_result.json` | Classifies document type via TOC/header analysis |
| **3** | `step3_section_detection` | `.md` files | `step3_section_detection_result.json` | AI detects sections, levels, and start/end page boundaries |
| **4** | `step4_chunking_indexing` | `.md` + Step 2/3 JSON | `step4_chunking_result.json` | Chunks text, computes embeddings, indexes into Azure Search |
| **6** | `step6_file_summary` | `.md` + Step 2/3 JSON | `step6_manifest.json` | Hierarchical summarization (sections -> files -> project manifest) |
| **7** | `step7_bid_mapping` | Manifest + Template | `step7_bid_mapping.json` | Maps proposal sections to target source files/sections |
| **8** | `step8_content_generation` | Mapping + Chunks | `final_bid_brief.md` | Grounded RAG content drafting per section & document compilation |
