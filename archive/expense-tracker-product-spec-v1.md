# Expense Tracker CLI — Product Specification

**Version:** v1
**Date:** 2026-03-12
**Previous Version:** N/A — initial draft
**Status:** Draft
**Author(s):** Claude (with user input)

---

## 1. Problem Statement & Vision

Tracking personal expenses is tedious when it requires opening a spreadsheet or app. Most off-the-shelf tools are bloated with features that get in the way of the core task: recording what you spent, seeing where your money goes, and understanding your spending patterns over time.

This project builds a lightweight CLI expense tracker in Python that makes it fast to log expenses from the terminal, query them with filters, and get spending summaries — all backed by a local SQLite database that persists between sessions. The tool should feel like a natural part of a developer's terminal workflow: quick to invoke, predictable in behavior, and easy to script.

The end state is a single CLI tool that handles the full lifecycle of expense tracking — add, list, edit, delete, summarize, and export — without requiring any external services or accounts.

## 2. Users & Stakeholders

### Primary User: Individual Developer / Power User

- **Role:** Single user tracking personal expenses from the terminal.
- **Interaction:** Invokes the CLI multiple times per day to log expenses. Reviews summaries weekly or monthly. Occasionally exports data to CSV for use in spreadsheets or other tools.
- **Needs:** Speed of entry (most commands should take under 5 seconds to type), reliable persistence, clear and scannable output, and the ability to correct mistakes (edit/delete).
- **Frustrations:** Slow startup, overly verbose output, losing data, having to remember complex command syntax.

## 3. Functional Requirements

### Expense Management

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

## 4. Non-Functional Requirements

**NFR-001: Startup Performance**
The CLI shall start and execute any command in under 1 second on typical hardware.
Acceptance: Timing 10 consecutive `expense list` calls on a database with 1,000 records averages under 1 second each.
Priority: Should

**NFR-002: Data Integrity**
The system shall not corrupt data on unexpected termination (e.g., Ctrl+C during a write).
Acceptance: SQLite transactions are used for all write operations. Interrupting a command mid-execution does not leave the database in an inconsistent state.
Priority: Must

**NFR-003: Clear Error Messages**
The system shall provide actionable error messages for invalid input (bad dates, unknown categories, negative amounts, nonexistent IDs).
Acceptance: Each error message states what went wrong and what the user should do instead.
Priority: Must

**NFR-004: Test Coverage**
The project shall include unit tests for core logic (database operations, filtering, summary calculations) and integration tests for CLI commands (invoking commands and verifying output).
Acceptance: `pytest` runs all tests successfully. Tests cover the acceptance criteria of all Must-priority functional requirements.
Priority: Must

## 5. Constraints

#### Technology Constraints

**C-001: Python**
The project shall be implemented in Python 3.10+. Reason: user's preferred language and ecosystem.

**C-002: Click Library**
The CLI shall be built using the Click library. Reason: user requirement for consistent CLI framework.

**C-003: SQLite**
Data persistence shall use SQLite via Python's built-in `sqlite3` module. No external database servers. Reason: zero-dependency local storage that works everywhere.

#### Operational Constraints

**C-004: Single Currency**
All amounts are in USD. No currency conversion or multi-currency support. Reason: simplicity for the current version.

**C-005: Single User**
The system is designed for a single user on a single machine. No authentication, no multi-user access. Reason: personal expense tracking tool.

## 6. Scope Boundaries

### In Scope

- Adding, listing, editing, and deleting expenses via CLI
- Filtering expenses by date range and/or category
- Spending summaries with total, per-category breakdown, and average daily spend
- Predefined categories with the ability to add custom ones
- CSV export with filter support
- SQLite persistence at a default location
- Unit and integration tests

### Out of Scope

- Multi-currency support
- GUI or web interface
- Cloud sync or backup
- Receipt scanning or image attachments
- Budget tracking or spending limits
- Recurring/scheduled expenses
- Multi-user or authentication

## 7. User Workflows

#### Quick Expense Entry

**Trigger:** User makes a purchase and wants to log it immediately.
**Steps:**
1. User runs `expense add --category food --amount 12.50`.
2. System creates the expense with today's date, category "food", amount $12.50, no note.
3. System confirms: "Added expense #42: $12.50 in food on 2026-03-12."
**Expected Outcome:** Expense is persisted and visible in `expense list`.
**What Could Go Wrong:** User specifies an unknown category — system shows an error listing available categories. User enters a negative amount — system rejects with a message.

#### Weekly Spending Review

**Trigger:** End of week, user wants to see where money went.
**Steps:**
1. User runs `expense summary --from 2026-03-06 --to 2026-03-12`.
2. System displays total spending, category breakdown, and average daily spend.
3. User runs `expense list --from 2026-03-06 --to 2026-03-12 --category food` to drill into a specific category.
**Expected Outcome:** User has a clear picture of the week's spending patterns.
**What Could Go Wrong:** No expenses in the date range — system shows a message like "No expenses found for the given period" instead of an empty table.

#### Correcting a Mistake

**Trigger:** User realizes they logged an expense with the wrong amount or category.
**Steps:**
1. User runs `expense list` to find the expense ID.
2. User runs `expense edit 42 --amount 15.00` to correct the amount.
3. System confirms the update and displays the corrected record.
**Expected Outcome:** The expense is updated in place; the ID remains the same.
**What Could Go Wrong:** User provides a nonexistent ID — system shows "Expense #99 not found."

#### Monthly Export

**Trigger:** End of month, user wants to export data for a spreadsheet.
**Steps:**
1. User runs `expense export --from 2026-03-01 --to 2026-03-31 --output march-2026.csv`.
2. System writes the CSV file and confirms: "Exported 47 expenses to march-2026.csv."
**Expected Outcome:** A valid CSV file exists at the specified path, importable into any spreadsheet tool.
**What Could Go Wrong:** File path is not writable — system shows a clear error. No expenses match the filters — system creates an empty CSV with headers and notes "0 expenses exported."

## 8. Success Criteria

1. **It's faster than a spreadsheet.** Adding an expense takes under 5 seconds from the moment the user starts typing the command.
2. **Data survives restarts.** Expenses persist across terminal sessions, reboots, and updates.
3. **Summaries are accurate.** The summary command produces correct totals, category breakdowns, and averages that match manual calculations.
4. **Tests pass.** `pytest` runs cleanly with coverage of all Must-priority requirements.
5. **Corrections are easy.** Editing or deleting an expense is a single command, not a multi-step process.

## 9. Assumptions & Dependencies

- **Python 3.10+** is installed on the user's machine.
- **Click** and **pytest** are available via pip.
- The user's home directory (`~`) is writable and has sufficient disk space for a SQLite database.
- The SQLite database will remain small enough (< 100MB) that performance is not a concern for the foreseeable future.

## 10. Open Questions

_None at this time. All key decisions were resolved during the interview phase._

## 11. Changelog

### v1 — 2026-03-12
- Initial draft based on user interview.
- Key decisions made during interview:
  - Single currency (USD) only.
  - Categories are predefined with the ability to add custom ones.
  - Date defaults to today when not provided.
  - CSV export included in scope.
  - Database at default location (`~/.expense-tracker/expenses.db`).
  - Edit and delete supported (not append-only).
