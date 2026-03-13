# Expense Tracker — Product Specification

**Version:** v2
**Date:** 2026-03-12
**Previous Version:** v1 — 2026-03-12
**Status:** Draft
**Author(s):** Claude (with user input)

---

## 1. Problem Statement & Vision

Tracking personal expenses is tedious when it requires opening a spreadsheet or app. Most off-the-shelf tools are bloated with features that get in the way of the core task: recording what you spent, seeing where your money goes, and understanding your spending patterns over time.

This project builds a lightweight expense tracker in Python that makes it fast to log expenses from the terminal, query them with filters, and get spending summaries — all backed by a local SQLite database that persists between sessions. The CLI should feel like a natural part of a developer's terminal workflow: quick to invoke, predictable in behavior, and easy to script.

In addition to the CLI, a web-based dashboard provides a visual interface for managing expenses through a browser. The dashboard is built with FastAPI and offers full CRUD capabilities — users can view, add, edit, and delete expenses, see summaries, and manage categories from a web UI. The CLI and web dashboard share the same database and service layer, ensuring data consistency regardless of which interface is used.

The end state is a dual-interface expense tracking system — CLI for power users and quick entry, web dashboard for visual review and management — backed by a single SQLite database.

## 2. Users & Stakeholders

### Primary User: Individual Developer / Power User

- **Role:** Single user tracking personal expenses.
- **Interaction via CLI:** Invokes the CLI multiple times per day to log expenses. Reviews summaries weekly or monthly. Occasionally exports data to CSV for use in spreadsheets or other tools.
- **Interaction via Web Dashboard:** Opens the dashboard in a browser to visually review expenses, add or edit entries through forms, view summary charts, and manage categories.
- **Needs:** Speed of entry (CLI commands under 5 seconds to type), visual overview (dashboard for browsing and filtering), reliable persistence, clear and scannable output, and the ability to correct mistakes (edit/delete) from either interface.
- **Frustrations:** Slow startup, overly verbose output, losing data, having to remember complex command syntax, needing to switch between tools to get a full picture.

## 3. Functional Requirements

### Expense Management (CLI)

**FR-001: Add an Expense**
The system shall allow the user to add an expense with a date, category, amount (USD), and optional note. The date shall default to today's date if not provided. The category must be selected from the predefined category list.
Acceptance: Running `expense add --category food --amount 12.50` creates a record with today's date, category "food", amount 12.50, and no note. Running `expense add --date 2026-03-01 --category transport --amount 35.00 --note "Uber to airport"` creates a record with all four fields populated.
Priority: Must

**FR-002: List Expenses**
The system shall display expenses in a formatted table. By default, it lists all expenses sorted by date (most recent first).
Acceptance: Running `expense list` outputs a table with columns for ID, date, category, amount, and note.
Priority: Must

**FR-003: Filter Expenses by Date Range**
The system shall allow filtering the expense list by a start date, end date, or both.
Acceptance: Running `expense list --from 2026-03-01 --to 2026-03-31` shows only expenses within that date range (inclusive).
Priority: Must

**FR-004: Filter Expenses by Category**
The system shall allow filtering the expense list by category.
Acceptance: Running `expense list --category food` shows only expenses in the "food" category.
Priority: Must

**FR-005: Combine Filters**
The system shall allow combining date range and category filters in a single query.
Acceptance: Running `expense list --from 2026-03-01 --to 2026-03-31 --category food` shows only food expenses in March 2026.
Priority: Must

**FR-006: Edit an Expense**
The system shall allow the user to edit any field of an existing expense by its ID.
Acceptance: Running `expense edit 5 --amount 15.00 --note "Updated amount"` modifies expense ID 5's amount and note while preserving other fields. The system confirms the change and shows the updated record.
Priority: Must

**FR-007: Delete an Expense**
The system shall allow the user to delete an expense by its ID, with a confirmation prompt.
Acceptance: Running `expense delete 5` prompts "Delete expense #5? [y/N]" and removes the record only if confirmed.
Priority: Must

### Summary & Analysis

**FR-008: Spending Summary**
The system shall display a summary for a given time period showing: total spending, spending per category, and average daily spend.
Acceptance: Running `expense summary --from 2026-03-01 --to 2026-03-31` outputs total spending, a breakdown by category (sorted by amount descending), and average daily spend for the 31-day period.
Priority: Must

**FR-009: Default Summary Period**
When no date range is provided to the summary command, the system shall default to the current calendar month.
Acceptance: Running `expense summary` in March 2026 shows the summary for 2026-03-01 through 2026-03-31.
Priority: Should

### Category Management

**FR-010: Predefined Categories**
The system shall ship with a default set of categories: food, transport, housing, utilities, entertainment, health, shopping, education, and other.
Acceptance: A fresh install has all listed categories available.
Priority: Must

**FR-011: Add Custom Categories**
The system shall allow the user to add new categories.
Acceptance: Running `expense category add travel` adds "travel" to the available categories.
Priority: Must

**FR-012: List Categories**
The system shall list all available categories (default + user-added).
Acceptance: Running `expense category list` outputs all categories, one per line.
Priority: Must

### Data Export

**FR-013: Export to CSV**
The system shall export expenses to a CSV file, respecting any active filters (date range, category).
Acceptance: Running `expense export --from 2026-03-01 --to 2026-03-31 --output march.csv` creates a valid CSV file with headers: date, category, amount, note.
Priority: Must

### Data Persistence

**FR-014: SQLite Storage**
The system shall persist all data in a SQLite database at `~/.expense-tracker/expenses.db`. The database and directory shall be created automatically on first use.
Acceptance: After adding an expense, closing the terminal, and reopening it, `expense list` shows the previously added expense.
Priority: Must

### Web Dashboard

**FR-015: Dashboard Home — Expense List**
The web dashboard shall display all expenses in a table with columns for ID, date, category, amount, and note. The table shall support filtering by date range and category via query parameters or UI controls.
Acceptance: Navigating to the dashboard home page shows all expenses. Applying a category filter shows only expenses in that category. Applying a date range filter shows only expenses within that range.
Priority: Must

**FR-016: Dashboard — Add Expense**
The web dashboard shall provide a form to add a new expense with date, category, amount, and optional note. The category field shall present a dropdown of available categories.
Acceptance: Submitting the form with valid data creates the expense and it appears in the expense list. The form validates inputs (positive amount, valid date, known category) and shows errors for invalid input.
Priority: Must

**FR-017: Dashboard — Edit Expense**
The web dashboard shall allow editing any field of an existing expense. The edit form shall be pre-populated with the expense's current values.
Acceptance: Clicking edit on an expense opens a pre-filled form. Submitting changes updates the expense. The updated values appear in the expense list.
Priority: Must

**FR-018: Dashboard — Delete Expense**
The web dashboard shall allow deleting an expense with a confirmation step.
Acceptance: Clicking delete on an expense prompts for confirmation. Confirming removes the expense from the list. Cancelling leaves the expense unchanged.
Priority: Must

**FR-019: Dashboard — Spending Summary**
The web dashboard shall display a spending summary for a configurable time period showing total spending, per-category breakdown, and daily average.
Acceptance: The summary view shows total, per-category totals with percentages, and daily average. The user can adjust the date range.
Priority: Must

**FR-020: Dashboard — Category Management**
The web dashboard shall display all categories and allow adding new ones.
Acceptance: The category management view lists all categories. Adding a new category via the form makes it available for expense creation.
Priority: Should

**FR-021: Dashboard API**
The web dashboard shall be backed by a RESTful JSON API that the frontend consumes. The API shall reuse the existing service layer.
Acceptance: API endpoints exist for listing/creating/updating/deleting expenses, listing/creating categories, and getting summaries. All endpoints return JSON. The API uses the same `ExpenseService` and `CategoryService` as the CLI.
Priority: Must

**FR-022: Dashboard Server Command**
The CLI shall include a command to start the web dashboard server.
Acceptance: Running `expense dashboard` starts the FastAPI server and opens the dashboard in the default browser or displays the URL.
Priority: Should

## 4. Non-Functional Requirements

**NFR-001: CLI Startup Performance**
The CLI shall start and execute any command in under 1 second on typical hardware.
Acceptance: Timing 10 consecutive `expense list` calls on a database with 1,000 records averages under 1 second each.
Priority: Should

**NFR-002: Data Integrity**
The system shall not corrupt data on unexpected termination (e.g., Ctrl+C during a write).
Acceptance: SQLite transactions are used for all write operations. Interrupting a command mid-execution does not leave the database in an inconsistent state.
Priority: Must

**NFR-003: Clear Error Messages**
The system shall provide actionable error messages for invalid input (bad dates, unknown categories, negative amounts, nonexistent IDs) in both CLI and web dashboard.
Acceptance: Each error message states what went wrong and what the user should do instead. API errors return appropriate HTTP status codes (400, 404, 422) with JSON error bodies.
Priority: Must

**NFR-004: Test Coverage**
The project shall include unit tests for core logic, integration tests for CLI commands, and API tests for dashboard endpoints.
Acceptance: `pytest` runs all tests successfully. Tests cover the acceptance criteria of all Must-priority functional requirements.
Priority: Must

**NFR-005: Dashboard Responsiveness**
The web dashboard shall render correctly on desktop browsers at common viewport widths (1024px and above).
Acceptance: The dashboard layout is usable and readable at 1024px viewport width.
Priority: Should

## 5. Constraints

#### Technology Constraints

**C-001: Python**
The project shall be implemented in Python 3.10+. Reason: user's preferred language and ecosystem.

**C-002: Click Library**
The CLI shall be built using the Click library. Reason: user requirement for consistent CLI framework.

**C-003: SQLite**
Data persistence shall use SQLite via Python's built-in `sqlite3` module. No external database servers. Reason: zero-dependency local storage that works everywhere.

**C-004: FastAPI**
The web dashboard shall be built using FastAPI. Reason: user's chosen framework for the web interface.

#### Operational Constraints

**C-005: Single Currency**
All amounts are in USD. No currency conversion or multi-currency support. Reason: simplicity for the current version.

**C-006: Single User**
The system is designed for a single user on a single machine. No authentication, no multi-user access. Reason: personal expense tracking tool.

**C-007: Shared Database**
The CLI and web dashboard shall share the same SQLite database. Reason: data consistency across interfaces.

## 6. Scope Boundaries

### In Scope

- Adding, listing, editing, and deleting expenses via CLI
- Adding, listing, editing, and deleting expenses via web dashboard
- Filtering expenses by date range and/or category (both interfaces)
- Spending summaries with total, per-category breakdown, and average daily spend (both interfaces)
- Predefined categories with the ability to add custom ones (both interfaces)
- CSV export with filter support (CLI)
- SQLite persistence at a default location
- RESTful JSON API backing the web dashboard
- Unit, integration, and API tests

### Out of Scope

- Multi-currency support
- Cloud sync or backup
- Receipt scanning or image attachments
- Budget tracking or spending limits
- Recurring/scheduled expenses
- Multi-user or authentication
- Mobile-optimized dashboard layout
- CSV export from web dashboard

## 7. User Workflows

#### Quick Expense Entry (CLI)

**Trigger:** User makes a purchase and wants to log it immediately.
**Steps:**
1. User runs `expense add --category food --amount 12.50`.
2. System creates the expense with today's date, category "food", amount $12.50, no note.
3. System confirms: "Added expense #42: $12.50 in food on 2026-03-12."
**Expected Outcome:** Expense is persisted and visible in `expense list` and the web dashboard.
**What Could Go Wrong:** User specifies an unknown category — system shows an error listing available categories. User enters a negative amount — system rejects with a message.

#### Weekly Spending Review (CLI)

**Trigger:** End of week, user wants to see where money went.
**Steps:**
1. User runs `expense summary --from 2026-03-06 --to 2026-03-12`.
2. System displays total spending, category breakdown, and average daily spend.
3. User runs `expense list --from 2026-03-06 --to 2026-03-12 --category food` to drill into a specific category.
**Expected Outcome:** User has a clear picture of the week's spending patterns.
**What Could Go Wrong:** No expenses in the date range — system shows a message like "No expenses found for the given period" instead of an empty table.

#### Correcting a Mistake (CLI)

**Trigger:** User realizes they logged an expense with the wrong amount or category.
**Steps:**
1. User runs `expense list` to find the expense ID.
2. User runs `expense edit 42 --amount 15.00` to correct the amount.
3. System confirms the update and displays the corrected record.
**Expected Outcome:** The expense is updated in place; the ID remains the same.
**What Could Go Wrong:** User provides a nonexistent ID — system shows "Expense #99 not found."

#### Monthly Export (CLI)

**Trigger:** End of month, user wants to export data for a spreadsheet.
**Steps:**
1. User runs `expense export --from 2026-03-01 --to 2026-03-31 --output march-2026.csv`.
2. System writes the CSV file and confirms: "Exported 47 expenses to march-2026.csv."
**Expected Outcome:** A valid CSV file exists at the specified path, importable into any spreadsheet tool.
**What Could Go Wrong:** File path is not writable — system shows a clear error. No expenses match the filters — system creates an empty CSV with headers and notes "0 expenses exported."

#### Dashboard Expense Review

**Trigger:** User wants to visually browse and manage expenses.
**Steps:**
1. User starts the dashboard with `expense dashboard` or navigates to the running server URL.
2. Dashboard displays all expenses in a table.
3. User applies category and date filters to narrow the view.
4. User clicks an expense to edit it, updates the amount, and saves.
5. User views the summary section to see totals and category breakdown.
**Expected Outcome:** User can browse, filter, edit, and review spending entirely from the browser.
**What Could Go Wrong:** Server is not running — user sees a connection error in the browser. Invalid form input — dashboard shows inline validation errors.

#### Dashboard Quick Add

**Trigger:** User has the dashboard open and wants to add an expense without switching to the terminal.
**Steps:**
1. User clicks the "Add Expense" button on the dashboard.
2. User fills in the form: selects a category from the dropdown, enters the amount, optionally picks a date and adds a note.
3. User submits the form.
4. The new expense appears in the expense table.
**Expected Outcome:** Expense is persisted and visible in both the dashboard and `expense list` in the CLI.
**What Could Go Wrong:** User enters an invalid amount (negative or zero) — form shows a validation error. User leaves required fields empty — form prevents submission.

## 8. Success Criteria

1. **It's faster than a spreadsheet.** Adding an expense via CLI takes under 5 seconds from the moment the user starts typing the command.
2. **Data survives restarts.** Expenses persist across terminal sessions, reboots, and updates.
3. **Summaries are accurate.** The summary command and dashboard summary produce correct totals, category breakdowns, and averages that match manual calculations.
4. **Tests pass.** `pytest` runs cleanly with coverage of all Must-priority requirements.
5. **Corrections are easy.** Editing or deleting an expense is a single command (CLI) or a few clicks (dashboard).
6. **Dashboard works.** The web dashboard renders correctly, displays accurate data, and all CRUD operations function correctly.
7. **Interfaces stay in sync.** Changes made via the CLI are immediately visible in the dashboard, and vice versa.

## 9. Assumptions & Dependencies

- **Python 3.10+** is installed on the user's machine.
- **Click**, **FastAPI**, **Uvicorn**, and **pytest** are available via pip.
- The user's home directory (`~`) is writable and has sufficient disk space for a SQLite database.
- The SQLite database will remain small enough (< 100MB) that performance is not a concern for the foreseeable future.
- The web dashboard will be accessed from a local browser on the same machine (localhost). No remote access is required.
- Only one instance of the web server runs at a time (no concurrent server instances).

## 10. Open Questions

_None at this time._

## 11. Changelog

### v2 — 2026-03-12

#### Added
- FR-015: Dashboard Home — Expense List (Must)
- FR-016: Dashboard — Add Expense (Must)
- FR-017: Dashboard — Edit Expense (Must)
- FR-018: Dashboard — Delete Expense (Must)
- FR-019: Dashboard — Spending Summary (Must)
- FR-020: Dashboard — Category Management (Should)
- FR-021: Dashboard API (Must)
- FR-022: Dashboard Server Command (Should)
- NFR-005: Dashboard Responsiveness (Should)
- C-004: FastAPI constraint
- C-007: Shared Database constraint
- Two new user workflows: "Dashboard Expense Review" and "Dashboard Quick Add"
- New success criteria: #6 (Dashboard works) and #7 (Interfaces stay in sync)
- New dependencies: FastAPI, Uvicorn

#### Changed
- Problem Statement updated to describe the dual-interface system (CLI + web dashboard)
- Users & Stakeholders updated to include dashboard interaction patterns
- NFR-003 expanded to include API error responses (HTTP status codes, JSON error bodies)
- NFR-004 expanded to include API tests for dashboard endpoints
- Scope Boundaries: "GUI or web interface" moved from Out of Scope to In Scope
- Existing workflows updated to note that changes are visible across both interfaces

#### Removed
- "GUI or web interface" from Out of Scope list (now in scope as the web dashboard)
