# SPEC · NWP-101 — Payments export: let ops choose columns and scope

> Written before any code. Generated with `/spec`, then edited by a human.
> Load it as context when you build: `@docs/specs/NWP-101-export-options.md`

**Ticket:** [NWP-101](../tickets/NWP-101.md)
**Author:** Claude  
**Status:** draft

## Problem

Dana's ops team exports the payments table several times a day for merchant queries, reconciliation, and Finance requests. Today every export includes the card last-four, which forces 3–4 hours of manual spreadsheet cleanup per month before it can be shared with a merchant. A near-miss last quarter where an unedited file almost went to the wrong merchant makes this a safety issue.

## Current state

- Export button exists on `/payments` page (`src/app/payments/page.tsx:68-76`), currently links directly to `/api/payments/export?{query}`
- Export API route at `src/app/api/payments/export/route.ts` calls `toCsv(rows)` with fixed columns, reusing `filterPayments()` and `sortPayments()` from the query builder
- CSV helper at `src/lib/csv.ts:58-67` already supports a `columns` parameter; the signature is `toCsv(payments, columns = EXPORT_COLUMNS)`
- `exportFilename()` at `src/lib/csv.ts:69-71` generates a fixed filename with only the date
- Payment type (`src/data/types.ts:24-37`) includes all fields: id, merchantId, amount, currency, status, method, cardBrand, last4, createdAt, description
- Query builder in `src/data/queries.ts:45-70` is the single source of truth for filtering; `parseFilters()` at line 18 validates all client input against allowlists
- Tests in `src/lib/csv.test.ts` cover escaping, column selection, merchant resolution, and filename generation

## Domain rules

| Rule | Source | What breaks if ignored |
| --- | --- | --- |
| **Validate on the server** — anything from the client (column names, currencies, limits, statuses) is checked against an allowlist before it reaches a query, a filename, or the store | `build-battle/merchant-console/CLAUDE.md:42` | Column names could be injected, filenames could contain dangerous characters |
| **Money is integer minor units** — `$250.00` is `25000`; no floats, no strings with currency symbols | `build-battle/merchant-console/CLAUDE.md:39` | Cents drift; amounts disagree between display and export |
| **One query builder** — payment filtering goes through the builder behind `GET /api/payments`; a second implementation is a bug, not a shortcut | `build-battle/merchant-console/CLAUDE.md:41` | Filter logic diverges; exports and the table show different data |
| **Format at the edge** — money components receive minor units and a currency code and render the string; format nowhere else | `build-battle/merchant-console/.claude/rules/money.md` | Two formatters disagree; tests mock different outputs |

## Approach

Add a Modal dialog to the Export button that lets ops choose which columns to include and whether to export the current filter or all payments. The dialog closes on cancel; on Download it POST s the choices to a new `/api/payments/export-options` endpoint that validates the column names, builds the full query (with pagination bypassed for "all"), and returns CSV. The filename includes the scope (`disputed` if a filter is active, empty if exporting all). The current UX (link to `/api/payments/export?{filter}`) remains for future API clients.

**Considered and rejected:** Building the export in the browser with the current page's data. The payments table is paginated; this would export only the current page, shipping the bug we would be replacing instead of the feature. The server must fetch all matching rows when "all payments" is selected.

## File map

| File | Add or change | Why |
| --- | --- | --- |
| `src/app/payments/page.tsx` | Change | Replace the `<a>` Export button with a Button that opens a Dialog; pass current filters as context |
| `src/components/ExportDialog.tsx` | Add | Dialog component with column checkboxes (last4 off by default) and scope radio buttons |
| `src/app/api/payments/export/route.ts` | Change | Extend to accept `columns` and `scope` query parameters; validate server-side |
| `src/lib/csv.ts` | Change | Add `validateColumns()` function; update `exportFilename()` to accept scope parameter |
| `src/lib/csv.test.ts` | Change | Add tests for column validation and filename generation with scope |
| `src/data/queries.ts` | Change | Extend `parseFilters()` to accept and validate `columns` parameter (or add separate validation) |

## Plan

1. **Add ExportDialog component** — UI component with column checkboxes and scope radio buttons, validates that at least one column is selected before Download is enabled. Done when: Dialog opens/closes on button click, Download button is disabled until a column is selected.

2. **Update export API to accept parameters** — Extend `/api/payments/export` to parse and validate `columns` and `scope` from the query string. Done when: `curl /api/payments/export?columns=id,amount&scope=current` returns CSV with only id and amount columns.

3. **Implement column validation** — Add `validateColumns(names: string[]): ExportColumn[]` to `src/lib/csv.ts` that checks each name against `EXPORT_COLUMNS` and rejects invalid ones. Done when: unit test passes for valid columns, invalid columns raise an error.

4. **Update exportFilename()** — Accept optional `scope` and `filter` parameters to include scope in the filename (e.g., `payments-disputed-2026-03-14.csv` for a filtered export, `payments-all-2026-03-14.csv` for all). Done when: test shows filenames reflect the scope.

5. **Wire up the dialog** — Replace the Export button link with a Button that opens the dialog. On Download, POST the choices to `/api/payments/export` and trigger the download. Done when: clicking Export opens the dialog, selecting columns and clicking Download produces a CSV with the right columns and scope.

6. **Add tests** — Extend `src/lib/csv.test.ts` with tests for `validateColumns()` and filename generation with scope. Done when: `npm test` passes and coverage includes the new validation logic.

## Verification

| Acceptance criterion | How it is proven |
| --- | --- |
| Ops can choose which columns are included. Card last-four is **off** by default. | Open `/payments`, click Export, verify checkboxes for all columns, verify `last4` is unchecked, check it, click Download, verify last4 is in the CSV. |
| Ops can choose scope: **current filter** or **all payments**. Current filter is the default, and the row count is visible before download. | Open `/payments`, apply a filter (e.g., status=disputed), click Export, verify current filter is selected and shows row count, select "all payments", verify count updates, click Download, verify export includes all payments regardless of filter. |
| The filename reflects the scope and the date, for example `payments-disputed-2026-08-13.csv`. | Apply a filter, export, verify filename includes the active filter name and date. Export with no filter, verify filename says `payments-all-{date}.csv`. |
| Amounts stay in minor units internally and are formatted once on the way out, with currency in its own column. | Open a downloaded CSV, verify amount column shows formatted values like `$250.00` and currency column shows `USD`. |
| Deselecting every column disables Download rather than producing an empty file. | Open Export dialog, uncheck all columns, verify Download button is disabled. |

## Risks

- **Column names must be validated server-side to prevent injection.** The column name check is done in `validateColumns()` against the `EXPORT_COLUMNS` constant; no client-provided name reaches the CSV serializer.
- **Exporting "all payments" could be slow or expensive.** Today the store is in-memory and small; this is not a blocker. Persistence is NWP-203. If load becomes an issue, add pagination or a background job then.
- **The dialog's state (selected columns, scope) is not persisted.** It resets every time it opens. This is acceptable; ops make a choice each time, and there is no "remember my preference" requirement in the ticket.

## Out of scope

- **UI polish:** Focus management, keyboard navigation, and animation are out of scope. The Dialog component already exists; use it as-is.
- **Alternative export formats** (JSON, Parquet, etc.). The ticket asks for CSV only.
- **Scheduled or background exports.** All exports happen on demand.
- **Undo or export history.** There is no requirement to audit what was exported or by whom.

## Open questions

- Should the "current filter" scope show a human-readable label of what is being filtered (e.g., "disputed payments, Acme Corp"), or just the row count? The ticket says "row count is visible"; I interpret this as just the count, but could be clarified.
