# FinSight: an LLM-powered personal finance assistant

FinSight answers natural-language questions about your spending ("How much
did I spend on dining last month?", "Am I over budget on groceries?") by
combining an LLM with **real, deterministic calculations**. The model never
guesses a number, it calls a Python function that queries the database.

It can also answer questions grounded in uploaded documents (bank
statements, tax forms) using retrieval-augmented generation (RAG).

## Why this project is more than "a chatbot wrapper"

LLMs are unreliable at arithmetic and prone to hallucinating plausible-looking
numbers. FinSight avoids that failure mode with a tool-calling architecture:
the LLM's job is to understand the question and pick the right tool; a plain
Python/SQL function does the actual math. Every number in a FinSight answer
is traceable back to a database query, not model generation.

## Architecture

1. **Data sources**: a CSV export or the Plaid sandbox API feed transactions in.
2. **Transaction DB (SQLite)**: everything lands here first.
3. From the database, two layers work in parallel:
   - **Tool layer**: deterministic Python/SQL functions for sums, budgets, and math.
   - **RAG layer (Chroma)**: search over uploaded statements and tax documents.
4. **LLM orchestrator (Gemini, tool calling)**: decides which layer to call and turns the result into a plain-language answer.
5. **Chat UI (Streamlit)**: where you actually type your question and read the answer.

## Project structure

- `finsight/`
  - `backend/`
    - `main.py`: FastAPI app (chat, upload, budgets, transactions endpoints)
    - `db.py`: SQLite schema and data access helpers
    - `tools.py`: deterministic finance calculations and tool schemas
    - `llm.py`: Gemini API wrapper with the tool-calling loop
    - `categorize.py`: batch LLM categorization of raw transactions
    - `rag.py`: document ingestion and retrieval (ChromaDB)
    - `generate_sample_data.py`: synthetic transaction generator (no bank needed)
  - `frontend/`
    - `app.py`: Streamlit chat UI
  - `data/`: SQLite DB and Chroma vector store live here (gitignored)
  - `requirements.txt`
  - `.env.example`


## Requirements

- **Python 3.10+** is recommended. The code uses modern type-hint syntax
  (`str | None`) guarded with `from __future__ import annotations`, which
  makes it compatible back to Python 3.9. If you hit unrelated syntax
  errors, upgrading your interpreter is the more durable fix.
- A free Gemini API key (see below), no credit card required.

## Setup

```bash
git clone <your-repo-url>
cd finsight
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 1. Get a free Gemini API key

1. Go to [aistudio.google.com](https://aistudio.google.com) and sign in with a Google account.
2. Click **Get API key** → **Create API key**. No credit card needed.
3. Copy the key.

### 2. Add your key

```bash
cp .env.example .env
```

Open `.env` and paste your key:

GEMINI_API_KEY=your-actual-key-here


`python-dotenv` loads this automatically at startup (see `main.py`,
`generate_sample_data.py`, `categorize.py`), so there's no manual `export`/`set`
needed, and it works the same in any terminal or editor.

### 3. Generate sample data

```bash
cd backend
python generate_sample_data.py
```

This creates ~120 days of realistic synthetic transactions (rent, groceries,
dining, etc.) in `data/finsight.db` so you can demo the project without
connecting a real bank account.

To test the LLM categorizer instead of pre-labeled sample data, open
`generate_sample_data.py`, set `"category": None` in `_make_row`, then run:

```bash
python categorize.py
```

### 4. Start the backend

From inside `backend/`:

```bash
uvicorn main:app --reload --port 8000
```

Leave this running. Check it's alive at `http://localhost:8000/docs`.

### 5. Start the frontend (in a **new** terminal)

```bash
cd frontend
streamlit run app.py
```

Activate your venv again in this new terminal first if it's not already
active. Opens at `http://localhost:8501`. Try asking:
- "How much did I spend on dining this month?"
- "Compare my transport spending this month vs last month"
- "What's my savings rate this month?"
- Set a budget in the sidebar, then ask "Am I over budget on groceries?"

### 6. (Optional) Upload documents for RAG

Use the sidebar file uploader to index a PDF or text statement, then ask
questions that reference it, and FinSight will search the document for relevant
context before answering.

## Swapping in real bank data

Replace `generate_sample_data.py` with a Plaid sandbox integration:
1. Sign up for a free Plaid developer account (sandbox mode requires no real bank).
2. Use `plaid-python` to call `/transactions/get` against a sandbox institution.
3. Map the response into the same `bulk_insert_transactions()` call used here.

## Troubleshooting

- **`ERROR: Could not import module "main"`**: you're running `uvicorn` from
  the wrong folder. `cd backend` first, then run the command again.
- **`TypeError: unsupported operand type(s) for |: 'type' and 'NoneType'`**:
  you're on Python 3.9 and a `from __future__ import annotations` line is
  missing from `tools.py` or `llm.py`. It should be the first line in both
  files. Upgrading to Python 3.11+ avoids this class of issue entirely.
- **`404 NOT_FOUND ... model ... no longer available`**: Google occasionally
  retires model names. Open `backend/llm.py` and `backend/categorize.py`,
  find the `MODEL = "..."` line, and swap in whatever current free-tier
  model name the error message suggests.
- **"Something went wrong while talking to the backend" in Streamlit**: the
  real error is in your `uvicorn` terminal, not the Streamlit one. Scroll up
  for a Python traceback and read the last line.
- **`chromadb` fails to install**: it has native dependencies; make sure
  you're on a supported Python version and have build tools available
  (on Windows, this usually means installing the "Desktop development with
  C++" workload via Visual Studio Build Tools).

## Notes on scope / what to extend for a stronger portfolio piece

- Add authentication (multiple users, not just one local DB).
- Move from SQLite to Postgres and containerize with Docker Compose.
- Add an eval script that checks categorization accuracy against a labeled
  test set, which is a strong thing to mention in interviews.
- Add spending forecasts (simple moving average or Prophet) for a "proactive
  insights" feature.
- Swap the Streamlit frontend for Next.js if you want full-stack React on
  your resume too.

## Disclaimer

This is an educational project. It is not financial advice, and the LLM is
explicitly instructed not to give definitive personalized financial,
investment, or tax guidance.
