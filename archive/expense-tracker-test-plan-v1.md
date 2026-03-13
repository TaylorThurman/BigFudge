# Expense Tracker CLI — Test Plan

**Version:** v1
**Date:** 2026-03-12
**Previous Version:** N/A — initial draft
**Status:** Draft
**Author(s):** Claude (with user input)
**Architecture Blueprint:** expense-tracker-architecture-blueprint-v1
**Implementation Spec:** expense-tracker-implementation-spec-v1

---

## 1. Testing Overview

The expense tracker is a three-layer CLI application (CLI → Service → Repository) with no external services, no network calls, and no concurrency. This makes it straightforward to test: every layer can be tested in isolation against a real SQLite database in a temp directory, and the full CLI can be exercised end-to-end via Click's `CliRunner`.

The highest-risk areas are: data integrity (writes must be transactional), summary calculations (aggregation math must be correct), and filter logic (date range and category filters must compose correctly). These areas receive the most test coverage.

All testing is automated. There is no manual validation step — the system is a CLI tool with deterministic output, so every behavior can be verified programmatically.

## 2. Test Layers

### Unit Tests

**Purpose:** Verify individual components in isolation — repository CRUD operations, service validation and business logic, and output formatting.
**Scope:** `repository.py`, `service.py`, `formatter.py`
**Location:** `tests/test_repository.py`, `tests/test_service.py`, `tests/test_formatter.py`
**Runs When:** Every commit. Developers run these continuously during development.
**Expected Runtime:** < 5 seconds

**What to Test:**
- Repository: all CRUD methods, schema initialization, default category seeding, transaction behavior, dynamic WHERE clause construction for filters
- Service: input validation (bad dates, negative amounts, unknown categories, nonexistent IDs), date defaulting, summary aggregation (total, per-category, daily average), CSV generation
- Formatter: table alignment, amount formatting (`$12.50`), note truncation, empty list handling, summary rendering with percentages

**What NOT to Test at This Layer:**
- Click argument parsing — that's integration testing territory
- CLI output formatting decisions — tested via integration tests that check full command output

### Integration Tests

**Purpose:** Verify end-to-end behavior from CLI command invocation through to output, testing the wiring between all three layers.
**Scope:** All CLI commands (`add`, `list`, `edit`, `delete`, `summary`, `export`, `category add`, `category list`)
**Location:** `tests/test_cli.py`
**Runs When:** Every commit, after unit tests pass.
**Expected Runtime:** < 10 seconds

**What to Test:**
- Each command with valid input produces correct output and exit code 0
- Each command with invalid input produces an actionable error message on stderr and exit code 1
- Multi-step workflows: add expenses → list → filter → summary → export
- Delete confirmation prompt behavior (accept and reject)
- Default behaviors (date defaults to today, summary defaults to current month)

**What NOT to Test at This Layer:**
- Internal implementation details (which SQL was generated, how validation works internally)
- Performance benchmarks — this is a correctness layer

## 3. Test Tooling & Frameworks

| Tool | Version | Purpose |
|------|---------|---------|
| pytest | >= 7.0 | Test runner, fixtures, assertions |
| click.testing.CliRunner | (bundled with Click >= 8.0) | CLI integration testing |
| tmp_path (pytest builtin) | — | Temp directories for isolated test databases |
| csv (stdlib) | — | Verifying CSV export output |

**pytest configuration** — add to `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
```

No mocking libraries needed. All tests use real SQLite databases in temp directories. This is possible because SQLite is fast, in-process, and requires no setup.

## 4. Test Data & Fixtures

### Shared Fixtures (`tests/conftest.py`)

**`tmp_db_path`** — Returns a path to a fresh SQLite database file in a temp directory. Each test gets its own database, ensuring full isolation.

```python
@pytest.fixture
def tmp_db_path(tmp_path):
    return str(tmp_path / "test.db")
```

**`repo`** — A `Repository` instance connected to the temp database. Schema and default categories are initialized automatically.

```python
@pytest.fixture
def repo(tmp_db_path):
    r = Repository(db_path=tmp_db_path)
    yield r
    r.close()
```

**`expense_service` / `category_service`** — Service instances wired to the temp repository.

```python
@pytest.fixture
def expense_service(repo):
    return ExpenseService(repo)

@pytest.fixture
def category_service(repo):
    return CategoryService(repo)
```

**`cli_runner`** — A Click `CliRunner` for integration tests. The CLI commands need to be wired to use the temp database. This can be done by patching the default database path or by using a Click context object.

### Sample Data

Tests that need pre-populated data should insert expenses explicitly at the top of the test using `repo.insert_expense(...)`. Do not rely on shared fixture data across tests — each test builds its own state.

Standard test expenses for reuse across test functions:

| Date | Category | Amount | Note |
|------|----------|--------|------|
| 2026-03-01 | food | 12.50 | Lunch |
| 2026-03-01 | transport | 35.00 | Uber to airport |
| 2026-03-05 | food | 8.75 | Coffee |
| 2026-03-10 | entertainment | 15.00 | Movie |
| 2026-03-15 | housing | 1200.00 | Rent |

## 5. Acceptance Criteria & Thresholds

Mapped to product spec requirements:

| Requirement | Criterion | How Measured |
|-------------|-----------|--------------|
| FR-001: Add expense | Expense persists with correct fields; date defaults to today | Unit + integration test |
| FR-002: List expenses | Table output contains all expenses, sorted by date desc | Integration test |
| FR-003: Filter by date | Only expenses within inclusive date range returned | Unit test on service |
| FR-004: Filter by category | Only expenses matching category returned | Unit test on service |
| FR-005: Combined filters | Date + category filters compose correctly | Unit test on service |
| FR-006: Edit expense | Only specified fields change; others preserved | Unit + integration test |
| FR-007: Delete expense | Confirmation prompt; record removed after confirm | Integration test |
| FR-008: Summary | Total, per-category (sorted desc), daily average are correct | Unit test with manual calculation verification |
| FR-009: Default period | Summary with no args uses current calendar month | Unit test |
| FR-010: Default categories | 9 categories exist on fresh database | Unit test on repository |
| FR-011: Add category | Custom category persists and is usable | Unit + integration test |
| FR-012: List categories | All categories returned, sorted | Integration test |
| FR-013: CSV export | Valid CSV with correct headers and filtered content | Unit test + file inspection |
| FR-014: SQLite storage | Data persists across repository instances | Unit test: close and reopen |
| NFR-002: Data integrity | Interrupted writes don't corrupt database | Unit test: verify transaction usage |
| NFR-003: Error messages | Every error case produces actionable message | Integration test per error path |

## 6. Validation Suite (Automated Gates)

There is one validation gate: **all tests pass**.

```
pytest tests/ -v
```

**Pass criteria:** 0 failures, 0 errors. All tests must pass for the build to be considered valid.

**What runs:**
1. All unit tests (`test_repository.py`, `test_service.py`, `test_formatter.py`)
2. All integration tests (`test_cli.py`)

**On failure:** Fix the failing test before proceeding. No skipping or xfail marking without documented justification.

There are no staged gates, nightly runs, or separate validation environments. The full suite runs in under 15 seconds and should be executed before every commit.

## 7. Manual Validation

No manual validation is required. All behavior is deterministic and testable programmatically.

Optional manual smoke test after completing all tickets:

1. `pip install -e .` in a clean virtualenv
2. `expense add --category food --amount 12.50` — verify confirmation
3. `expense list` — verify table output
4. `expense summary` — verify totals
5. `expense export --output test.csv` — open CSV in a spreadsheet

## 8. Test Commands Reference

| What | Command | Expected Runtime |
|------|---------|-----------------|
| All tests | `pytest` | < 15s |
| Unit tests only | `pytest tests/test_repository.py tests/test_service.py tests/test_formatter.py` | < 5s |
| Integration tests only | `pytest tests/test_cli.py` | < 10s |
| Single test file | `pytest tests/test_service.py -v` | < 3s |
| Single test class | `pytest tests/test_service.py::TestAddExpense -v` | < 1s |
| With coverage | `pytest --cov=expense_tracker --cov-report=term-missing` | < 15s |

## 9. Coverage Expectations

**Target:** 90%+ line coverage across `repository.py`, `service.py`, and `formatter.py`. These modules contain all business logic and data access.

**Coverage for `cli.py`:** Covered indirectly by integration tests. No explicit line coverage target — the integration tests in `test_cli.py` exercise every command and error path.

**Exempt from coverage:**
- `__init__.py` — typically empty or minimal
- `models.py` — dataclass definitions with no logic

**Coverage tool:** `pytest-cov` (add to dev dependencies if coverage reporting is desired). Not required for v1 but recommended.

## 10. Regression Strategy

Given the small scope of this project, the full test suite serves as the regression suite. There is no separate regression layer.

**When a bug is found:**
1. Write a test that reproduces the bug (fails before the fix).
2. Fix the bug.
3. Verify the test passes.
4. The test remains in the suite permanently, preventing regression.

**Cross-component regression:** Changes to `models.py` (shared data structures) or `repository.py` (data layer) affect all layers. The full test suite must pass after any change to these files. Since the suite runs in under 15 seconds, this is practical for every commit.

## 11. Open Questions

_None at this time._

## 12. Changelog

### v1 — Initial Draft
- Initial test plan created from architecture blueprint v1 and implementation spec v1.
- Defined two test layers (unit, integration) with no manual validation required.
- All 14 functional requirements and 3 testable non-functional requirements mapped to test criteria.
- Full suite expected runtime: < 15 seconds.
