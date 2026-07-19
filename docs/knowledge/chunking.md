# Document Chunking

This document describes how raw documents are split into chunks suitable for embedding and retrieval. It covers the semantic chunking algorithm, boundary detection, overlap strategy, storage schema, and integration with the upstream store stage and downstream embed stage.

## Why Chunking Is Necessary

Large language models and embedding models operate within finite context windows. A typical document from the scraping layer can be 10,000-50,000 characters -- far too large to embed as a single vector without losing semantic detail. A single embedding for an entire page about "Python asyncio" would dilute the signal from specific subtopics (event loops, coroutines, tasks, streams) into one averaged-out vector that matches none of them well.

Chunking solves this by splitting documents into segments of 500-1,500 characters, each small enough to produce a focused embedding vector, but large enough to carry meaningful context.

## Semantic Splitting

The `SemanticChunker` (defined in `tools/semantic_chunker.py`) splits text at natural boundaries rather than at arbitrary character positions. The algorithm differentiates between prose and code, applying different splitting strategies to each.

### Prose Documents

For non-code content, the chunker splits on paragraph boundaries:

```
Input: "Paragraph about event loops.\n\nParagraph about coroutines.\n\n..."
                                    ^                            ^
                                    split point                  split point
```

The split regex (`\n\s*\n` or `\n#{1,6}\s`) targets double newlines and Markdown headings. This ensures that splits happen between logical sections, never mid-paragraph.

### Code Documents

For code-heavy content (detected when 30%+ of lines match code indicators like indentation, braces, or function/class definitions), the chunker splits at function and class boundaries:

```
Input:
    def process_data(input_file):    <-- split point
        data = read_file(input_file)
        return clean(data)

    def clean(data):                 <-- split point
        data = data.dropna()
        return normalize(data)
```

The chunker detects programming language from structural patterns (e.g., `def`/`class` for Python, `function`/`const` for JavaScript) and includes this as metadata on each chunk.

### Chunk Size Control

After initial splitting, adjacent fragments are merged until they approach the maximum chunk size:

```
Parameters:
    min_chunk_size:  500 characters   (floor -- fragments below this merge upward)
    max_chunk_size: 1500 characters   (ceiling -- triggers forced split)
    overlap_size:    100 characters   (shared context between adjacent chunks)
```

If a single paragraph or code block exceeds `max_chunk_size`, it is force-split at that boundary. Fragments smaller than `min_chunk_size` are merged with the next fragment to avoid producing degenerate micro-chunks that carry too little context.

## Overlap

Adjacent chunks share an overlap region at their boundaries. The last `overlap_size` characters of chunk N are recorded as the `overlap_suffix` of chunk N and the `overlap_prefix` of chunk N+1.

```
Chunk 0:  [===================|overlap|]
Chunk 1:            [overlap|===================|overlap|]
Chunk 2:                       [overlap|===================]
```

**Why overlap matters:** Without overlap, a sentence spanning a chunk boundary would be split across two chunks, and neither chunk would contain the complete thought. With 100-character overlap, sentences near boundaries are preserved in at least one chunk. This improves retrieval quality when a query matches concepts expressed at a boundary.

## Chunk Size Tradeoff

The choice of chunk size involves a fundamental tradeoff:

| Chunk Size | Retrieval Precision | Context per Chunk | Total Chunks |
|-----------|--------------------|--------------------|--------------|
| Small (200-500 chars) | High -- tight semantic focus | Low -- may lose surrounding context | Many |
| Medium (500-1500 chars) | Balanced | Balanced | Moderate |
| Large (1500-3000 chars) | Low -- diluted by unrelated content | High -- full section context | Few |

The default configuration (500-1500 characters) targets the middle ground. It produces chunks large enough to carry a complete concept (a function definition, a configuration example, a conceptual explanation) but small enough that retrieval queries match specifically relevant content rather than entire pages.

## Memory Efficiency

The chunker provides both a batch interface (`chunk_text()`) and a streaming interface (`chunk_text_stream()`). The streaming variant uses a Python generator to yield chunks one at a time, avoiding allocation of the full chunk list in memory. This matters for documents approaching the 10MB size limit:

```python
chunker = SemanticChunker(max_chunk_size=1500, overlap_size=100)

# Streaming -- constant memory regardless of document size
for chunk in chunker.chunk_text_stream(large_document):
    process(chunk)

# Batch -- collects all chunks into a list (backward-compatible)
all_chunks = chunker.chunk_text(large_document)
```

Documents exceeding `max_text_size` (default: 10MB) are rejected with a `ValueError` rather than silently producing degraded output.

## Storage Schema

Chunks are stored in the `knowledge.chunks` table:

```sql
CREATE TABLE knowledge.chunks (
    id            SERIAL PRIMARY KEY,
    document_id   INTEGER NOT NULL,          -- FK to knowledge.documents
    chunk_index   INTEGER NOT NULL,          -- position within document
    content       TEXT NOT NULL,             -- the chunk text
    content_hash  TEXT NOT NULL,             -- SHA-256 for deduplication
    token_count   INTEGER,                  -- approximate token count
    created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(document_id, chunk_index)         -- no duplicate chunks
);
```

The `(document_id, chunk_index)` uniqueness constraint guarantees idempotent re-chunking: if a document is re-processed, its chunks are replaced cleanly rather than duplicated.

A GIN index on a `tsvector` column (where available) enables full-text search over chunk content as a complement to vector similarity search.

## Pipeline Integration

Chunking occupies the middle position in the linear pipeline:

```
[store queue] --> Store --> [chunk queue] --> Chunk --> [embed stage]
```

When the store stage completes for a document, it auto-enqueues a chunk job into the Redis `chunk` queue. The chunk worker:

1. Fetches the document content from PostgreSQL.
2. Runs `SemanticChunker.chunk_text()` to produce chunk dicts.
3. Inserts each chunk into `knowledge.chunks` with its content hash and index.
4. Auto-enqueues the resulting chunk IDs to the embed stage.

This ensures that every stored document is automatically chunked and queued for embedding without manual intervention. If chunking fails (malformed content, size limit exceeded), the failure is logged and the pipeline continues with remaining documents.
