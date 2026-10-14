# Chunking for RAG: Strategies, Trade-offs and Mistakes

Oct 14, 2026 · @Shahad Mohammed

How you split documents before indexing decides retrieval quality more than the embedding model or the prompt. There is no universal best chunk size, only a best one for your documents and your users' questions.

## Why chunking exists

Embedding models accept a limited number of tokens; anything past that limit is silently cut off. Chunking keeps each piece inside that limit while keeping it meaningful.

- **Too small:** each piece is precise but stripped of the context the LLM needs to reason.
- **Too large:** one vector has to represent several topics, so it matches none of them strongly.
- **One rule for everything fails:** a legal filing, a codebase and a blog post carry meaning in different shapes, so a single strategy hurts most of them.

## The main strategies

Recursive splitting is the sensible default; move up the table only when your benchmark shows a gap.

| Strategy | How it splits | Ingestion cost | Best for | Watch out for |
| --- | --- | --- | --- | --- |
| Fixed-size | Every N characters, words or tokens, optional overlap | Lowest | Prototypes, baselines, uniform plain text | Cuts sentences and ideas in half |
| Recursive | Paragraphs first, then lines, then spaces, until under the size limit | Low, no embedding calls | Most pipelines as a starting point | Chunk sizes vary somewhat |
| Semantic | Embeds adjacent sentences and breaks where similarity drops | Medium, embeds every sentence | Long text with clear topic shifts | More chunks, bigger index, gains are inconsistent |
| Structure-based | Headings, sections, paragraphs, tables; splits an element only if it is too big | Low to medium, needs good parsing | Legal, manuals, API docs, financial reports | Only works if extraction kept the structure |
| LLM-driven (agentic) | An LLM rewrites text into single-fact statements, then groups them | Highest cost and latency | Small, high-value corpora | Depends on prompt quality and model ability |

## Chunk size

Let the type of question decide the size, not a framework default.

- **Fact lookups favour small chunks.** One study cut chunks from 1,024 to 64 tokens and saw recall@1 rise by roughly 10 to 15 points on entity-heavy data, because each vector describes one tight idea.
- **Narrative or analytical questions favour larger chunks**, which keep the surrounding passage the answer depends on.
- **More context is not always better.** Past roughly 2,500 tokens of retrieved context, answer quality can start to drop, so good retrieval can still be undone by stuffing the prompt.

## Common production mistakes

A bad chunking choice degrades every later stage: retrieval, reranking and generation.

| Mistake | What goes wrong | Fix |
| --- | --- | --- |
| Chunks too small | Old tutorial defaults (around 200 characters) give the embedding model almost nothing to work with | Treat size as a tunable parameter and test it |
| Chunks too large | One vector blurs several topics and matches none well | Small-to-big: retrieve small chunks, send their larger parent to the LLM |
| Ignoring structure | Tables lose their headers, lists lose their headings, paragraphs break across pages | Chunk along headings, sections and tables; handle each file type on its own terms |
| Stripping metadata | A chunk becomes an orphan fragment that cannot be located or ranked well | Attach title, section heading and page number before embedding |
| Vector search only | Exact terms such as product names, IDs and domain jargon get missed | Hybrid search: BM25 plus vectors, merged with Reciprocal Rank Fusion |

## Emerging techniques

All three aim to stop chunks losing context at their edges, and all three trade extra ingestion cost for better retrieval.

| Technique | Idea | Reported result | Cost / constraint |
| --- | --- | --- | --- |
| Late chunking | Embed the whole document first, then cut it into chunks, so every chunk's vectors "saw" the full document | About 3% average relative gain over naive chunking on long documents (4 BeIR datasets) | Embedding model must fit the whole document in one pass |
| Contextual retrieval | An LLM writes a short note on where each chunk sits in its document, prepended before embedding | Fewer retrieval failures when combined with hybrid search and reranking | One LLM call per chunk at indexing time |
| Pseudo-Instruction Chunking (PIC) | A document-level summary guides where boundaries go, without an LLM call per chunk | hits@5 of 58.4 vs 54.5 fixed-size and 56.0 semantic (6 QA datasets) | Needs one summary per document |

## Infrastructure impact

Chunking is not only a quality decision; it also sizes your infrastructure.

&#91;embedded content: chunking cascade · 5 linked decisions\]

Finer chunks mean more vectors, a bigger HNSW index, and possibly a move from in-memory to disk-based indexes. Reranking can recover quality lost to poor chunking, but one study measured it at 9.2x the latency (0.22 s to 2.02 s), so fix chunking before leaning on a reranker.

## Where to start

1. Use recursive chunking as the baseline.
2. Write down what kinds of questions users actually ask (lookups vs analysis).
3. Treat strategy and chunk size as parameters you benchmark on your own corpus, not settings you copy.
4. Plan the vector index together with the chunking choice, since one drives the other.

## Source

- [Chunking for RAG: Strategies, tradeoffs & common mistakes](https://redis.io/blog/chunking-strategy-rag-pipelines/), Redis blog, Jim Allen Wallace, April 13, 2026. Summarised in my own words; figures are as reported there. The article's product claims about Redis are left out.
