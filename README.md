# 3GPP Change Detector

## Problem Statement
3GPP specifications evolve across releases, introducing new features, modifying existing sections, and deprecating outdated elements.  
Manually identifying and comparing these changes across versions is a labor-intensive and error-prone task.  
Engineers and analysts need an automated system to efficiently detect and summarize differences between versions and to answer specific change-related queries in natural language.

---

## Overview

![Overview Diagram](/images/pipeline.png) 

---

## 1. Document Parsing

**Purpose**  
Extract and structure content from Word documents while preserving hierarchical organization through numbered sections.

**How It Works**
- **First page extraction** – Identifies preamble content by stopping at table of contents or first numbered heading.  
- **Numbered heading detection** – Uses regex patterns to identify section headers (e.g., 1, 1.1, 1.2).  
- **Hierarchical structure building** – Creates parent-child relationships based on section numbering depth.  
- **Content aggregation** – Combines paragraphs, tables, and headers under their respective sections.  
- **Two-phase processing** – Preamble handling followed by structured section-based parsing.

**Tools / Libraries**
- `python-docx` – read paragraphs, tables, styles, and core properties
- `re`, `json`, `collections` (stdlib) – pattern detection, serialization, grouping

---

## Hybrid Chunking

**Purpose**  
Break large text into smaller, manageable segments (chunks) for easier processing in later stages (e.g., embeddings, retrieval).

**Hybrid Chunking Approach**
- Combines hierarchical chunking (splitting by document structure) with semantic chunking (splitting by meaning or sentence boundaries).
- Ensures chunks are contextually complete and size-optimized for processing.
- Improves retrieval accuracy by preserving document hierarchy while keeping chunks manageable.

**Steps in Code**
1. Read Input Data from DOCX file.
2. Sentence Segmentation using spaCy (`doc = nlp(text)`).
3. Chunk Creation – Group sentences into chunks under `max_chunk_size`.
4. Store Chunks in memory for the next phase.

---

## 2. Change Detection

**Purpose**  
Identify and classify added, removed, modified, and moved chunks between two versions of a specification.

**How It Works**
- Group old/new chunks by `section_id`.
- Use `difflib.SequenceMatcher` on chunk lists to get delete, insert, replace operations.
- Classification:
  - **REMOVED** – Old chunk missing, not in `version_map`.
  - **MOVED** – Old chunk missing, mapped in `version_map`.
  - **ADDED** – New chunk not mapped from old.
  - **MODIFIED** – Same position, similarity < 0.85.
- Output `Change` objects with details and optional HTML diffs.

**Tools / Libraries**
- `difflib` (similarity, HTML diff), `dataclasses` (structured records), `defaultdict`, `enum`, `os`

---

## 3. Version Mapping

**Purpose**  
Map each `old_chunk_id` → `new_chunk_id` (or None) to identify moved vs removed chunks.

**How It Works**
- Section-aware matching (compare only within same `section_id`).
- Scoring: `combined_similarity = 0.7 * title_ratio + 0.3 * content_ratio`
- Mapping: If score ≥ threshold (0.6), map; else None.
- Used by `ChangeDetector` to label MOVED vs REMOVED.

**Tools / Libraries**
- `difflib.SequenceMatcher`, `json`, `collections.defaultdict`

---

## 4. VectorDB Storage

**Purpose**  
Efficiently store, manage, and search change snippets and grouped events using vector similarity.

**How It Works**
- **Embed Text Data** – Using Sentence Transformers.
- **Index with FAISS** – Chunk-level and event-level indexes for fast similarity search.
- **Persist & Query** – Save indexes, metadata, versions; retrieve top matches.

**Tools / Libraries**
- FAISS, Sentence Transformers, NumPy, Pickle, JSON

---

## 5. QA Bot Integration

**Purpose**  
Implement a hybrid retrieval-augmented QA assistant that uses thematic clustering and chunk-level search to answer queries about 3GPP changes.

**How It Works**
- Loads vector DB, old/new chunks, clustered change events.
- Detects counting questions, generates summaries.
- Calls GroqLLM wrapper for natural language answers.

**Tools / Libraries**
- `json`, `os`, `langchain.schema.HumanMessage`, custom `VectorDB`, Groq SDK

---

## 6. Streamlit UI

**Purpose**  
Expose an API and interactive Streamlit UI to query and explore 3GPP changes via natural language.

**How It Works**
- Loads configuration, version metadata, FAISS DB.
- Instantiates QABot.
- FastAPI `/qa` endpoint serves answers.
- Streamlit chat interface with example queries and session stats.

**Tools / Libraries**
- FastAPI, pydantic, yaml, dotenv, Streamlit, requests

---

## Testing

![Testing Screenshot](/images/test%20case.png) <!-- Replace with actual testing image -->

---

## Dataset
*The dataset used consists of parsed DOCX versions of 3GPP TS 24.301 Release 10 (v10.15.0) and Release 17 (v17.12.0) specifications.*

