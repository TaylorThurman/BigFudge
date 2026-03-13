# Expense Tracker — Implementation Specification

**Version:** v2
**Date:** 2026-03-12
**Previous Version:** v1 — 2026-03-12
**Status:** Draft
**Author(s):** Claude (with user input)
**Architecture Blueprint:** expense-tracker-architecture-blueprint-v2

---

## 1. Project Overview

A Python expense tracker with two interfaces: a CLI (Click) and a web dashboard (FastAPI), both backed by SQLite (stdlib `sqlite3`). The codebase follows a layered architecture where both interfaces share the same service and repository layers. Runtime dependencies: Click, FastAPI, Uvicorn, Jinja2. All business logic lives in the service layer, independently testable without CLI or web framework involvement.

## 2. Repo & Code Structure

```
expense-tracker/
├── pyproject.toml              <- Package metadata, dependencies, entry point
├── CLAUDE.md                   <- Developer context (Section 6)
├── src/
│   └── expense_tracker/
│       ├── __init__.py
│       ├── cli.py              <- Click command group and subcommands
│       ├── service.py          <- ExpenseService, CategoryService
│       ├── repository.py       <- Repository class (all SQL)
│       ├── formatter.py        <- Table and CSV output formatting (CLI only)
│       ├── models.py           <- Dataclasses: Expense, CategorySummary, Summary
│       └── web/
│           ├── __init__.py
│           ├── app.py          <- FastAPI app factory, lifespan, dependency injection
│           ├── api.py          <- JSON API route handlers (/api/*)
│           ├── views.py        <- HTML route handlers (/, /add, /edit/{id})
│           ├── schemas.py      <- Pydantic request/response models
│           └── templates/
│               ├── base.html   <- Shared layout (head, nav, footer)
│               ├── list.html   <- Expense table with filters + summary sidebar
│               ├── add.html    <- Add expense form
│               └── edit.html   <- Edit expense form (pre-populated)
└── tests/
    ├── __init__.py
    ├── conftest.py             <- Shared fixtures (temp DB, repo, services, test client)
    ├── test_repository.py      <- Unit tests for Repository
    ├── test_service.py         <- Unit tests for service logic
    ├── test_formatter.py       <- Unit tests for output formatting
    ├── test_cli.py             <- Integration tests via CliRunner
    └── test_api.py             <- API tests via FastAPI TestClient
```

**Import boundaries:** `cli.py` imports from `service` and `formatter`. `web/app.py` imports from `service` and `repository`. `web/api.py` and `web/views.py` import from `web/app` (for dependencies), `service`, and `models`. `web/schemas.py` imports nothing from the project. `service.py` imports from `repository` and `models`. `repository.py` imports only `sqlite3` and `models`. No circular imports. No layer may import upward.

## 3. Module Interfaces & Contracts

### models.py (unchanged)

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class Expense:
    id: int
    date: str            # YYYY-MM-DD
    category: str
    amount: float
    note: Optional[str]

@dataclass
class CategorySummary:
    category: str
    total: float

@dataclass
class Summary:
    total: float
    by_category: list[CategorySummary]  # sorted by total descending
    daily_average: float
    start_date: str
    end_date: str
    num_days: int
```

### repository.py (unchanged)

```python
class Repository:
    def __init__(self, db_path: str = None):
        """Defaults to ~/.expense-tracker/expenses.db. Creates dir and schema if needed."""

    def insert_expense(self, date: str, category: str, amount: float, note: str | None) -> Expense: ...
    def get_expense(self, expense_id: int) -> Expense | None: ...
    def update_expense(self, expense_id: int, **fields) -> Expense: ...
    def delete_expense(self, expense_id: int) -> bool: ...
    def list_expenses(self, start_date: str = None, end_date: str = None, category: str = None) -> list[Expense]: ...

    def get_category(self, name: str) -> str | None: ...
    def list_categories(self) -> list[str]: ...
    def add_category(self, name: str) -> str: ...

    def close(self) -> None: ...
```

### service.py (unchanged)

```python
class ExpenseService:
    def __init__(self, repo: Repository): ...
    def add_expense(self, category: str, amount: float, date: str = None, note: str = None) -> Expense: ...
    def list_expenses(self, start_date: str = None, end_date: str = None, category: str = None) -> list[Expense]: ...
    def edit_expense(self, expense_id: int, **fields) -> Expense: ...
    def delete_expense(self, expense_id: int) -> Expense: ...
    def get_summary(self, start_date: str = None, end_date: str = None) -> Summary: ...
    def export_csv(self, path: str, start_date: str = None, end_date: str = None, category: str = None) -> int: ...

class CategoryService:
    def __init__(self, repo: Repository): ...
    def list_categories(self) -> list[str]: ...
    def add_category(self, name: str) -> str: ...
```

### web/schemas.py (new)

```python
from pydantic import BaseModel, Field
from typing import Optional

class ExpenseCreate(BaseModel):
    category: str
    amount: float = Field(gt=0)
    date: Optional[str] = None       # YYYY-MM-DD, defaults to today
    note: Optional[str] = None

class ExpenseUpdate(BaseModel):
    category: Optional[str] = None
    amount: Optional[float] = Field(default=None, gt=0)
    date: Optional[str] = None
    note: Optional[str] = None

class ExpenseResponse(BaseModel):
    id: int
    date: str
    category: str
    amount: float
    note: Optional[str]

class CategoryCreate(BaseModel):
    name: str

class SummaryResponse(BaseModel):
    total: float
    by_category: list[dict]          # [{category: str, total: float}, ...]
    daily_average: float
    start_date: str
    end_date: str
    num_days: int

class ErrorResponse(BaseModel):
    detail: str
```

### web/app.py (new)

```python
from fastapi import FastAPI
from expense_tracker.repository import Repository
from expense_tracker.service import ExpenseService, CategoryService

def create_app(db_path: str = None) -> FastAPI:
    """Create and configure the FastAPI app with routes and dependencies."""
    ...

def get_repo() -> Repository:
    """FastAPI dependency that provides a Repository instance."""
    ...

def get_expense_service(repo: Repository = Depends(get_repo)) -> ExpenseService:
    """FastAPI dependency that provides an ExpenseService."""
    ...

def get_category_service(repo: Repository = Depends(get_repo)) -> CategoryService:
    """FastAPI dependency that provides a CategoryService."""
    ...
```

### web/api.py (new)

```python
from fastapi import APIRouter

router = APIRouter(prefix="/api")

# GET    /api/expenses          -> list expenses (query params: start_date, end_date, category)
# POST   /api/expenses          -> create expense (JSON body: ExpenseCreate)
# PUT    /api/expenses/{id}     -> update expense (JSON body: ExpenseUpdate)
# DELETE /api/expenses/{id}     -> delete expense
# GET    /api/summary           -> get summary (query params: start_date, end_date)
# GET    /api/categories        -> list categories
# POST   /api/categories        -> add category (JSON body: CategoryCreate)
```

Error handling: `KeyError` → 404 response, `ValueError` → 400 response. Use FastAPI exception handlers registered in `app.py`.

### web/views.py (new)

```python
from fastapi import APIRouter, Request
from fastapi.templating import Jinja2Templates

router = APIRouter()

# GET  /            -> render expense list page with filters and summary
# GET  /add         -> render add expense form
# GET  /edit/{id}   -> render edit expense form (pre-populated)
```

HTML routes render Jinja2 templates. Form submissions are handled by the API routes — HTML forms POST/PUT to `/api/` endpoints, and the view redirects after success.

## 4. Code Patterns & Conventions

**Naming:** Snake_case for files, functions, variables. PascalCase for classes. No abbreviations except `db` and `repo`.

**Error handling (CLI):** Services raise `ValueError` (bad input) or `KeyError` (not found). CLI catches and renders via `click.echo(f"Error: {e}", err=True)` then `sys.exit(1)`.

**Error handling (API):** Register FastAPI exception handlers for `ValueError` → `400` and `KeyError` → `404`. Return `ErrorResponse` JSON body.

```python
# In web/app.py
@app.exception_handler(KeyError)
async def key_error_handler(request, exc):
    return JSONResponse(status_code=404, content={"detail": str(exc)})

@app.exception_handler(ValueError)
async def value_error_handler(request, exc):
    return JSONResponse(status_code=400, content={"detail": str(exc)})
```

**Date handling:** All dates are `YYYY-MM-DD` strings. No datetime objects cross module boundaries.

**Amount formatting:** Display with two decimal places: `f"${amount:.2f}"`.

**FastAPI dependencies:** Use Depends() for repo and service injection. Each request gets its own repo instance; the repo is closed after the request completes (use a generator dependency or lifespan).

**Templates:** Jinja2 templates extend `base.html`. Template variables use the same naming as the dataclass fields.

**Testing:** Unit tests use real SQLite in temp dirs. CLI tests use CliRunner. API tests use FastAPI's `TestClient` with `httpx`. All tests use the `EXPENSE_TRACKER_DB` env var to point to a temp database.

## 5. Dependencies & Environment Setup

**pyproject.toml:**
```toml
[project]
name = "expense-tracker"
version = "0.2.0"
requires-python = ">=3.10"
dependencies = [
    "click>=8.0",
    "fastapi>=0.100.0",
    "uvicorn[standard]>=0.20.0",
    "jinja2>=3.0",
]

[project.scripts]
expense = "expense_tracker.cli:cli"

[project.optional-dependencies]
dev = ["pytest>=7.0", "httpx>=0.24.0"]

[build-system]
requires = ["setuptools>=64"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**Setup:**
```
pip install -e ".[dev]"
```

## 6. Developer Context (CLAUDE.md)

```markdown
# Expense Tracker

Personal expense tracker with CLI + web dashboard. Python + Click + FastAPI + SQLite.

## Key Rules
- Layered architecture: cli.py and web/ both call service.py → repository.py. No skipping layers.
- All SQL lives in repository.py. No SQL anywhere else.
- Services raise ValueError/KeyError. CLI catches and renders. API returns 400/404 JSON.
- All dates are YYYY-MM-DD strings everywhere.
- All write operations must use transactions.
- Web routes go in web/api.py (JSON) and web/views.py (HTML). App setup in web/app.py.

## Commands
- Install: `pip install -e ".[dev]"`
- Run tests: `pytest`
- Run CLI: `expense --help`
- Run dashboard: `expense dashboard`

## Testing
- Unit tests: test against real SQLite in temp dirs, not mocks
- CLI integration tests: use click.testing.CliRunner
- API tests: use FastAPI TestClient (httpx)
- Every change must include tests for the acceptance criteria it addresses

## Files rarely modified
- models.py (shared data structures — changes ripple everywhere)
- repository.py (stable SQL interface — changes affect both CLI and web)
```

## 7. Git Workflow

- **Branch naming:** `feat/<short-description>`, `fix/<short-description>`, `test/<short-description>`
- **Commits:** Imperative mood, concise. E.g., "Add expense API endpoints"
- **Main branch:** `main` — always passing tests
- **Feature branches:** One per ticket, merged after tests pass

## 8. Open Questions

_None at this time._

## 9. Changelog

### v2 — 2026-03-12

#### Added
- `web/` module with `app.py`, `api.py`, `views.py`, `schemas.py`, and `templates/`
- Pydantic schema interfaces (ExpenseCreate, ExpenseUpdate, ExpenseResponse, etc.)
- FastAPI app factory and dependency injection interfaces
- API and HTML route contracts
- API error handling pattern (exception handlers for ValueError/KeyError)
- New dependencies: FastAPI, Uvicorn, Jinja2, httpx
- `test_api.py` for API endpoint testing
- Updated CLAUDE.md with web-specific rules and commands

#### Changed
- Project version bumped to 0.2.0
- pyproject.toml updated with new dependencies
- CLAUDE.md expanded with web layer guidance
- Import boundary documentation updated to include web module

#### Clarified
- Repository and service interfaces remain unchanged — web layer consumes them as-is
