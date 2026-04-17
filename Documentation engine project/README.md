# Production-Ready Documentation Search Engine

Built by Denis Mbugua
LinkedIn: https://www.linkedin.com/in/dynamo-denis-mbugua-53304b197/

---

## What This Project Does

This is a documentation search engine built on top of Qdrant that goes beyond simple keyword matching. When you search for something like "how to configure HNSW parameters," the system understands both the meaning behind your query and the exact technical terms in it, then uses a third layer of fine-grained reranking to surface the most relevant section, not just the most relevant page.

The system indexes the entire Qdrant documentation website, chunks it intelligently, and makes it searchable through a hybrid pipeline that combines three different retrieval signals at query time.

---

## Why Three Vectors Per Document

Most search systems use one type of embedding. This project uses three, each capturing something different about the text.

The dense vector from BAAI/bge-small-en-v1.5 represents the overall semantic meaning of a chunk. It understands that "parmesan" and "grated hard cheese" are related concepts even when the exact words do not overlap.

The sparse vector from BM25 handles exact keyword matching. When someone searches for a specific function name, configuration key, or API parameter, sparse retrieval finds it precisely and ranks it by how rare and important that term is across the entire corpus.

The ColBERT multivector does something neither of the above can do cleanly. Instead of representing a whole chunk as one vector, it stores one 128-dimensional vector per token. At query time, each query token independently finds its best matching token in the document, and the scores are summed. This catches nuanced matches that single-vector approaches miss.

At search time, dense and sparse retrieval each fetch 50 candidates independently. Those candidates are merged and the ColBERT model reranks the combined pool to produce the final results. This is why ColBERT is configured with m=0, it does not need its own HNSW graph because it only ever scores a small candidate set, never the full collection.

---

## Chunking Strategy

Documentation text has natural structure that most generic chunkers ignore. This project uses a three-layer approach that respects that structure.

Layer one splits on Markdown headings. Every section under a heading becomes a candidate chunk, and heading context is always prepended so each chunk is self-contained and meaningful on its own.

Layer two handles sections that are too long for the embedding models but do not contain major topic shifts. It splits on sentence boundaries with a two-sentence overlap so context is not lost at chunk edges.

Layer three uses semantic similarity to detect when a long section actually contains multiple distinct topics. It runs the all-MiniLM-L6-v2 model to find natural break points and only activates when a section exceeds 200 words. If semantic splitting produces a chunk that is still too long, it falls back to layer two automatically.

After chunking, any chunk under 40 words is merged with its neighbour. Tiny orphaned chunks carry little retrievable meaning and add noise to the index.

The result of chunking is not just the text. Each chunk carries its page title, section title, the URL with an anchor fragment pointing to the exact heading, breadcrumb trail, adjacent section text for context, and tags derived from the URL path and page metadata.

---

## Ingestion Pipeline

The ingestion process is designed to be fast and then precise, not both at once.

When the collection is created, HNSW index building is disabled entirely by setting m=0 and indexing_threshold=0. This means uploads are just writes to disk with no graph construction happening in parallel. Vectors are uploaded in batches of 16 to stay within Qdrant's 32MB per-request payload limit.

Once all documents are uploaded, the collection is updated to enable HNSW with m=16 and ef_construct=100, and the indexing threshold is set to 1000. Qdrant then builds the graph once across the full dataset, which is faster and produces a better-quality graph than building it incrementally during upload.

INT8 scalar quantization is applied to the dense vectors. This compresses each 32-bit float to an 8-bit integer, reducing memory usage by a factor of four with less than one percent recall loss in practice. The quantized vectors are kept in RAM while the full-precision originals stay on disk for final rescoring.

The sparse inverted index is also kept in RAM. This is the lookup structure that maps token indices to document IDs. Keeping it in memory avoids disk reads on every keyword lookup, which would be the main bottleneck for sparse retrieval at scale.

---

## Collection Configuration

```
Collection: docs_search

Dense vector
  Model:          BAAI/bge-small-en-v1.5
  Dimensions:     384
  Distance:       Cosine
  HNSW:           m=16, ef_construct=100
  Quantization:   INT8 scalar, always_ram=True

ColBERT multivector
  Model:          colbert-ir/colbertv2.0
  Dimensions:     128 per token
  Distance:       Cosine
  Comparator:     MaxSim
  HNSW:           m=0 (reranking only, no ANN graph built)
  on_disk:        False

Sparse vector
  Model:          BM25 via FastEmbed
  Index:          on_disk=False (inverted index in RAM)
  IDF modifier:   Enabled (Qdrant computes IDF at query time)

Payload indexes:  section, url, page_title (keyword), chunk_index (integer)
```

---

## Search Pipeline

A search request goes through these steps:

1. The query string is embedded with all three models simultaneously.
2. Dense prefetch retrieves the 50 most semantically similar chunks using the HNSW graph.
3. Sparse prefetch retrieves the 50 best keyword matches using the inverted index.
4. The combined candidate pool is passed to ColBERT, which computes MaxSim scores between query tokens and document tokens.
5. ColBERT reranks the pool and returns the top results.

The prefetch limits are intentionally set to 50 so ColBERT always has a meaningful candidate set to rerank, even when dense and sparse overlap heavily.

---

## Evaluation

The system was evaluated against five ground truth queries covering the main ways people actually use documentation: configuration how-to questions, conceptual comparisons, API usage patterns, filtering and search logic, and operational tasks like backups and recovery.

Each query was run through the full hybrid pipeline and the rank of the first correct result was recorded. Mean Reciprocal Rank was used as the primary metric because it directly measures how far down a user has to scroll before finding what they need.

Results:

```
Query                                                    Rank    RR Score
------------------------------------------------------------------------
how to configure HNSW parameters for better recall       1       1.000
scalar vs binary quantization memory reduction           2       0.500
create collection with replication factor and shards     1       1.000
combining vector search with boolean payload filters     2       0.500
restore a collection from a remote snapshot url          1       1.000
------------------------------------------------------------------------
Mean Reciprocal Rank (MRR):                                      0.8000
```

Three of the five queries returned the correct page as the top result. The two that ranked second were still found within the top two positions, meaning a user would see the right answer immediately in every case. An MRR of 0.80 on a five-query sample is a strong result for a system with no query-time fine-tuning.

The quantization and filtering queries landed at position two rather than one. This is expected behaviour. The quantization query matches content spread across multiple sections of the same page, and the filtering query has relevant content in both the filtering page and the hybrid queries page. In both cases the system surfaced relevant results, just with a different page ranked first.

---

## How to Improve Further

The current MRR of 0.80 is solid but not the ceiling. A few directions would push it higher.

Increasing the prefetch limit from 50 to 100 gives ColBERT more candidates to rerank and improves recall on queries where the correct result sits just outside the current window.

Setting a search-time hnsw_ef parameter higher than the default would improve the quality of the dense candidate set at the cost of slightly higher latency. For a documentation search tool where latency is secondary to precision, this is a reasonable trade.

The two queries that landed at position two instead of one both involve pages with multiple relevant sections. Payload filters on the section field could let users scope searches to specific parts of the documentation, which would improve precision on those queries significantly.

For a production deployment with significantly more documents, replacing BM25 sparse with SPLADE++ would improve semantic coverage on keyword queries at the cost of slower embedding at index time. The current BM25 setup is faster and works well for technical documentation where users tend to search with exact terms.

---

## Dependencies

```
qdrant-client~=1.15.1
fastembed~=0.7.3
numpy
requests
beautifulsoup4
markdownify
lxml
llama_index
llama-index-embeddings-huggingface
```

---

## Project Structure

```
collection creation      create_collection() with quantization and IDF modifier
scraping                 get_qdrant_urls() + scrape_page() via sitemap
chunking                 3-layer: structural + sentence + semantic
embedding                TextEmbedding + SparseTextEmbedding + LateInteractionTextEmbedding
ingestion                batch upsert with HNSW disabled, then rebuild after
search                   prefetch dense+sparse, rerank with ColBERT
display                  format_snippet() + score_to_label() + display_results()
evaluation               MRR against 5 ground truth query-URL pairs
```
