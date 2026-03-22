# Part 2: Data Loading, Preparation & Embedding Generation

## Overview

In this part you will:
1. Stream 1,000 ArXiv research papers from Hugging Face
2. Clean and structure the data into a DataFrame
3. Generate vector embeddings using a local sentence-transformer model

## 2.1 Data Loading From Hugging Face

We use `load_dataset` from the `datasets` library with `streaming=True` to avoid downloading the entire dataset into memory. This lets you work with large datasets efficiently.

**Key points:**
- The dataset is `nick007x/arxiv-papers`
- We extract: `arxiv_id`, `title`, `abstract`, `authors`, and a combined `text` field
- Authors are normalized to a list of strings regardless of input format

## 2.2 Embedding Generation

We use the **nomic-ai/nomic-embed-text-v1.5** model from sentence-transformers. This is a 768-dimensional model that runs locally — no API key required.

**Important detail:** Nomic embeddings use a prefix scheme:
- `search_document:` for documents being indexed
- `search_query:` for queries at retrieval time

This asymmetric prefixing improves retrieval quality by signaling intent to the model.

**What happens:**
1. Each document text gets prefixed with `search_document:`
2. The model encodes each text into a 768-dim normalized vector
3. Embeddings are stored as `float32` lists in the DataFrame

The resulting dimension (768) determines the `VECTOR` column size in Oracle.

## No TODOs in This Part

This section is pre-built. Read through the code to understand how data flows from Hugging Face into embeddings ready for Oracle ingestion.
