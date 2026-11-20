# Component Methods: ReportTrace

This document gives the main types and method signatures of each component in `components.md`, with their purpose and their input and output types. The signatures are Python 3.12 with type hints. Private helpers are not listed.

**Business rules are not defined here.** Examples: how passages are split, how a quote is matched, how a figure value is chosen, how a number is rounded, and what each tool returns as text. These rules are defined in Functional Design for each unit. Field lists of the domain types are the minimum that the interfaces need; Functional Design may add fields.

---

## C1. Domain types (`reporttrace.domain`)

All are frozen dataclasses (Q2).

| Type | Main fields | Purpose |
| --- | --- | --- |
| `Selection` | `ticker: str`, `fiscal_years: list[int]` | The user's selection; `is_comparison` is true for two or three years |
| `Company` | `ticker: str`, `name: str`, `cik: str` | A company in the library |
| `FiscalYear` | `ticker: str`, `fiscal_year: int`, `label: str`, `period_end: date` | An available year, for example "FY2025 (ended Jan 31, 2025)" (FR-1.7) |
| `Filing` | `accession_number: str`, `ticker: str`, `fiscal_year: int`, `period_end: date`, `filed_on: date`, `filing_url: str` | One 10-K filing |
| `Passage` | `evidence_id: str`, `ticker: str`, `fiscal_year: int`, `section: str`, `position: int`, `text: str`, `filing_url: str` | One stored passage; `position` orders the passages in a section |
| `Figure` | `figure_id: str`, `ticker: str`, `line_item: str`, `filed_label: str`, `fiscal_year: int`, `period_start: date or None`, `period_end: date`, `value: Decimal`, `unit: str`, `source_fiscal_year: int`, `filing_url: str` | One filed value; `source_fiscal_year` names the report that filed it (FR-6.2) |
| `Calculation` | `calculation_id: str`, `operation: str`, `operands: list[Decimal]`, `operand_sources: list[str]`, `result: Decimal`, `formula: str` | One result of `calculate` |
| `StepKind` | enum: `SEARCH`, `READ_SECTION`, `FIGURE_LOOKUP`, `CALCULATION`, `TOOL_ERROR`, `LIMIT_REACHED`, `SECOND_ATTEMPT` | The kind of a step |
| `Step` | `index: int`, `kind: StepKind`, `description: str`, `fiscal_year: int or None`, `elapsed_ms: int` | One step shown to the user (FR-4.1) |
| `QuestionRequest` | `question: str`, `selection: Selection`, `conversation_id: str or None` | One question to answer |
| `AnswerSentence` | `text: str`, `citations: list[str]`, `topic: str` | One sentence of the Answer part |
| `FigureRow` | `label: str`, `fiscal_year: int`, `display_value: str`, `unit: str`, `source_id: str`, `formula: str or None`, `quoted: bool`, `based_on_quoted: bool` | One row of the Figures part (FR-6.5, FR-6.6) |
| `QuoteItem` | `text: str`, `evidence_id: str`, `fiscal_year: int`, `topic: str` | One quote with its citation |
| `ComparisonRow` | `aspect: str`, `status: str`, `cells: dict[int, list[AnswerSentence]]` | One row of a side-by-side comparison (FR-5.2) |
| `GapItem` | `text: str` | One line under "Not found in the reports" |
| `CheckedAnswer` | `sentences`, `figures`, `quotes`, `comparison`, `gaps`, `notes: list[str]`, `scope: str`, `suggestions: list[str]` | The answer that is shown; holds no removed text |
| `CheckFailure` | `part: str`, `item_index: int`, `reason: str`, `text: str`, `topic: str` | One item that failed the check |
| `CheckResult` | `failures: list[CheckFailure]` | `passed` is true when there are no failures |
| `RemovedItem` | `part: str`, `text: str`, `reason: str`, `attempt: int` | One removed item, for the run log only (FR-8.7) |
| `StepEvent` | `conversation_id: str`, `step: Step` | Sent as soon as a step happens (Q7) |
| `AnswerEvent` | `conversation_id: str`, `selection: Selection`, `answer: CheckedAnswer`, `steps: list[Step]`, `limit_reached: bool` | Sent once, at the end |
| `ErrorEvent` | `conversation_id: str or None`, `code: str`, `message: str`, `steps: list[Step]` | Sent instead of an answer when the run fails (NFR-5.2) |
| `QuestionEvent` | `StepEvent` or `AnswerEvent` or `ErrorEvent` | The union streamed by the question service |
| `RunRecord` | The fields of FR-10.1 | One line of the run log |

---

## C2. Settings (`reporttrace.settings`)

```python
class Settings(BaseSettings):
    openai_api_key: SecretStr
    openai_base_url: str
    llm_model: str = "gpt-4o"
    embedding_model: str = "text-embedding-3-small"
    embedding_backend: Literal["api", "local"] = "api"   # "local" = Chroma's built-in model (NFR-7.3)
    embedding_check_ctx_length: bool = True              # False for the compatible-API fallback (NFR-7.3)
    sec_user_agent: str
    data_dir: Path = Path("data")
    backend_host: str = "127.0.0.1"
    backend_port: int = 8000
    page_port: int = 8501

def get_settings() -> Settings:
    """Return the settings read from the environment and the .env file."""
```

---

## C3. Storage (`reporttrace.storage`)

```python
def open_library(path: Path, *, read_only: bool) -> sqlite3.Connection:
    """Open the library file; read_only=True for the backend (NFR-3.7)."""

class CompanyStore:
    def __init__(self, connection: sqlite3.Connection) -> None: ...
    def list_companies(self) -> list[Company]: ...
    def get_company(self, ticker: str) -> Company | None: ...
    def list_fiscal_years(self, ticker: str) -> list[FiscalYear]: ...
    def get_filing(self, ticker: str, fiscal_year: int) -> Filing | None: ...

class PassageStore:
    def __init__(self, connection: sqlite3.Connection) -> None: ...
    def get(self, evidence_id: str) -> Passage | None: ...
    def neighbours(self, evidence_id: str) -> tuple[Passage | None, Passage | None]:
        """Return the passages just before and just after, in the same section (FR-7.3)."""
    def list_sections(self, ticker: str, fiscal_year: int) -> list[str]: ...
    def list_section(self, ticker: str, fiscal_year: int, section: str) -> list[Passage]: ...

class FigureStore:
    def __init__(self, connection: sqlite3.Connection) -> None: ...
    def find(self, ticker: str, line_item: str, source_fiscal_years: list[int]) -> list[Figure]:
        """Return every filed value of a line item reported in the given reports."""
    def get(self, figure_id: str) -> Figure | None: ...
    def list_line_items(self) -> list[str]: ...

class PassageIndex:
    def __init__(self, persist_directory: Path, embeddings: Embeddings) -> None: ...
    def search(self, query: str, ticker: str, fiscal_year: int, top_k: int) -> list[Passage]:
        """Search the passages collection by meaning, filtered on ticker and fiscal_year."""

class LibraryInfoStore:
    def __init__(self, connection: sqlite3.Connection) -> None: ...
    def embedding_model(self) -> str | None:
        """Return the embedding model that built the index (D10)."""

# storage/writers.py: imported by the ingestion script only
class LibraryWriter:
    def __init__(self, connection: sqlite3.Connection) -> None: ...
    def create_schema(self) -> None: ...
    def upsert_company(self, company: Company) -> None: ...
    def upsert_filing(self, filing: Filing, label: str) -> None: ...
    def replace_passages(self, filing: Filing, passages: list[Passage]) -> None: ...
    def replace_figures(self, ticker: str, figures: list[Figure]) -> None: ...
    def set_embedding_model(self, name: str) -> None: ...

class IndexWriter:
    def __init__(self, persist_directory: Path, embeddings: Embeddings) -> None: ...
    def replace_filing(self, filing: Filing, passages: list[Passage]) -> None:
        """Embed each passage once and store it with its metadata (FR-1.5)."""
```

---

## C4. Ingestion (`reporttrace.ingestion`)

```python
class EdgarClient:
    def __init__(self, http: httpx.Client, user_agent: str, raw_dir: Path,
                 max_requests_per_second: float = 10.0) -> None: ...
    def get_cik(self, ticker: str) -> str: ...
    def list_annual_reports(self, cik: str, count: int) -> list[Filing]: ...
    def download_report(self, filing: Filing) -> Path:
        """Return the cached 10-K web page; download it only if it is not in data/raw (FR-1.4)."""
    def download_company_facts(self, cik: str) -> Path: ...

@dataclass(frozen=True)
class ParsedSection:
    label: str            # for example "Item 7" or "Item 8, Note 12" or "Other"
    paragraphs: list[str]

class TenKParser:
    def parse(self, html: str) -> list[ParsedSection]:
        """Split a 10-K web page into items, notes in Item 8, and Other (FR-1.6)."""

class PassageSplitter:
    def split(self, filing: Filing, sections: list[ParsedSection]) -> list[Passage]:
        """Split sections into passages and give each an evidence ID (FR-1.5)."""

def load_line_item_map(path: Path) -> LineItemMap: ...

class FigureMapper:
    def __init__(self, line_items: LineItemMap) -> None: ...
    def map(self, company_facts: dict[str, object], filings: list[Filing]) -> FigureMapping:
        """Return the figures of the fixed line items and the line items that are missing (FR-1.8)."""

@dataclass(frozen=True)
class CompanyReport:
    ticker: str
    reports_ingested: list[str]
    sections_recognised: dict[str, list[str]]
    other_text_share: dict[str, float]
    missing_line_items: list[str]
    error: str | None

@dataclass(frozen=True)
class IngestionReport:
    companies: list[CompanyReport]

class IngestionService:
    def __init__(self, edgar: EdgarClient, parser: TenKParser, splitter: PassageSplitter,
                 mapper: FigureMapper, writer: LibraryWriter, index_writer: IndexWriter,
                 embedding_model: str) -> None: ...
    def ingest(self, tickers: list[str], years: int) -> IngestionReport:
        """Build the library for each ticker; continue after a failure (FR-1.9, D7)."""

def main(argv: list[str] | None = None) -> int:
    """Entry point of reporttrace-ingest."""
```

---

## C5. Tools (`reporttrace.tools`)

Each tool has a plain function, which holds the logic and is unit-tested, and a `@tool` wrapper, which the model calls. The wrapper's docstring is the description that the model reads. `runtime` is hidden from the model.

```python
class ToolError(Exception):
    def __init__(self, code: str, message: str) -> None: ...

# tools/search_report.py
def find_passages(query: str, fiscal_year: int, selection: Selection,
                  index: PassageIndex, top_k: int = 5) -> list[Passage]: ...
@tool
def search_report(query: str, fiscal_year: int, runtime: ToolRuntime[AgentContext]) -> str: ...

# tools/read_section.py
def read_section_passages(section: str, fiscal_year: int, selection: Selection,
                          store: PassageStore) -> list[Passage]: ...
@tool
def read_section(section: str, fiscal_year: int, runtime: ToolRuntime[AgentContext]) -> str: ...

# tools/find_financial_data.py
def lookup_figures(line_item: str, fiscal_year: int, selection: Selection,
                   store: FigureStore) -> list[Figure]:
    """Return the value to use for one line item and year (FR-6.2, FR-6.3, D1)."""
@tool
def find_financial_data(line_item: str, fiscal_year: int,
                        runtime: ToolRuntime[AgentContext]) -> str: ...

# tools/calculate.py
def compute(operation: str, operands: list[Decimal], operand_sources: list[str]) -> Calculation:
    """Apply one operation from the fixed list; never evaluates text (FR-6.4, NFR-3.6)."""
@tool
def calculate(operation: str, operands: list[str], operand_sources: list[str],
              runtime: ToolRuntime[AgentContext]) -> str: ...

# tools/formatting.py
def format_passages(passages: list[Passage]) -> str: ...
def format_figures(figures: list[Figure]) -> str: ...
def format_calculation(calculation: Calculation) -> str: ...

ALL_TOOLS: list[BaseTool]   # the four tools, in a fixed order
```

Each wrapper adds what it returned to `runtime.context.evidence` before it returns (Q5).

---

## C6. Agent core (`reporttrace.agent`)

```python
# agent/models.py: the only module that imports langchain_openai
def build_chat_model(settings: Settings) -> BaseChatModel: ...
def build_embeddings(settings: Settings) -> Embeddings:
    """Return API embeddings, or Chroma's built-in model when settings say so (NFR-7.3)."""

# agent/context.py: a leaf module that tools may import
class EvidenceRecord:
    def add_passages(self, passages: list[Passage]) -> None: ...
    def add_figures(self, figures: list[Figure]) -> None: ...
    def add_calculation(self, calculation: Calculation) -> None: ...
    def passage(self, evidence_id: str) -> Passage | None: ...
    def figure(self, figure_id: str) -> Figure | None: ...
    def calculation(self, calculation_id: str) -> Calculation | None: ...
    def passages(self) -> list[Passage]: ...
    def figures(self) -> list[Figure]: ...
    def calculations(self) -> list[Calculation]: ...

@dataclass
class AgentContext:
    selection: Selection
    index: PassageIndex
    passages: PassageStore
    figures: FigureStore
    evidence: EvidenceRecord

# agent/prompts.py
SYSTEM_PROMPT: str
def build_retry_message(failures: list[CheckFailure]) -> str: ...

# agent/answer_schema.py: Pydantic, because it is the model's output schema (Q2, Q4)
class AgentAnswer(BaseModel):
    scope: Literal["answered", "outside_reports", "other_company"]
    answer: list[AgentSentence]
    figures: list[AgentFigure]
    quotes: list[AgentQuote]
    comparison: list[AgentComparisonRow]
    not_found: list[AgentGap]
    suggestions: list[str]

# agent/middleware.py
def build_middleware(tool_call_limit: int) -> list[AgentMiddleware]:
    """Tool-error middleware (FR-3.6) and ToolCallLimitMiddleware(run_limit=tool_call_limit)."""

# agent/factory.py
def build_agent(model: BaseChatModel, checkpointer: InMemorySaver,
                tool_call_limit: int) -> CompiledAgent:
    """create_agent with ALL_TOOLS, SYSTEM_PROMPT, build_middleware(tool_call_limit),
    context_schema=AgentContext, response_format=AgentAnswer, and the checkpointer."""

# agent/steps.py
def steps_from_update(update: dict[str, object], first_index: int, elapsed_ms: int) -> list[Step]:
    """Describe the tool calls and tool errors in one streamed update in plain words (FR-4.1)."""
def count_tool_calls(update: dict[str, object]) -> int: ...
def token_usage(update: dict[str, object]) -> TokenUsage: ...
```

`CompiledAgent` is a type alias for what `create_agent` returns.

---

## C7. Conversations (`reporttrace.agent.conversations`)

```python
@dataclass
class Conversation:
    conversation_id: str
    thread_id: str
    selection: Selection
    evidence: EvidenceRecord
    started_at: datetime

class ConversationRegistry:
    def resolve(self, conversation_id: str | None, selection: Selection) -> Conversation:
        """Return the conversation, or start a new one if the ID is missing or unknown,
        or if the selection differs (FR-2.5)."""
    def clear(self, conversation_id: str) -> None: ...
    def get(self, conversation_id: str) -> Conversation | None: ...
```

---

## C8. Answer check (`reporttrace.agent.checking`)

```python
class AnswerChecker:
    def check(self, answer: AgentAnswer, evidence: EvidenceRecord) -> CheckResult:
        """Check quotes (FR-8.2), numbers (FR-8.3), and citations (FR-8.4). No model call."""
    def build_checked_answer(self, answer: AgentAnswer, result: CheckResult,
                             limit_reached: bool) -> tuple[CheckedAnswer, list[RemovedItem]]:
        """Drop what failed, add a neutral gap line per removed statement (FR-8.7, D4),
        and return the removed items for the run log."""

def normalise_quote(text: str) -> str: ...
def number_matches(shown: str, exact: Decimal) -> bool: ...
```

The matching rules of `normalise_quote` and `number_matches` are defined in Functional Design.

---

## C9. Run log (`reporttrace.agent.run_log`)

```python
class RunLogWriter:
    def __init__(self, log_dir: Path) -> None: ...
    def write(self, record: RunRecord) -> None:
        """Append one JSON object as one line to the log file (FR-10.1)."""
```

---

## C10. Question service (`reporttrace.agent.service`)

```python
class QuestionService:
    def __init__(self, model: BaseChatModel, index: PassageIndex, passages: PassageStore,
                 figures: FigureStore, registry: ConversationRegistry, checker: AnswerChecker,
                 run_log: RunLogWriter, checkpointer: InMemorySaver,
                 tool_call_limit: int = 10) -> None: ...
    def ask(self, request: QuestionRequest) -> Iterator[QuestionEvent]:
        """Answer one question: yield StepEvents as they happen, then one AnswerEvent
        or one ErrorEvent."""
    def clear(self, conversation_id: str) -> None:
        """Forget a conversation and its memory (FR-3.4)."""
```

---

## C11. API (`reporttrace.api`)

```python
# api/app.py
def create_app() -> FastAPI:
    """Build the app; on startup, open the stores read-only and build one QuestionService."""

# api/errors.py
class ApiError(Exception):
    def __init__(self, status_code: int, code: str, message: str) -> None: ...
def api_error_handler(request: Request, exc: ApiError) -> JSONResponse: ...
def validation_error_handler(request: Request, exc: RequestValidationError) -> JSONResponse: ...

# api/dependencies.py
def get_company_store() -> CompanyStore: ...
def get_passage_store() -> PassageStore: ...
def get_figure_store() -> FigureStore: ...
def get_question_service() -> QuestionService: ...
```

| Endpoint | Function | Request | Response | Errors |
| --- | --- | --- | --- | --- |
| `GET /api/health` | `get_health()` | none | `HealthResponse` (library loaded, embedding model matches the index) | none |
| `GET /api/companies` | `list_companies(store)` | none | `CompaniesResponse`: companies with their fiscal years, labels, and end dates | none |
| `POST /api/questions` | `ask_question(body, service, store)` | `AskRequest`: `question` (1 to 1,000 characters), `ticker`, `fiscal_years` (1 to 3), `conversation_id` (optional) | A stream of `StepEventSchema` lines, then one `AnswerEventSchema` or `ErrorEventSchema` line (media type `application/x-ndjson`) | `TICKER_NOT_FOUND`, `YEAR_NOT_AVAILABLE`, `VALIDATION_ERROR` |
| `DELETE /api/conversations/{conversation_id}` | `clear_conversation(conversation_id, service)` | none | 204 | none (clearing an unknown ID does nothing) |
| `GET /api/evidence/{evidence_id}` | `get_evidence(evidence_id, store)` | `evidence_id` matching the pattern in `tech-env.md` | `EvidenceResponse` | `EVIDENCE_NOT_FOUND` |
| `GET /api/evidence/{evidence_id}/context` | `get_evidence_context(evidence_id, store)` | as above | `EvidenceContextResponse`: the passages before and after | `EVIDENCE_NOT_FOUND` |
| `GET /api/figures/{figure_id}` | `get_figure(figure_id, store)` | `figure_id` matching its pattern (defined in Functional Design) | `FigureResponse` | `FIGURE_NOT_FOUND` |

All error responses have the shape `{"error": {"code": "...", "message": "..."}}` (NFR-5.1). `api/schemas.py` holds every request, response, and event schema as Pydantic models.

---

## C12. Web page (`reporttrace.ui`)

```python
# ui/client.py
class BackendUnavailable(Exception): ...

class BackendClient:
    def __init__(self, base_url: str, http: httpx.Client) -> None: ...
    def health(self) -> HealthResponse: ...
    def list_companies(self) -> CompaniesResponse: ...
    def ask(self, question: str, ticker: str, fiscal_years: list[int],
            conversation_id: str | None) -> Iterator[QuestionEventSchema]:
        """Send the question and yield each event as its line arrives (Q7)."""
    def clear(self, conversation_id: str) -> None: ...
    def get_evidence(self, evidence_id: str) -> EvidenceResponse: ...
    def get_evidence_context(self, evidence_id: str) -> EvidenceContextResponse: ...
    def get_figure(self, figure_id: str) -> FigureResponse: ...

# ui/views.py
def render_selection_panel(companies: CompaniesResponse) -> Selection | None: ...
def render_selection_note(previous: Selection, current: Selection) -> None: ...
def render_steps(steps: list[StepSchema]) -> None: ...
def render_answer(answer: AnswerEventSchema, client: BackendClient) -> None: ...
def render_evidence_card(evidence: EvidenceResponse, quoted: str, client: BackendClient) -> None: ...
def render_figure_card(figure: FigureResponse) -> None: ...

# ui/app.py
def main() -> None:
    """The Streamlit script: session state holds the chat, the selection, and the conversation ID."""
```

---

## C13. Evaluation (`reporttrace.evaluation`)

```python
@dataclass(frozen=True)
class EvalQuestion:
    question_id: str
    ticker: str
    fiscal_years: list[int]
    question: str
    kind: Literal["text", "figure", "comparison", "unanswerable"]
    expected_figures: list[ExpectedFigure]
    expected_sections: list[str]

@dataclass(frozen=True)
class EvalResult:
    question_id: str
    answer: CheckedAnswer | None
    error: ErrorEvent | None
    retrieved_sections: list[str]
    quotes_checked: int
    quotes_matching: int
    seconds: float

def load_question_set(path: Path) -> list[EvalQuestion]: ...

class EvaluationRunner:
    def __init__(self, service: QuestionService, passages: PassageStore) -> None: ...
    def run(self, questions: list[EvalQuestion]) -> list[EvalResult]: ...

# evaluation/metrics.py
def quote_accuracy(results: list[EvalResult]) -> float: ...
def figure_accuracy(results: list[EvalResult], questions: list[EvalQuestion]) -> float: ...
def retrieval_rate(results: list[EvalResult], questions: list[EvalQuestion]) -> float: ...
def honest_gap_rate(results: list[EvalResult], questions: list[EvalQuestion]) -> float: ...
def answer_quality(ratings: list[Rating]) -> QualityScore: ...
def median_seconds(results: list[EvalResult], questions: list[EvalQuestion]) -> dict[str, float]: ...

# evaluation/report.py
def write_results(results: list[EvalResult], path: Path) -> None: ...
def write_rating_sheet(results: list[EvalResult], questions: list[EvalQuestion], path: Path) -> None: ...
def read_ratings(path: Path) -> list[Rating]: ...
def write_report(metrics: MetricSummary, path: Path) -> None: ...

# evaluation/cli.py
def main(argv: list[str] | None = None) -> int:
    """Entry point of reporttrace-eval: 'run' runs the set; 'report' adds the ratings and writes the report."""
```

---

## C14. Launcher (`reporttrace.launcher`)

```python
def main() -> int:
    """Start uvicorn (reporttrace.api.app:create_app) and Streamlit (reporttrace/ui/app.py)
    on 127.0.0.1 as two child processes, and stop both on Ctrl+C."""
```

---

## Entry points (`pyproject.toml`)

| Command | Function |
| --- | --- |
| `reporttrace` | `reporttrace.launcher:main` |
| `reporttrace-ingest` | `reporttrace.ingestion.cli:main` |
| `reporttrace-eval` | `reporttrace.evaluation.cli:main` |
