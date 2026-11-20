# Technical Environment: ReportTrace

## Project Technical Summary

- **Type:** new project (greenfield), built by four students in about three weeks of Construction.
- **Shape:** a local web app with two parts. A Streamlit page (the selection panel and the chatbot) calls a local FastAPI backend, which runs the LLM agent, its tools, and the answer check. The agent is built with LangChain. A separate command-line script builds the report library.
- **Runs on:** the team's laptops only (Windows 11 and macOS). Nothing is deployed to the cloud.
- **Leaves the laptop:** only calls to the LLM API and, while the library is being built, downloads from the SEC.

## Language and Package Manager

- **Python 3.12** (`requires-python = ">=3.12,<3.13"`). No other programming language; there is no JavaScript front end.
- **uv** for everything: installing Python (`uv python install 3.12`), dependencies, and running commands. Do not use pip, poetry, or conda.
- `pyproject.toml` holds all project and tool configuration. `uv.lock` is committed to Git. Use the latest stable release of each library when the project starts; `uv.lock` pins the exact versions.
- `src/` layout with one package, `reporttrace`.
- The code must run on Windows and macOS: use `pathlib` for paths, and write no shell scripts (`.sh`, `.bat`). Every command is a Python entry point started with `uv run`.

| Command                                                     | What it does                                        |
| ----------------------------------------------------------- | --------------------------------------------------- |
| `uv sync`                                                 | Installs Python and all dependencies                |
| `uv run reporttrace-ingest --tickers AAPL MSFT --years 3` | Downloads the filings and builds the report library |
| `uv run reporttrace`                                      | Starts the backend and the web page                 |
| `uv run pytest`                                           | Runs the automated tests                            |
| `uv run ruff check .` and `uv run ruff format .`        | Lints and formats the code                          |
| `uv run mypy src`                                         | Checks types                                        |
| `uv run reporttrace-eval`                                 | Runs the evaluation set against the real model      |

## Frameworks and Libraries

| Purpose                | Use                                                                       | Notes                                                                                                                                                                                                            |
| ---------------------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agent                  | **LangChain 1.x** (`langchain>=1.5,<2`)                           | `create_agent`, tools defined with `@tool`, and middleware. The team has used LangChain before                                                                                                               |
| LLM and embeddings     | **langchain-openai** (`>=1.8,<2`)                                 | `ChatOpenAI` and `OpenAIEmbeddings`, pointed at the Cornell API through `base_url`                                                                                                                         |
| Vector search          | **langchain-chroma** (`>=1.1,<2`)                                 | `Chroma` with a local `persist_directory`. One collection, `passages`, with the metadata `ticker`, `fiscal_year`, `section`, `evidence_id`. Every search filters on `ticker` and `fiscal_year` |
| Backend API            | **FastAPI** with **Pydantic v2**, served by **uvicorn** | Endpoints are plain `def` functions (not `async`); FastAPI runs them in its thread pool                                                                                                                      |
| Web page               | **Streamlit**                                                       | Selection in the sidebar, chat in the main area with `st.chat_message`, evidence cards in `st.expander`                                                                                                      |
| Other stored data      | **sqlite3** from the standard library                               | Companies, filings, passages, and financial figures in one file                                                                                                                                                  |
| HTTP calls             | **httpx**                                                           | For SEC downloads and for the web page calling the backend                                                                                                                                                       |
| Reading 10-K web pages | **beautifulsoup4** with **lxml**                              |                                                                                                                                                                                                                  |
| Settings               | **pydantic-settings**                                               | Reads the `.env` file                                                                                                                                                                                          |
| Logging                | `logging` from the standard library                                     | Run logs are written as one JSON object per line                                                                                                                                                                 |

## LLM Access

- **Model:** OpenAI GPT-4o, through the API that Cornell provides to students. **Embeddings:** `text-embedding-3-small` through the same API.
- **Settings** come from environment variables, never from code: `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `LLM_MODEL`, `EMBEDDING_MODEL`.
- **How the model is called:** `ChatOpenAI(model=..., base_url=..., api_key=..., temperature=0, timeout=60, max_retries=2)`. Tool calling goes through Chat Completions. Do not use the Assistants API, the Responses API, or any provider-side file search, code interpreter, or vector store.
- **One place creates the models.** `agent/models.py` builds the chat model and the embedding model from the settings. All other code receives them as parameters typed as LangChain's `BaseChatModel` and `Embeddings`, so tests can pass fakes. No other module imports `langchain_openai`.
- **Limits:** at most 10 tool calls per question, enforced by `ToolCallLimitMiddleware(run_limit=10)`. Each model call has a 60-second timeout and 2 retries.
- **Embeddings through a compatible API:** if the Cornell API rejects embedding requests, set `check_embedding_ctx_length=False` on `OpenAIEmbeddings` so that plain text is sent.
- **Fallback:** if the Cornell API has no embedding model, embeddings are computed on the laptop with Chroma's built-in model. If the Cornell API is unavailable, the same code can point at a local Ollama server through its OpenAI-compatible endpoint by changing the environment variables, as long as the model supports tool calling.
- **No hosted tracing.** LangSmith tracing is never switched on.

## Cloud Provider and Deployment Model

- **Cloud provider:** none. The product runs locally and is not hosted online.
- **How it runs:** `uv run reporttrace` starts two local processes and stops both on Ctrl+C.

| Process                   | Address                   | Role                                                                            |
| ------------------------- | ------------------------- | ------------------------------------------------------------------------------- |
| FastAPI backend (uvicorn) | `http://127.0.0.1:8000` | Runs the agent, the tools, and the answer check                                 |
| Streamlit web page        | `http://127.0.0.1:8501` | The selection panel, the chatbot, and the evidence cards; opened in the browser |

- **Data:** everything is stored in a local `data/` folder that Git ignores. Each team member builds it once with the ingestion script.

| Path                    | Content                                                                                      |
| ----------------------- | -------------------------------------------------------------------------------------------- |
| `data/raw/{ticker}/`  | Downloaded 10-K web pages and financial data files, kept so that nothing is downloaded twice |
| `data/reporttrace.db` | SQLite file with companies, filings, passages, and financial figures                         |
| `data/chroma/`        | Chroma vector index of the passages                                                          |
| `data/logs/`          | Run logs                                                                                     |

- No Docker, no cloud services, and no continuous deployment. The checks in the Testing section are run on a laptop before each commit.

## Architecture and Patterns

Application Design decides the modules. These rules are fixed:

- **Layers go one way:** `ui` → `api` → `agent` → `tools` → `storage`. The web page never imports agent code; it only calls the backend over HTTP.
- **There is one agent, created with LangChain's `create_agent`.** It gets the chat model, the four tools (`search_report`, `read_section`, `find_financial_data`, `calculate`), a system prompt, and middleware. Do not write a custom LangGraph graph, and do not add a second agent or sub-agents.
- **Retrieval happens only through tools.** There is no chain that retrieves once and then answers. The agent decides what to search and whether to search again.
- **Tools are read-only.** They read from SQLite and Chroma and never write. Only the ingestion script writes to the library.
- **The selection is enforced by code.** The user's selection and the data stores are passed to the agent as runtime context (`context_schema=AgentContext`, and `context=...` on each call). Tools read them from `runtime.context` through a `ToolRuntime` parameter, which is hidden from the model. The model can choose a year only from the selection and can never choose the company.
- **Each tool has two parts:** a plain function that holds the logic and is unit-tested directly, and a thin `@tool` wrapper that takes the model's arguments, reads the context, and calls the plain function.
- **Tool errors go back to the model.** A `wrap_tool_call` middleware turns a tool error into a tool message, so the model can correct itself.
- **Agent steps are streamed** with `agent.stream(..., stream_mode="updates")`. Each update becomes a `step` event for the web page and a line in the run log.
- **The answer check runs after the agent and before the response is sent.** It is ordinary code and makes no model call.
- **Conversation memory**, if follow-up questions are to use earlier turns (an open question in `vision.md`), uses LangGraph's `InMemorySaver` with one `thread_id` per conversation. Chat history is not written to a database.

## Testing Requirement

- **pytest** with **pytest-cov**. Line coverage must be at least 80% for `tools`, `agent`, `storage`, and `ingestion`. The Streamlit page is excluded from coverage.
- **hypothesis** for property-based tests of the `calculate` tool and the answer check.
- **ruff** for linting and formatting (line length 100), and **mypy** for type checking. Every function has type hints.
- **Tests never use the network.** The chat model is replaced by `FakeToolCallingModel`, a scripted fake: a subclass of LangChain's `GenericFakeChatModel` whose `bind_tools` returns the model itself. Embeddings are replaced by LangChain's `DeterministicFakeEmbedding`, and SEC downloads by `httpx.MockTransport`. Each test builds its own small library in pytest's `tmp_path`.

| Test type           | What it covers                                                                               | Tools                                                                      |
| ------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Unit                | The plain function of each tool, the answer check, passage splitting, figure selection       | pytest, hypothesis                                                         |
| Integration         | Backend endpoints from request to checked answer, with the fake model and the sample report  | FastAPI `TestClient`                                                     |
| Web page smoke test | The page loads and the selection panel lists the companies                                   | Streamlit `AppTest`                                                      |
| Evaluation          | At least 30 questions against the real model, reporting the success metrics in `vision.md` | `uv run reporttrace-eval`, run by hand and not part of `uv run pytest` |

Before each commit, `uv run ruff check .`, `uv run mypy src`, and `uv run pytest` must pass.

## Do NOT Use

| Prohibited                                                                                                                                                                                                                             | Reason                                                                                                                                          | Use instead                                                                              |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| LlamaIndex, CrewAI, AutoGen, Haystack, OpenAI Agents SDK                                                                                                                                                                               | One agent framework for the whole project, and the team already knows LangChain                                                                 | LangChain `create_agent`                                                               |
| Old LangChain APIs: `AgentExecutor`, `initialize_agent`, `create_react_agent`, `LLMChain`, `RetrievalQA`, `ConversationalRetrievalChain`, `langchain_community.vectorstores.Chroma`, anything from `langchain_classic` | Replaced in LangChain 1.x. Old tutorials still show them, and mixing them with 1.x code causes errors                                           | `create_agent`, `@tool`, `ToolRuntime`, `langchain_openai`, `langchain_chroma` |
| Custom LangGraph graphs (`StateGraph`), several agents, sub-agents, Deep Agents                                                                                                                                                      | One agent with four tools is enough; more structure adds work without helping the user                                                          | One agent from `create_agent`                                                          |
| Chains that retrieve once and then answer                                                                                                                                                                                              | The product depends on the agent choosing what to search and searching again                                                                    | Retrieval only through the `search_report` and `read_section` tools                  |
| Direct calls with the `openai` SDK                                                                                                                                                                                                   | A second way to call the model would bypass the one place where models are created                                                              | `ChatOpenAI` and `OpenAIEmbeddings` from `langchain-openai`                        |
| OpenAI Assistants API, Responses API, provider-side file search, code interpreter, hosted vector stores                                                                                                                                | May not be available through the Cornell API, and they take retrieval out of our control, so we could not enforce the selection or check quotes | Tools that we define with `@tool`                                                      |
| `eval()`, `exec()`, `compile()`                                                                                                                                                                                                  | The `calculate` tool receives input written by the model, so this would allow arbitrary code to run                                           | A fixed list of operations using `decimal.Decimal`                                     |
| Pinecone, Weaviate, Qdrant, FAISS, pgvector                                                                                                                                                                                            | The product runs locally with one vector store. Hosted stores need accounts and send data out; FAISS has no filter by company and year          | Chroma through `langchain-chroma`, stored locally                                      |
| sentence-transformers, transformers, torch (as direct dependencies)                                                                                                                                                                    | Downloads of several gigabytes and slow installs on laptops                                                                                     | Embeddings through the API; fallback is Chroma's built-in model                          |
| PyPDF2, pypdf, pdfplumber, PyMuPDF, unstructured, LangChain document loaders for PDF                                                                                                                                                   | The MVP has no PDF input; reports come from EDGAR as web pages                                                                                  | beautifulsoup4 with lxml                                                                 |
| sec-api, edgartools, sec-edgar-downloader                                                                                                                                                                                              | A paid key or a large dependency, when we need only a few public SEC addresses                                                                   | Direct calls to EDGAR with httpx                                                         |
| requests, aiohttp, urllib                                                                                                                                                                                                              | One HTTP client for the whole project; httpx has timeouts, streaming, and a mock transport for tests                                            | httpx                                                                                    |
| Flask, Django                                                                                                                                                                                                                          | One backend framework                                                                                                                           | FastAPI                                                                                  |
| React, Vue, other JavaScript front ends, Gradio, Chainlit                                                                                                                                                                              | The team works in Python only, with one user interface framework                                                                                | Streamlit                                                                                |
| SQLAlchemy and other ORMs; PostgreSQL, MySQL                                                                                                                                                                                           | Four small tables that are only read while the app runs do not need an ORM or a database server                                                 | `sqlite3` with parameterised queries                                                   |
| pandas                                                                                                                                                                                                                                 | Not needed to look up single figures                                                                                                            | Plain Python with `decimal.Decimal`                                                    |
| pickle                                                                                                                                                                                                                                 | Unsafe to load and not readable                                                                                                                 | JSON or SQLite                                                                           |
| pip, poetry, conda                                                                                                                                                                                                                     | The project uses uv only                                                                                                                        | uv                                                                                       |
| black, flake8, isort                                                                                                                                                                                                                   | Replaced by ruff                                                                                                                                | ruff                                                                                     |
| unittest test classes, Selenium, Playwright                                                                                                                                                                                            | One test style; browser automation is too heavy for this project                                                                                | pytest functions; Streamlit `AppTest`                                                  |
| Docker, AWS, Azure, Google Cloud, LangSmith and other hosted logging or tracing services                                                                                                                                               | The product runs locally and no data should leave the laptop except to the model API and the SEC                                                | Local processes and local log files                                                      |

## Security

- **No login.** The product has one local user and no accounts. Both processes bind to `127.0.0.1` only, never to `0.0.0.0`, and CORS is not enabled.
- **Secrets.** The API key and the SEC contact string are stored in a `.env` file that Git ignores. `.env.example` is committed with the variable names and no values. The key is held as a Pydantic `SecretStr`, and it is never logged, never returned by an endpoint, and never sent to the web page.
- **Input validation.** Every request is validated with Pydantic: the question has 1 to 1,000 characters, the ticker must be in the library, `fiscal_years` has 1 to 3 values that exist for that company, and an evidence ID must match its pattern.
- **Tool arguments are untrusted.** LangChain checks the types of the arguments that the model writes, and each tool checks its own rules, for example that the year is one of the selected years. A tool error is returned to the model as a tool message and is never shown to the user as a crash.
- **Prompt injection.** Report text and user questions are data, not instructions. Passages reach the model only as tool results, and the system prompt tells the model to ignore instructions inside them. Because tools are read-only and the answer check runs afterwards, injected text cannot change stored data or put an unchecked quote in front of the user.
- **No dynamic code.** `calculate` supports a fixed list of operations and never evaluates text as code.
- **Database queries** use parameters only, never string formatting. The backend opens SQLite in read-only mode.
- **Data sent out.** Only the user's question, passages from public reports, and public financial figures are sent to the model API. The product collects no personal data. LangSmith tracing is never switched on, and usage reporting is switched off in Streamlit (`gatherUsageStats = false`) and, if the installed version has it, in Chroma.
- **Run logs** stay in `data/logs/`, are ignored by Git, and contain no keys.
- **Dependencies.** `uv.lock` is committed so that every laptop installs the same versions.

`.env.example`:

```text
OPENAI_API_KEY=
OPENAI_BASE_URL=
LLM_MODEL=gpt-4o
EMBEDDING_MODEL=text-embedding-3-small
SEC_USER_AGENT=
```

## Example Code Patterns

The examples show the patterns to follow. Modules they import, such as `reporttrace.domain`, do not exist yet; they show the naming style to use.

### An endpoint

Endpoints are thin: they validate the input, call one function from a lower layer, and return a Pydantic model. Errors are raised as `ApiError`, and one handler turns them into the same shape for every endpoint: `{"error": {"code": "EVIDENCE_NOT_FOUND", "message": "..."}}`.

```python
# src/reporttrace/api/routers/evidence.py
from typing import Annotated

from fastapi import APIRouter, Depends, Path

from reporttrace.api.dependencies import get_passage_store
from reporttrace.api.errors import ApiError
from reporttrace.api.schemas import EvidenceResponse
from reporttrace.storage.passages import PassageStore

router = APIRouter(prefix="/api", tags=["evidence"])

EVIDENCE_ID_PATTERN = r"^[A-Z]{1,5}-FY\d{4}-[A-Z0-9]{4,8}-\d{4}$"


@router.get("/evidence/{evidence_id}", response_model=EvidenceResponse)
def get_evidence(
    evidence_id: Annotated[str, Path(pattern=EVIDENCE_ID_PATTERN)],
    store: Annotated[PassageStore, Depends(get_passage_store)],
) -> EvidenceResponse:
    """Return one stored passage so that the web page can show it on an evidence card."""
    passage = store.get(evidence_id)
    if passage is None:
        raise ApiError(
            status_code=404,
            code="EVIDENCE_NOT_FOUND",
            message=f"No passage has the id {evidence_id}.",
        )
    return EvidenceResponse(
        evidence_id=passage.evidence_id,
        ticker=passage.ticker,
        fiscal_year=passage.fiscal_year,
        section=passage.section,
        text=passage.text,
        filing_url=passage.filing_url,
    )
```

### A function (an agent tool)

The plain function `find_passages` holds the logic. The `@tool` wrapper `search_report` is what the model calls: the model supplies `query` and `fiscal_year`, and the company comes from the user's selection in `runtime.context`, which the model cannot see or change. The docstring of the wrapper is the description that the model reads.

```python
# src/reporttrace/tools/search_report.py
from langchain.tools import ToolRuntime, tool

from reporttrace.agent.context import AgentContext
from reporttrace.domain import Passage, Selection
from reporttrace.storage.vectors import PassageIndex
from reporttrace.tools.errors import ToolError
from reporttrace.tools.formatting import format_passages


def find_passages(
    query: str, fiscal_year: int, selection: Selection, index: PassageIndex, top_k: int = 5
) -> list[Passage]:
    """Find passages by meaning in one selected fiscal year of the selected company."""
    if fiscal_year not in selection.fiscal_years:
        raise ToolError(
            code="YEAR_NOT_SELECTED",
            message=f"FY{fiscal_year} is not one of the selected years.",
        )
    return index.search(
        query=query, ticker=selection.ticker, fiscal_year=fiscal_year, top_k=top_k
    )


@tool
def search_report(query: str, fiscal_year: int, runtime: ToolRuntime[AgentContext]) -> str:
    """Search one fiscal year of the selected company's annual report by meaning.

    Use this to find what the company says about a topic. Call it once for each
    fiscal year that you need.

    Args:
        query: What to look for, in plain English.
        fiscal_year: One of the fiscal years that the user selected.
    """
    context = runtime.context
    passages = find_passages(query, fiscal_year, context.selection, context.index)
    return format_passages(passages)
```

### A test

A unit test calls the plain function of a tool with fakes and checks one behaviour. Test names say what must be true.

```python
# tests/unit/tools/test_search_report.py
import pytest

from reporttrace.domain import Selection
from reporttrace.tools.errors import ToolError
from reporttrace.tools.search_report import find_passages
from tests.fakes.fake_index import FakePassageIndex


def test_search_returns_passages_from_the_selected_year_only() -> None:
    index = FakePassageIndex.with_sample_passages()
    selection = Selection(ticker="AAPL", fiscal_years=[2024, 2025])

    passages = find_passages("customer concentration", 2025, selection, index)

    assert passages
    assert all(p.ticker == "AAPL" and p.fiscal_year == 2025 for p in passages)


def test_search_rejects_a_year_that_the_user_did_not_select() -> None:
    index = FakePassageIndex.with_sample_passages()
    selection = Selection(ticker="AAPL", fiscal_years=[2025])

    with pytest.raises(ToolError) as error:
        find_passages("customer concentration", 2023, selection, index)

    assert error.value.code == "YEAR_NOT_SELECTED"
```
