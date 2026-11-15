## Insurellm Expert Assistant

A retrieval-augmented assistant that answers questions about a company knowledge base: the business, products, employees, and contracts. Answers are grounded in retrieved source documents, and each chunk sent to the model includes the file it came from.

The corpus is a fictional insurance-tech company (Insurellm), stored as markdown.

### Techniques

- **LLM chunking.** A model splits each document into overlapping chunks sized for specific questions, and returns a headline, a short summary, and the original text unchanged. The stored text is headline + summary + original, so both the wording of the source and a query-friendly description are embedded.
- **Embeddings and store.** Chunks are embedded with either OpenAI `text-embedding-3-large`  or HuggingFace embedding models and stored in Chroma, with source path and document type as metadata. Ingestion runs documents in parallel (3 workers) and retries with exponential backoff on rate limits.
- **Query rewriting.** Before search, the model turns the current question plus chat history into one short, specific query, so follow-ups like "what does she cover?" resolve to the person already discussed.
- **Dual retrieval.** The original question and the rewritten query are each searched for the top 20 chunks. The two result lists are merged, dropping duplicates.
- **LLM re-ranking.** A model reorders the merged chunks by relevance to the original question and keeps the top 10. Generation uses Groq-hosted `openai/gpt-oss-120b`, with the instruction to be accurate, complete, and limited to the question.

## How the assistant is evaluated

The RAG assistant is scored in two separate layers. Retrieval is checked first: did the right source text come back, and how high was it ranked? The generated answer is checked second: is it correct, complete, and limited to the question? Splitting the two shows whether a bad answer came from the search or from the model.

The test set is 150 questions in `evaluation/tests.jsonl`. Each row has the question, keywords that must appear in the retrieved text, a reference answer, and a category. Categories are there so a single average cannot hide a weak type of question.

| Category | What it stresses |
| --- | --- |
| Direct fact | One fact in one place: a title, a price, a contract number |
| Numerical | Amounts, counts, and percentages |
| Temporal | Join dates, contract dates, and launch quarters |
| Comparative | A change or comparison, such as "reduced deployment time by 60%" |
| Relationship | A link between two records, such as which product a person owns |
| Spanning | Facts that only make sense together, often across people, products, and contracts |
| Holistic | A question that needs several documents at once, such as a total or a ranking across the portfolio |

### Retrieval metrics

These score the chunks returned for the question. They do not look at the written answer. Matching is case-insensitive, and each keyword is scored on its own, then averaged for that question.

**Mean Reciprocal Rank (MRR), 0 to 1.** For each keyword, find the first retrieved chunk that contains it. Rank 1 scores 1.0, rank 2 scores 0.5, rank 3 scores 0.33, and a miss scores 0. A high MRR means the supporting text is near the top of the list, where the model is most likely to use it. Finding the keyword only at rank 10 still counts as found, but it barely moves MRR.

**nDCG, 0 to 1.** Normalized Discounted Cumulative Gain with binary relevance: a chunk is relevant if it contains the keyword. Gain is discounted by rank, so a hit at the top is worth more than the same hit further down. The score is that discounted gain divided by the ideal ranking for the same chunks. nDCG asks whether the useful chunks are ordered well. MRR only cares where the first hit sits.

**Keyword coverage, 0 to 100%.** The share of expected keywords that appear anywhere in the retrieved set. Coverage can be high while MRR is low: the facts were retrieved, but they were buried. Coverage can also be low while one keyword ranks first: the top hit is good, but other required facts never came back.

On the dashboard, MRR and nDCG are green at 0.9 or above and amber from 0.75. Coverage is green at 90% and amber from 75%. Results are also averaged by category, so a failure on spanning or holistic questions shows up on its own.

### Answer metrics

A separate judge model scores the generated answer against the reference answer. The judge returns structured scores from 1 to 5, plus short feedback. The rubric is strict on purpose: a 5 is reserved for a perfect answer.

**Accuracy.** Does the answer match the facts in the reference? Any wrong fact scores 1. An acceptable answer scores 3. A 5 means the facts are right.

**Completeness.** Does the answer include everything the reference answer includes? Missing a required detail lowers the score. A 5 is only given when all of the reference information is present.

**Relevance.** Does the answer address this question and stop there? Extra material lowers the score even when it is true. A 5 means the answer is on the question and adds nothing else.

Accuracy, completeness, and relevance are green at 4.5 or above and amber from 4.0. The dashboard charts average accuracy by category.

Retrieval and answer scores are read together. High MRR with low accuracy means the right context was retrieved and the model still answered badly. Low coverage with a fluent answer means the model was writing without the supporting documents.

### How it runs

1. **Ingest** (`pro_implementation/ingest.py`). Load every markdown file, ask the model for structured chunks, embed them, and rebuild the Chroma collection.
2. **Answer** (`pro_implementation/answer.py`). Rewrite the question from the conversation, retrieve on both queries, merge, re-rank, and generate the answer from the top chunks with their sources.

### Stack

Python, Chroma, OpenAI/HuggingFace embeddings, LiteLLM/Langchain, Groq, Pydantic.
