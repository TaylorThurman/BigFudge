# Expense Tracker CLI — Implementation Specification

**Version:** v1
**Date:** 2026-03-12
**Previous Version:** N/A — initial draft
**Status:** Draft
**Author(s):** Claude (with user input)
**Architecture Blueprint:** expense-tracker-architecture-blueprint-v1

---

## 1. Project Overview

A Python CLI expense tracker using Click for the interface and SQLite (stdlib `sqlite3`) for persistence. The codebase follows a three-layer architecture: CLI → Service → Repository. The single third-party runtime dependency is Click. All business logic lives in the service layer, independently testable without Click or database mocks.

## 2. Repo & Code Structure

```
expense-tracker/
├── pyproject.toml              <- Package metadata, dependencies, entry point
├── README.md                   <- Usage instructions
├── CLAUDE.md                   <- Developer context (Section 6)
├── src/
│   └── expense_tracker/
│       ├── __init__.py
│       ├── cli.py              <- Click command group and subcommands
│       ├── service.py          <- ExpenseService, CategoryService
│       ├── repository.py       <- Repository class (all SQL)
│       ├── formatter.py        <- Table and CSV output formatting
│       └── models.py           <- Dataclasses: Expense, Category, Summary
└── tests/
    ├── __init__.py
    ├── conftest.py             <- Shared fixtures (temp DB, test repo, services)
    ├── test_repository.py      <- Unit tests for Repository
    ├── test_service.py         <- Unit tests for service logic
    ├── test_formatter.py       <- Unit tests for output formatting
    └── test_cli.py             <- Integration tests via CliRunner
```

**Import boundaries:** `cli.py` imports from `service` and `formatter`. `service.py` imports from `repository` and `models`. `repository.py` imports only `sqlite3` and `models`. `formatter.py` imports only `models`. No circular imports. No layer may import upward.

## 3. Module Interfaces & Contracts

### models.py

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

### repository.py

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

All write methods run inside explicit `BEGIN`/`COMMIT` blocks. `update_expense` accepts only the fields being changed via `**fields`; unrecognized keys raise `ValueError`.

### service.py

```python
class ExpenseService:
    def __init__(self, repo: Repository): ...

    def add_expense(self, category: str, amount: float, date: str = None, note: str = None) -> Expense:
        """Validates category exists, amount > 0, date format. Defaults date to today."""

    def list_expenses(self, start_date: str = None, end_date: str = None, category: str = None) -> list[Expense]: ...
    def edit_expense(self, expense_id: int, **fields) -> Expense:
        """Validates expense exists and any changed fields. Returns updated expense."""

    def delete_expense(self, expense_id: int) -> Expense:
        """Validates expense exists. Returns the expense before deletion."""

    def get_summary(self, start_date: str = None, end_date: str = None) -> Summary:
        """Defaults to current calendar month if no range given."""

    def export_csv(self, path: str, start_date: str = None, end_date: str = None, category: str = None) -> int:
        """Writes CSV. Returns count of exported rows."""

class CategoryService:
    def __init__(self, repo: Repository): ...

    def list_categories(self) -> list[str]: ...
    def add_category(self, name: str) -> str:
        """Lowercase, strip, reject duplicates. Returns the normalized name."""
```

Services raise `ValueError` for validation failures and `KeyError` for not-found entities. The CLI layer catches these and calls `click.echo` + `sys.exit(1)`.

### formatter.py

```python
def format_expense_table(expenses: list[Expense]) -> str:
    """Returns a formatted table string with columns: ID, Date, Category, Amount, Note."""

def format_summary(summary: Summary) -> str:
    """Returns formatted summary with total, category breakdown, and daily average."""
```

## 4. Code Patterns & Conventions

**Naming:** Snake_case for files, functions, variables. PascalCase for classes. No abbreviations except `db` and `repo`.

**Error handling:** Services raise `ValueError` (bad input) or `KeyError` (not found). CLI catches and renders via `click.echo(f"Error: {e}", err=True)` then `sys.exit(1)`.

```python
# In cli.py
try:
    expense = expense_service.add_expense(category=category, amount=amount)
except (ValueError, KeyError) as e:
    click.echo(f"Error: {e}", err=True)
    raise SystemExit(1)
```

**Date handling:** All dates are `YYYY-MM-DD` strings. Validation uses `datetime.date.fromisoformat()`. No datetime objects cross module boundaries — strings only.

**Amount formatting:** Always display with two decimal places: `f"${amount:.2f}"`.

**Testing:** Every ticket includes tests that verify its acceptance criteria. Unit tests use a real SQLite database in a temp directory (no mocks). Integration tests use Click's `CliRunner`.

## 5. Dependencies & Environment Setup

**pyproject.toml:**
```toml
[project]
name = "expense-tracker"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = ["click>=8.0"]

[project.scripts]
expense = "expense_tracker.cli:cli"

[project.optional-dependencies]
dev = ["pytest>=7.0"]

[build-system]
requires = ["setuptools>=64"]
build-backend = "setuptools.backends._legacy:_Backend"
```

**Setup:**
```
python -m venv .venv
source .venv/bin/activate   # or .venv\Scripts\activate on Windows
pip install -e ".[dev]"
```

## 6. Developer Context (CLAUDE.md)

```markdown
# Expense Tracker CLI

Personal CLI expense tracker. Python + Click + SQLite.

## Key Rules
- Three-layer architecture: cli.py → service.py → repository.py. No skipping layers.
- All SQL lives in repository.py. No SQL anywhere else.
- Services raise ValueError/KeyError. CLI catches and renders errors.
- All dates are YYYY-MM-DD strings everywhere.
- All write operations must use transactions.

## Commands
- Install: `pip install -e ".[dev]"`
- Run tests: `pytest`
- Run CLI: `expense --help`

## Testing
- Unit tests: test against real SQLite in temp dirs, not mocks
- Integration tests: use click.testing.CliRunner
- Every change must include tests for the acceptance criteria it addresses

## Files rarely modified
- models.py (shared data structures — changes ripple everywhere)
```

## 7. Git Workflow

- **Branch naming:** `feat/<short-description>`, `fix/<short-description>`, `test/<short-description>`
- **Commits:** Imperative mood, concise. E.g., "Add expense summary command"
- **Main branch:** `main` — always passing tests
- **Feature branches:** One per ticket, merged via PR or direct merge after tests pass

## 8. Open Questions

_None at this time._

## 9. Changelog

### v1 — Initial Draft
- Initial implementation spec created from expense-tracker-architecture-blueprint-v1.
- Defined module interfaces for Repository, ExpenseService, CategoryService, and formatter.
- Established code conventions and project structure.
