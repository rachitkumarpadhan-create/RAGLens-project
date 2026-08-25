# RAGLens

> A small, self-contained pipeline that builds a RAG system over Wikipedia articles and then scores it — objectively, with [RAGAS] — instead of eyeballing whether the answers "look right."

---

## The Problem This Solves

It's easy to get a RAG demo working. It's much harder to know whether it's actually good.

A pipeline can pass a casual glance and still be broken in ways that only show up under scrutiny:
- The model states things that were never in the retrieved chunks (hallucination)
- The answer is technically grounded in context, but doesn't address the question that was asked
- The retriever returns chunks that are topically related but useless for the actual query
- The retriever fails to surface a chunk that was needed, so the model answers with partial information

None of these show up in a quick manual check. RAGLens exists to catch them automatically, with numbers you can track across changes to your pipeline.

---

## The 4 Metrics

| Metric | Question it answers |
|---|---|
| **Faithfulness** | Can every claim in the answer be traced back to the retrieved context? |
| **Answer Relevancy** | Does the answer actually address what was asked? |
| **Context Precision** | Of the chunks retrieved, how many were actually relevant? |
| **Context Recall** | Did retrieval surface everything needed to fully answer the question? |

Faithfulness and Answer Relevancy evaluate the **generation** step. Context Precision and Context Recall evaluate the **retrieval** step. Looking at all four together tells you *which half of the pipeline* is responsible when quality drops — that separation is the main reason to use a framework like RAGAS instead of a single end-to-end "correctness" score.

---

## How RAGAS Scoring Works

RAGAS is an LLM-as-a-judge framework:

- **No labeled training data needed** — it evaluates inputs and outputs, not the pipeline's internals
- **Architecture-agnostic** — works whether the RAG system is built with LangChain, LlamaIndex, or something custom
- **An LLM does the scoring** — this project uses whichever model `PROVIDER` currently points to (OpenAI `gpt-4o-mini` or HuggingFace `Mistral-7B-Instruct-v0.3`) to reason over the question, answer, retrieved context, and ground truth for each metric
- **Deterministic enough to compare runs** — the same dataset produces comparable scores, so you can measure whether a change (chunk size, k, embedding model) actually helped

---

## Pipeline

```
Wikipedia Articles (5 AI topics)
          │
          ▼
   [ ingest.py ]
   Chunk → Embed → ChromaDB
          │
          ▼
   [ rag_pipeline.py ]
   For each question in questions.json:
     Retrieve top-3 chunks → Generate answer → Save to rag_outputs.json
          │
          ▼
   [ evaluate.py ]
   Load rag_outputs.json → RAGAS scores each sample
     → results/ragas_results.csv
```

**Retrieval + generation**
- Knowledge base: 5 Wikipedia articles on AI topics (~952 chunks total)
- Chunking: `RecursiveCharacterTextSplitter`, 500 characters per chunk, 50 character overlap
- Embeddings: `text-embedding-3-small` (OpenAI) or `all-MiniLM-L6-v2` (HuggingFace, runs locally)
- Vector store: ChromaDB, persisted locally — no external infra
- Retrieval: top-3 nearest chunks per question
- Generation: `gpt-4o-mini` (OpenAI) or `Mistral-7B-Instruct-v0.3` (HuggingFace)

**Evaluation**
- Judge model: same provider-selected LLM as generation
- Input per sample: question, generated answer, retrieved contexts, ground truth
- Output: per-sample scores for all 4 metrics, averaged and written to CSV

---

## Tech Stack

| Component | Technology |
|---|---|
| Orchestration | LangChain (LCEL) |
| Vector store | ChromaDB (local) |
| Document source | Wikipedia API |
| Generation LLM | OpenAI `gpt-4o-mini` or HuggingFace `Mistral-7B-Instruct-v0.3` |
| Embeddings | OpenAI `text-embedding-3-small` or `sentence-transformers/all-MiniLM-L6-v2` |
| Evaluation | RAGAS |
| Output | JSON (raw RAG outputs) + CSV (scored results) |

---

## Project Structure

```
raglens/
├── data/questions.json        # 10 pre-generated Q&A pairs with ground truths
├── outputs/rag_outputs.json   # RAG pipeline outputs (created on run)
├── results/ragas_results.csv  # RAGAS evaluation scores (created on run)
├── chroma_db/                 # Local vector store (created on run)
├── config.py                  # Provider switch — OpenAI vs HuggingFace
├── ingest.py                  # Load Wikipedia → chunk → embed → ChromaDB
├── rag_pipeline.py            # Run retrieval + generation → save outputs
├── evaluate.py                # Run RAGAS scoring → save results
├── requirements.txt
└── .env.example
```

---

## Setup

**1. Install dependencies**
```bash
pip install -r requirements.txt
```

**2. Configure environment**
```bash
cp .env.example .env
# set PROVIDER and the matching API key
```

`.env` example:
```env
PROVIDER=openai
OPENAI_API_KEY=your_key_here
```

---

## Usage

Run in order — each step depends on the output of the one before it:

```bash
# 1. Build the knowledge base
python3 ingest.py

# 2. Run retrieval + generation over all questions
python3 rag_pipeline.py

# 3. Score the results with RAGAS
python3 evaluate.py
```

---

## Sample Output

```
==================================================
        RAGAS Evaluation Summary
==================================================
  Faithfulness:       0.9857
  Answer Relevancy:   0.9148
  Context Precision:  0.8917
  Context Recall:     0.7167
==================================================
```

Context Recall being the lowest of the four here is a useful, realistic signal: it points at the retriever occasionally missing needed information, rather than the generator making things up — which is exactly the kind of diagnosis this pipeline is meant to surface.

---

## Knowledge Base Topics

- Artificial Intelligence
- Machine Learning
- Deep Learning
- Natural Language Processing
- Large Language Model

---

## Provider Options

| `PROVIDER` value | Generation LLM | Embeddings |
|---|---|---|
| `openai` | `gpt-4o-mini` | `text-embedding-3-small` |
| `huggingface` | `Mistral-7B-Instruct-v0.3` (HF Inference API) | `all-MiniLM-L6-v2` (local) |

---

## Known Limitations / Next Steps

- The evaluation set is small (10 questions) and drawn from the same 5 articles used to build the index — good for a fast sanity check, not a rigorous benchmark.
- Retrieval is plain top-k cosine similarity; no reranking or hybrid (keyword + vector) search yet.
- The same model currently acts as both generator and judge, which risks self-favoring bias. A stronger, independent judge model would give more trustworthy scores.
- No automated tests yet on the chunking/formatting logic.

These are the first things to change if this pipeline needed to go from "internal eval script" to something a team would rely on.

---

## License

MIT
