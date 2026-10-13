# How RAG Works: All the Steps

Oct 13, 2026 · @Shahad Mohammed

## Phase 1: Indexing (offline, once per document)

1. **Load and parse**
   - Pull in your sources: PDFs, scans, HTML, DB rows, and so on.
   - Run OCR on scanned documents and extract the text and tables.
   - Clean the output: remove headers, footers, and page numbers, fix broken lines, and normalize text. For Arabic this includes alef variants, diacritics, and tatweel.
   - Keep the structure (headings, sections, tables) because the next step depends on it.
   - Bad extraction here can't be fixed later. Most RAG quality problems start at this step.
2. **Chunk**
   - Split documents into retrievable pieces: structure-aware, or recursive splitting at \~512 tokens.
   - Attach metadata to each chunk: doc ID, title, date, section path, page, and source URL.
   - Optionally prepend a short context line ("This chunk is from Circular X, section Y…"), known as contextual retrieval.
3. **Embed**
   - Each chunk passes through an embedding model, which outputs a vector (e.g. 1024 numbers).
   - Chunks with similar meaning get similar vectors, even when they share no exact words.
   - Pick a multilingual model that handles Arabic well, such as BGE-M3, multilingual-e5, or Cohere/OpenAI multilingual models.
4. **Index**
   - Store the vectors in a vector database such as pgvector, Qdrant, Milvus, Weaviate, or FAISS, using an ANN index like HNSW for fast search.
   - Also build a keyword index (BM25) over the same chunks. Together these give you hybrid search.
   - Store the chunk text and metadata alongside so you can filter and cite.

## Phase 2: Query time (every question)

1. **Process the query**
   - Optionally rewrite the question: fix spelling, expand abbreviations, or resolve "it/that" from chat history.
   - Advanced variants include multi-query (several rephrasings) and HyDE (embedding a hypothetical answer).
   - Embed the query with the same embedding model used for the chunks.
2. **Retrieve**
   - Vector search finds the top-k most similar chunks (often k = 20–50), typically by cosine similarity.
   - BM25 search finds exact-term matches such as document numbers, names, and legal terms.
   - Fuse the two lists, usually with Reciprocal Rank Fusion.
   - Apply metadata filters where relevant (date range, department, doc type).
3. **Rerank**
   - A cross-encoder reranker (e.g. bge-reranker, Cohere Rerank) scores each query–chunk pair together.
   - This is more accurate than vector similarity but slower, so it runs only on the shortlist.
   - Keep the top \~3–10 chunks.
4. **Build the prompt and generate**
   - Assemble a prompt with the instructions, the retrieved chunks (labeled with their sources), and the user question.
   - Instruct the LLM to answer only from the provided context, cite sources, and say "not found" when the context lacks the answer.
   - The LLM generates the answer with citations back to the documents or pages.
5. **Post-process (optional)**
   - Validate citations, check that the answer is grounded in the retrieved chunks, and format the output.

## Around the pipeline

- **Evaluation:** measure retrieval (Recall@k, MRR) and answers (faithfulness, correctness) separately so you know which stage is failing.
- **Updates:** when documents change, re-chunk and re-embed only those documents. If you switch embedding models, re-embed everything.
- **Debugging rule:** a wrong answer is caused by either retrieval (the right chunk wasn't retrieved) or generation (it was retrieved but misused). Check retrieval first.
