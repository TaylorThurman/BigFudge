# Expense Tracker — Test Plan

**Version:** v2
**Date:** 2026-03-12
**Previous Version:** v1 — 2026-03-12
**Status:** Draft
**Author(s):** Claude (with user input)
**Architecture Blueprint:** expense-tracker-architecture-blueprint-v2
**Implementation Spec:** expense-tracker-implementation-spec-v2

---

## 1. Testing Overview

The expense tracker has two interfaces — a CLI (Click) and a web dashboard (FastAPI) — sharing a common service and repository layer backed by SQLite. The testing strategy adds a third test layer (API tests) to the existing unit and integration tests.

The highest-risk areas remain: data integrity, summary calculations, and filter logic. For the web dashboard, the additional risk areas are: API error handling (correct HTTP status codes), request validation (Pydantic schemas), and data consistency between CLI and web interfaces.

All testing is automated. Manual validation is optional but recommended for the dashboard's visual layout.

## 2. Test Layers

### Unit Tests

**Purpose:** Verify individual components in isolation — repository CRUD operations, service validation and business logic, and output formatting.
**Scope:** `repository.py`, `service.py`, `formatter.py`
**Location:** `tests/test_repository.py`, `tests/test_service.py`, `tests/test_formatter.py`
**Runs When:** Every commit.
**Expected Runtime:** < 5 seconds

**What to Test:**
- Repository: all CRUD methods, schema initialization, default category seeding, transaction behavior, filter construction
- Service: input validation (bad dates, negative amounts, unknown categories, nonexistent IDs), date defaulting, summary aggregation, CSV generation
- Formatter: table alignment, amount formatting, note truncation, empty list handling, summary rendering

**What NOT to Test at This Layer:**
- Click argument parsing or FastAPI route wiring

### Integration Tests (CLI)

**Purpose:** Verify end-to-end CLI behavior from command invocation through to output.
**Scope:** All CLI commands
**Location:** `tests/test_cli.py`
**Runs When:** Every commit, after unit tests pass.
**Expected Runtime:** < 10 seconds

**What to Test:**
- Each command with valid input produces correct output and exit code 0
- Each command with invalid input produces an error message and exit code 1
- Multi-step workflows, delete confirmation, default behaviors

### API Tests (new)

**Purpose:** Verify the FastAPI JSON API endpoints return correct responses, status codes, and handle errors properly.
**Scope:** All `/api/` endpoints — expenses CRUD, summary, categories
**Location:** `tests/test_api.py`
**Runs When:** Every commit, after unit tests pass.
**Expected Runtime:** < 10 seconds

**What to Test:**
- Each endpoint with valid input returns correct JSON and status code
- `POST /api/expenses` with invalid data returns 400 or 422 with error detail
- `PUT /api/expenses/999` returns 404
- `DELETE /api/expenses/999` returns 404
- `GET /api/summary` returns correct totals and category breakdown
- `GET /api/categories` returns all categories
- `POST /api/categories` with duplicate returns 400
- Filter query parameters work correctly on `GET /api/expenses`

**What NOT to Test at This Layer:**
- HTML rendering and template correctness — manual validation
- JavaScript behavior in templates — out of scope for automated tests
- Service-layer logic already covered by unit tests

## 3. Test Tooling & Frameworks

| Tool | Version | Purpose |
|------|---------|---------|
| pytest | >= 7.0 | Test runner, fixtures, assertions |
| click.testing.CliRunner | (bundled with Click) | CLI integration testing |
| httpx | >= 0.24.0 | FastAPI TestClient backend |
| FastAPI TestClient | (bundled with FastAPI) | API endpoint testing |
| tmp_path (pytest builtin) | — | Temp directories for isolated test databases |

**pytest configuration:**

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
```

No mocking libraries needed. All tests use real SQLite databases in temp directories.

## 4. Test Data & Fixtures

### Shared Fixtures (`tests/conftest.py`)

Existing fixtures (unchanged):

```python
@pytest.fixture
def tmp_db_path(tmp_path):
    return str(tmp_path / "test.db")

@pytest.fixture
def repo(tmp_db_path):
    r = Repository(db_path=tmp_db_path)
    yield r
    r.close()

@pytest.fixture
def expense_service(repo):
    return ExpenseService(repo)

@pytest.fixture
def category_service(repo):
    return CategoryService(repo)
```

New fixture for API tests:

```python
@pytest.fixture
def api_client(tmp_path, monkeypatch):
    db_path = str(tmp_path / "test.db")
    monkeypatch.setenv("EXPENSE_TRACKER_DB", db_path)
    from expense_tracker.web.app import create_app
    app = create_app(db_path=db_path)
    from fastapi.testclient import TestClient
    return TestClient(app)
```

### Sample Data

Same as v1. Tests insert data explicitly at the top of each test. No shared fixture data.

## 5. Acceptance Criteria & Thresholds

All v1 criteria remain. New criteria for dashboard requirements:

| Requirement | Criterion | How Measured |
|-------------|-----------|--------------|
| FR-015: Dashboard list | API returns expenses as JSON; filters work | API test |
| FR-016: Dashboard add | POST creates expense, returns 201 | API test |
| FR-017: Dashboard edit | PUT updates fields, returns updated expense | API test |
| FR-018: Dashboard delete | DELETE removes expense, returns 204 | API test |
| FR-019: Dashboard summary | GET /api/summary returns correct totals | API test |
| FR-020: Category management | GET/POST categories via API | API test |
| FR-021: Dashboard API | All API endpoints return JSON, reuse service layer | API test |
| FR-022: Dashboard command | `expense dashboard` starts server | Manual smoke test |
| NFR-003: API errors | 400 for bad input, 404 for not found, JSON body | API test |
| NFR-005: Responsiveness | Layout usable at 1024px | Manual check |

## 6. Validation Suite (Automated Gates)

One gate: **all tests pass**.

```
pytest tests/ -v
```

**Pass criteria:** 0 failures, 0 errors across all test files:
1. `test_repository.py` — unit tests
2. `test_service.py` — unit tests
3. `test_formatter.py` — unit tests
4. `test_cli.py` — CLI integration tests
5. `test_api.py` — API endpoint tests

**On failure:** Fix before proceeding.

## 7. Manual Validation

Optional but recommended for the dashboard:

1. `expense dashboard` — verify server starts and URL is printed
2. Open the URL in a browser — verify the expense list page loads
3. Add an expense via the form — verify it appears in the table
4. Edit an expense — verify the form is pre-populated and changes are saved
5. Delete an expense — verify confirmation and removal
6. Apply date and category filters — verify the table updates
7. Check summary sidebar — verify totals are correct
8. Run `expense list` in a separate terminal — verify CLI sees the same data
9. Resize browser to 1024px width — verify layout remains usable

## 8. Test Commands Reference

| What | Command | Expected Runtime |
|------|---------|-----------------|
| All tests | `pytest` | < 20s |
| Unit tests only | `pytest tests/test_repository.py tests/test_service.py tests/test_formatter.py` | < 5s |
| CLI integration tests | `pytest tests/test_cli.py` | < 10s |
| API tests | `pytest tests/test_api.py` | < 10s |
| Single test file | `pytest tests/test_api.py -v` | < 5s |
| With coverage | `pytest --cov=expense_tracker --cov-report=term-missing` | < 20s |

## 9. Coverage Expectations

**Target:** 90%+ line coverage across `repository.py`, `service.py`, `formatter.py`, `web/api.py`, and `web/schemas.py`.

**Coverage for `cli.py`:** Covered indirectly by CLI integration tests.

**Coverage for `web/views.py`:** Partially covered by API tests for data flow. HTML rendering validated manually.

**Exempt from coverage:**
- `__init__.py` files
- `models.py` — dataclass definitions
- `web/templates/` — HTML templates (not Python code)

## 10. Regression Strategy

Same as v1. The full test suite is the regression suite. When a bug is found: write a reproducing test, fix, verify.

Changes to `repository.py`, `service.py`, or `models.py` require the full suite to pass since both CLI and API depend on them.

## 11. Open Questions

_None at this time._

## 12. Changelog

### v2 — 2026-03-12

#### Added
- API Tests layer (Section 2) for FastAPI endpoint testing
- `api_client` fixture using FastAPI TestClient
- httpx as a test tool
- Acceptance criteria for FR-015 through FR-022 and NFR-003/NFR-005
- Manual validation checklist for dashboard visual review
- `test_api.py` in test commands reference

#### Changed
- Testing Overview updated to describe dual-interface testing approach
- Validation suite now includes `test_api.py`
- Coverage expectations expanded to include `web/api.py` and `web/schemas.py`
- Total expected runtime increased from 15s to 20s
- Regression strategy notes that changes to shared modules affect both CLI and API

#### Clarified
- HTML template rendering is validated manually, not via automated tests
