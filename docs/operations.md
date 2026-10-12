# Operations guide

Setup, commands, and runtime options for the Space Mission Research Assistant.

## System capabilities

### Embedding and data pipeline

- `chunk_size` and `chunk_overlap` are configurable at runtime.
- Every chunk is checked against the configured token limit.
- Consecutive chunks use consistent overlap.
- OpenAI embeddings are generated for every stored chunk.
- Every chunk carries metadata including mission, source, and filepath.
- Existing documents support `skip`, `update`, and `replace` update modes.
- ChromaDB persistence directory and collection name are configurable.
- `--stats-only` reports the collection name, stored chunk/document count,
  source-file count, and mission/type aggregates.

### Retrieval and LLM integration

- Questions are embedded and queried against the persistent ChromaDB
  collection.
- Retrieval `top_k` is configurable, with optional mission metadata filtering.
- Results are ranked, deduplicated, and formatted with clear source
  attribution.
- The NASA expert prompt requires citations, reliance on retrieved evidence,
  and explicit uncertainty when the evidence is insufficient.
- Conversation history is maintained as role/content turns while retrieved
  context is supplied only for the current request.

### Evaluation

- Interactive evaluation measures response relevancy and faithfulness.
- Reference-based batch evaluation also measures context precision, context
  recall, factual correctness, and derived retrieval F1.
- The batch workflow loads the supplied test set, evaluates each question, and
  writes both per-question results and aggregate means.
- Malformed inputs and individual metric failures return structured errors
  instead of terminating the whole evaluation run.

## Requirements and installation

- Python 3.12
- [`uv`](https://docs.astral.sh/uv/)
- An OpenAI-compatible API endpoint and API key

Install the locked environment:

```bash
uv sync
```

`uv sync` creates or updates `.venv` and installs the exact dependency set
recorded in `uv.lock`, including the development test dependencies.

Create a local `.env` file:

```dotenv
OPENAI_API_KEY=your_api_key
OPENAI_BASE_URL=your_compatible_endpoint
```

`OPENAI_BASE_URL` is optional when using the standard OpenAI endpoint. For the
Streamlit app, the following optional environment variables override the
default generation and judge models:

```dotenv
OPENAI_CHAT_MODEL=gpt-5.4-nano
OPENAI_EVALUATOR_MODEL=gpt-5.4-mini
```

The batch workflow instead uses the explicit `--generator-model` and
`--evaluator-model` CLI flags shown below.

Do not commit `.env`; it is ignored by Git.

## 1. Test the implementation

```bash
uv run python -m pytest -q
```

This runs the complete test suite inside the locked project environment.

## 2. Build or update the vector collection

```bash
uv run python embedding_pipeline.py \
  --data-path data_text \
  --chroma-dir ./chroma_db_openai \
  --collection-name nasa_space_missions_text \
  --embedding-model text-embedding-3-small \
  --chunk-size 500 \
  --chunk-overlap 100 \
  --batch-size 50 \
  --update-mode replace
```

The important runtime options are:

| Option | Meaning |
|---|---|
| `--chunk-size` | Maximum number of tokens allowed in a chunk |
| `--chunk-overlap` | Target number of repeated tokens between neighboring chunks |
| `--batch-size` | Number of chunks sent in one embedding/storage batch |
| `--update-mode skip` | Embed only chunk IDs not already in the collection |
| `--update-mode update` | Add new IDs and overwrite matching existing IDs |
| `--update-mode replace` | Synchronize each source by upserting current chunks and deleting stale chunks |

Use `replace` after cleaning or chunking rules change, because the new chunk set
may no longer have the same IDs as the old set. Use `skip` for a safe,
incremental rerun when the source and chunking configuration have not changed.

To inspect an existing collection without rebuilding it:

```bash
uv run python embedding_pipeline.py \
  --chroma-dir ./chroma_db_openai \
  --collection-name nasa_space_missions_text \
  --stats-only
```

Initialization still validates the API configuration, but `--stats-only` does
not create new embeddings.

## 3. Run the chat application

```bash
uv run streamlit run chat.py
```

The Streamlit sidebar controls the ChromaDB backend, collection, retrieval
count, optional mission filter, generator model, and whether evaluation is
enabled. Generated answers are instructed to cite numbered context documents,
and the interface displays real-time metrics when evaluation is enabled. The
evaluator model is configured through `OPENAI_EVALUATOR_MODEL`; a separate
retrieved-source panel is not currently rendered.

## 4. Run the full batch evaluation

```bash
uv run python batch_evaluation.py \
  --dataset test_questions.json \
  --chroma-dir ./chroma_db_openai \
  --collection-name nasa_space_missions_text \
  --top-k 5 \
  --generator-model gpt-5.4-nano \
  --evaluator-model gpt-5.4-mini \
  --embedding-model text-embedding-3-small \
  --output evaluation_results.json
```

This is a live API workflow: it performs retrieval, answer generation, and
model-based evaluation for every test question, so it consumes API tokens and
can take several minutes.

See [the evaluation guide](evaluation.md) for metric definitions and the saved baseline.
