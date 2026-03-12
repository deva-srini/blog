---
layout: post
title: "KnowledgeForge: A Minimal Document ETL Framework for RAG Systems"
date: 2026-03-01
categories: projects rag
---

Building a RAG system is straightforward until you realize that the hard part isn't just the retrieval or the generation -- it's curating the knowledge that feeds it. How do you go from a folder of PDFs, Word docs, and PowerPoint decks to a clean, chunked, indexed knowledge base? And how do you track what happened to each document along the way?

KnowledgeForge grew out of that process -- not as a planned tool, but as the thing I kept reaching for while working through each of these problems. What started as a few scripts to wrangle PDFs turned into a structured ETL framework: a sequential pipeline that takes documents in, processes them through well-defined stages, and produces indexed, queryable chunks in a vector database -- with full lineage tracking from source file to stored embedding.

## KnowledgeForge Background

The questions I kept running into were mundane but persistent:

- Which version of this PDF is actually in my vector store right now?
- Did the chunking step fail silently on some documents?
- How do I reprocess a document when the source file changes?

Solving each of them individually left me with a growing set of scripts and utilities that were clearly trying to be something more coherent. It's built around [Docling](https://github.com/docling-project/docling) for deep document understanding -- layout analysis, table structure recognition, OCR -- and layers on top of it the operational pieces I kept needing.

Here's what that turned into:

- **Automated file watching** — a folder watcher that detects new and changed documents and triggers the full pipeline automatically, no manual intervention needed
- **SHA-256 versioning** — every document is fingerprinted on arrival; unchanged files are skipped, changed files get a new version and a clean re-index, while the full history stays intact in the database
- **Workflow system** — multiple document workflows (e.g. `fund_factsheets`, `annual_reports`) run concurrently with independent watch folders, chunk sizes, and ChromaDB collections, hot-reloaded without restarting the service
- **Post-indexing analysis** — two standalone tools for auditing what's in the knowledge base: a vector index summarizer that produces a structured TOML snapshot of all collections, and an overlap detector that finds semantically duplicate content within or across collections using cosine similarity

## Architecture Overview

The core idea is a **six-stage sequential pipeline** that every document passes through, followed by two post-indexing analysis tools:

```
Source File  -->  Parse  -->  Extract  -->  Transform  -->  Chunk  -->  Embed  -->  Index
   (PDF)        (Docling)   (content     (normalize     (structure   (vectors)   (ChromaDB)
                             traversal)   + cleanup)      aware)

                                                      Post-indexing analysis:
                                                      Summarize  -->  Overlap Detection
                                                      (TOML)          (cosine similarity)
```

Each stage is independently toggleable, tracked in a SQLite database, and produces metadata that forms the document's processing lineage. If any stage fails, the pipeline stops and records exactly where and why.

## File Watcher and Ingestion

Processing starts when a file lands in the watch folder. KnowledgeForge uses the `watchdog` library to monitor a configured directory:

```yaml
# kf_config.yaml
source:
  watch_folder: "./data/source"
  staging_folder: "./data/staging"
  file_patterns: ["*.pdf", "*.docx", "*.xlsx", "*.html", "*.pptx"]
```

When a file is detected:

1. **SHA-256 hash** is computed for the file content
2. If the hash matches an existing record, processing is **skipped** (deduplication)
3. If the content changed, the **version is incremented** (e.g., `report.pdf` v1 becomes `report_v2.pdf` in staging)
4. A `Document` record is created in SQLite with status `pending`
5. The workflow pipeline is triggered automatically

This means you can update a source file and KnowledgeForge will detect the change, bump the version, and reprocess it -- while the old version's lineage remains intact in the database.

## The Six Pipeline Stages

### 1. Parsing

The parser uses Docling to break a document into its structural elements. This isn't simple text extraction -- Docling performs layout analysis, recognizes table structures (via TableFormer), and identifies content types like headers, text blocks, tables, and images.

```yaml
processing:
  parsing:
    library: "docling"
    pipeline: "standard"    # or "vlm" for scanned/complex documents
```

The output is a structured representation: page count, estimated tokens, content type counts, and a per-page breakdown. For a typical fund factsheet PDF, this might produce:

```
4 pages, 3373 tokens, 98 items (84 text, 4 tables, 10 images)
```

There's also a VLM (Vision Language Model) pipeline option for scanned or complex-layout documents, which uses models like `granite_docling` for improved parsing accuracy at the cost of speed.

### 2. Extraction

The extractor walks the parsed document tree in reading order, maintaining a **header stack** to build hierarchical paths. Each content item gets classified:

- **Text blocks** are extracted with their header context (e.g., `"Fund Overview > Investment Strategy"`)
- **Tables** are exported to markdown format with metadata like row/column counts and captions
- **Images** get their captions, classifications, and optionally -- AI-generated descriptions

The image description feature is where the VLM capability comes in. When enabled, KnowledgeForge saves extracted images to disk and calls Claude's vision API to generate descriptions of charts, diagrams, and figures:

```yaml
processing:
  extraction:
    save_picture_images: true
    describe_pictures: true        # requires ANTHROPIC_API_KEY
    vision_model: "claude-haiku-4-5-20251001"
```

This turns an opaque "Figure 3" placeholder into a description like *"Bar chart showing quarterly returns from Q1 2024 to Q4 2025, with the fund (blue) consistently outperforming the benchmark (gray) by 1-2 percentage points."*

### 3. Transformation

The transformation stage normalizes and cleans the extracted content:

- **Unicode normalization**: smart quotes to standard quotes, em-dashes to hyphens, removal of zero-width characters
- **Whitespace cleanup**: multiple spaces collapsed, leading/trailing whitespace stripped
- **Table formatting**: consistent markdown pipe alignment
- **Markdown generation**: optionally produces a full structured markdown document for downstream use

This is a simple but important stage. Raw document extraction often contains encoding artifacts that can subtly degrade embedding quality.

### 4. Chunking

This is where KnowledgeForge gets opinionated. The `StructureAwareChunker` doesn't just split text at arbitrary token boundaries. It respects document structure:

- **Groups content by header path**: consecutive items under the same header stay together
- **Tables and images are standalone chunks**: a table is never split across chunks
- **Text is split at semantic boundaries**: uses the `semchunk` library for sentence-level splitting within token limits
- **Small documents skip chunking**: if total tokens are below a threshold (default 1000), the entire document becomes a single chunk

```yaml
processing:
  chunking:
    strategy: "structure_aware"
    chunk_size_tokens: 512
    chunk_overlap_tokens: 50
    skip_threshold_tokens: 1000
```

The overlap is applied between consecutive text chunks -- trailing tokens from the previous chunk are prepended to the next. This preserves context at boundaries, which is critical for retrieval quality.

Each chunk gets a deterministic sequential index, which feeds into the indexing stage's upsert logic.

### 5. Embedding

The embedder generates vector representations using `sentence-transformers` (default model: `all-MiniLM-L6-v2`, 384 dimensions). It processes chunks in configurable batches (default 32), with automatic fallback to individual embedding if a batch fails.

The embedding provider is pluggable -- the architecture uses an abstract `EmbeddingClient` base class, so swapping in an API-based provider (OpenAI, Cohere) requires implementing a single interface.

### 6. Indexing

Finally, embedded chunks are upserted into ChromaDB with deterministic IDs (`{document_id}_{chunk_index}`). This means:

- Reprocessing a document **updates** existing chunks rather than creating duplicates
- Version bumps produce clean re-indexing (old chunks deleted, new ones inserted)

Each chunk is stored with rich metadata: document ID, file name, version, page number, chunk index, content type, header path, and token count.

Workflows can route documents to different collections using fnmatch patterns:

```yaml
indexing:
  default_collection: "fund_factsheets"
  collection_mapping:
    "reports/*.pdf": "reports_collection"
```

## Post-Indexing Analysis

Once documents are indexed, two questions come up almost immediately: *what's actually in my vector store?* and *is there redundant content that will confuse retrieval?* These are hard to answer by querying ChromaDB directly -- you'd have to iterate through collections, aggregate metadata, and eyeball similarity yourself. I added two standalone tools to handle this.

### Vector Index Summarizer

The `VectorIndexSummarizer` scans all ChromaDB collections and produces a structured TOML summary of everything that's been indexed:

```bash
python cli.py summary
python cli.py summary --output ./reports/index_snapshot.toml
```

For each collection it reports chunk counts, content types, and embedding metadata. For each document within a collection it breaks down chunk count, total tokens, content types present, and the section headers found. Running it against a freshly indexed factsheet produces something like:

```
Collections scanned: 1
Total chunks       : 33
Embedding model    : sentence-transformers/all-MiniLM-L6-v2
Embedding dim      : 384

Collection: 'fund_factsheets'
  Document: 'bgf_factsheet.pdf' (v1)
    Chunks   : 33
    Tokens   : 3,048
    Types    : ['image_description', 'table', 'text']
    Sections : ['CUMULATIVE & ANNUALISED PERFORMANCE', 'INVESTMENT OBJECTIVE', ...]
```

The TOML output is designed to be machine-readable -- you can pass it as context to a RAG system's LLM so it knows upfront what knowledge is available before it even starts retrieving. Instead of blind retrieval, the model can say *"I have fund factsheets covering X, Y, Z topics"* before answering.

You can also attach human-readable descriptions per document in the config, which get embedded into the summary:

```yaml
indexing:
  document_descriptions:
    bgf_factsheet.pdf: "BlackRock World Technology Fund factsheet, December 2025"
```

### Overlap Detection

The `OverlapDetector` answers a different question: are there chunks in my knowledge base that are saying the same thing? This matters more than it sounds. Duplicate or near-duplicate content inflates retrieval scores for certain topics, makes your RAG answers repetitive, and is usually a sign that you've indexed the same document twice under different names or versions.

```bash
# Check for overlaps within a single collection
python cli.py overlap --collection fund_factsheets

# Compare two collections against each other
python cli.py overlap --source fund_factsheets --target annual_reports --threshold 0.9
```

It works by fetching all embeddings from the collection, running a batch cosine similarity query, and filtering to pairs above the threshold. Symmetric duplicates are deduplicated so you don't see A→B and B→A reported separately. The output is a ranked list of overlapping chunk pairs with a similarity score and a content preview:

```
Overlap Report: within-collection "fund_factsheets"
  Chunks analyzed: 33
  Threshold: 0.85
  Overlaps found: 4

  Matches:
    1. [0.97] doc_abc_5 (factsheet_q3.pdf) ~ doc_xyz_5 (factsheet_q4.pdf)
       "The Fund invests globally at least 70% of its total assets..."  ~
       "The Fund invests globally at least 70% of its total assets..."
```

A similarity of 1.0 almost always means the exact same content indexed twice -- usually from processing the same document under different versions or filenames. Scores in the 0.85-0.95 range are more interesting: similar but not identical, which might indicate boilerplate that appears across multiple documents or sections that were updated between versions.

Both tools run entirely post-hoc and are decoupled from the ingestion pipeline -- you can run them on any existing ChromaDB collection without reprocessing anything.

## Multi-Workflow Support

A single KnowledgeForge instance can run multiple workflows, each with its own watch folder, processing settings, and target collection. Workflows are registered in a central registry:

```yaml
# workflows/registry.yaml
workflows:
  - name: "fund_factsheet"
    config: "fund_factsheet.yaml"
    active: true
    description: "Process fund factsheet PDFs"
```

Each workflow config is a **deep-merge overlay** on the base `kf_config.yaml`. For example, the fund factsheet workflow overrides chunk size and targets a specific collection:

```yaml
# workflows/fund_factsheet.yaml
source:
  watch_folder: "./data/source/factsheets"
  file_patterns: ["*.pdf"]

processing:
  chunking:
    chunk_size_tokens: 256
    chunk_overlap_tokens: 25

indexing:
  default_collection: "fund_factsheets"
```

Workflows support **hot-reload** -- the registry is synced every 30 seconds. Setting `active: false` stops a workflow's file watcher without requiring a restart.

## Metrics, Observability, and Versioning

Every processing run is fully tracked in three database tables:

- **Documents**: file metadata, version, hash, status (pending / processing / indexed / failed)
- **WorkflowRuns**: per-document run records with start/end timestamps, total chunks, total tokens, and error messages
- **WorkflowStages**: per-stage records (parse, extract, transform, chunk, embed, index) with individual status, timing, and metadata

Each stage records a metadata JSON blob with stage-specific details:

```
Parse:     { "pages": 4, "tokens": 3373 }
Extract:   { "items": 98 }
Transform: { "items": 98, "markdown_length": 12394 }
Chunk:     { "chunks": 33, "total_tokens": 3049 }
Embed:     { "model": "all-MiniLM-L6-v2", "dimension": 384 }
Index:     { "collection": "fund_factsheets", "indexed_ids": [...] }
```

This gives you full lineage: for any chunk in your vector store, you can trace it back through the exact processing steps, the document version, and the source file hash.

The CLI provides quick access to metrics:

```bash
python cli.py metrics
```

This surfaces aggregate stats: document counts by status, total chunks and tokens, average processing time, per-stage success/failure rates, and ChromaDB collection counts.

## End-to-End: What Happens When You Drop a File

Here's the full flow for a fund factsheet PDF:

```bash
# Start KnowledgeForge with all workflow watchers
python cli.py start

# Drop a file into the watched folder
cp bgf_factsheet.pdf data/source/factsheets/
```

Behind the scenes:

```
1. File watcher detects bgf_factsheet.pdf
2. SHA-256 hash computed -> new file -> version 1
3. Staged to data/staging/bgf_factsheet.pdf
4. Document record created (status: pending)
5. Pipeline triggered:
   Parse:     4 pages, 3373 tokens, 98 items
   Extract:   98 items with headers + table markdown
   Transform: Unicode normalization + markdown export
   Chunk:     33 chunks (256-token limit, 50 overlap)
   Embed:     33 x 384-dim vectors
   Index:     33 chunks -> "fund_factsheets" collection
6. Document status -> "indexed"
7. Lineage recorded for all 6 stages
```

Now update the same file:

```bash
cp bgf_factsheet_v2.pdf data/source/factsheets/bgf_factsheet.pdf
```

KnowledgeForge detects the hash change, bumps to version 2, deletes old chunks from ChromaDB, and reprocesses the updated file -- all automatically.

Once ingestion is done, you can inspect and audit the knowledge base from the CLI:

```bash
# Summarize all indexed collections to a TOML file
python cli.py summary
python cli.py summary --output ./reports/index_snapshot.toml

# Check for semantic overlaps within a collection
python cli.py overlap --collection fund_factsheets

# Compare two collections against each other
python cli.py overlap --source fund_factsheets --target annual_reports --threshold 0.9
```

## More Information

KnowledgeForge is open source. Check the [GitHub repository](https://github.com/deva-srini/knowledgeforge) for installation instructions, configuration reference, and the full source code.

---

*This is part of my ongoing exploration of building RAG systems. If you're working on similar problems, I'd love to hear about your approach.*
