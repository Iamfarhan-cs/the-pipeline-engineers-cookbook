# E08 — Extract Data from Excel Files

Excel files are common in finance, operations, logistics, compliance, reporting, and partner integrations.

Unlike CSV, an Excel workbook is a container that can hold multiple sheets, formulas, formatting, merged cells, hidden content, named ranges, and metadata.

That flexibility makes Excel extraction fundamentally different from plain-text file extraction.

The goal is:

> Extract the intended worksheet data while proving that the workbook, sheet, structure, and values match the agreed source contract.

## 1. Problem Recognition

Common production problems include:

- the workbook contains multiple sheets
- the wrong sheet is selected
- sheet names change
- headers start several rows below the top
- merged cells distort the apparent table structure
- formulas are mistaken for stored values
- cached formula values are stale or unavailable
- hidden sheets contain unexpected data
- columns change between workbooks
- numeric identifiers lose formatting or precision
- dates are interpreted incorrectly
- blank rows appear inside the dataset
- a workbook is partially uploaded
- duplicate files are delivered
- a very large workbook exhausts memory
- macros or external links introduce operational risk

Do not equate:

    workbook opens successfully

with:

    workbook contains the correct extractable dataset

## 2. Excel Extraction Architecture

    FILE ARRIVES
        ↓
    DISCOVER
        ↓
    VERIFY COMPLETENESS
        ↓
    IDENTIFY WORKBOOK
        ↓
    INSPECT SHEETS
        ↓
    SELECT EXPECTED SHEET
        ↓
    DETECT / VALIDATE HEADER
        ↓
    READ ROWS
        ↓
    VALIDATE RECORDS
        ↓
    PERSIST
        ↓
    CHECKPOINT
        ↓
    MARK COMPLETE

Each stage has a different responsibility.

## 3. Workbook vs Worksheet

An Excel file is a workbook.

A workbook can contain:

- multiple worksheets
- hidden worksheets
- formulas
- tables
- charts
- named ranges
- formatting
- workbook metadata

The extraction contract must identify the intended worksheet or selection rule.

## 4. Sheet Discovery

Never assume the first sheet is the correct dataset.

Inspect:

- sheet names
- visible/hidden state
- approximate dimensions
- expected sheet names
- workbook version

Example expectation:

    Sheet: Payments

Possible producer change:

    Sheet: Payment Export

That should be treated as a contract change, not silently accepted if the pipeline depends on the original name.

## 5. Sheet Selection

Prefer an explicit selection rule.

Examples:

    exact sheet name

or:

    workbook metadata + expected dataset marker

or:

    named table/range

Do not select:

    workbook.active

as a production contract unless the producer explicitly defines the active sheet as the interface.

## 6. Basic Python Extraction

Using openpyxl:

    from openpyxl import load_workbook

    workbook = load_workbook(
        'payments.xlsx',
        read_only=True,
        data_only=True,
    )

    sheet = workbook['Payments']

    for row in sheet.iter_rows(values_only=True):
        process(row)

Read-only mode is useful for large workbooks because it avoids loading the full workbook into a normal editable object model.

## 7. Formula Values

Excel cells may contain formulas:

    =SUM(B2:B10)

An extractor may need either:

- the formula expression
- the cached calculated value

`data_only=True` asks openpyxl to return cached values rather than formula expressions where available.

Important:

> Python does not calculate Excel formulas for you merely by reading the workbook.

If the workbook was not recalculated and saved by Excel or another calculation engine, cached values may be stale or missing.

Define whether the pipeline trusts cached values.

## 8. Formula Policy

Choose an explicit policy:

- accept calculated cached values
- extract formulas as metadata
- reject formula cells
- require producer-side recalculation

For financial or regulatory extracts, silently using stale formula values can be dangerous.

## 9. Header Detection

Excel exports frequently contain title rows before the actual table.

Example:

    Payment Export — September 2026
    Generated: 2026-09-26
    
    ID | Amount | Currency | Status

The header begins on row 4.

Do not assume row 1 is always the header.

Define a header rule based on:

- expected column names
- named table
- known row position
- source-specific marker

## 10. Header Validation

Validate:

- required columns
- duplicate column names
- expected data types
- unexpected columns
- column order where required

Example contract:

    ID
    Amount
    Currency
    Status

Do not silently map a changed header to an old field name unless that mapping is part of the source contract.

## 11. Merged Cells

Merged cells are common in human-designed spreadsheets.

Example:

    A1:D1 = 'September Payments'

The value may appear only in the top-left cell of the merged range.

Merged cells can make a visual report look structured while the underlying rows are not a clean table.

Before extraction, determine whether the worksheet is:

- a machine-readable table
- a human report
- a mixture of both

Do not treat every visible spreadsheet region as a relational table.

## 12. Blank Rows

Blank rows may separate report sections.

Example:

    headers
    records
    blank row
    totals
    notes

A naive extractor may interpret everything after the header as records.

Define the table boundary explicitly.

## 13. Totals and Subtotals

Human reports often contain:

    Payment 1
    Payment 2
    Payment 3
    Total

The total row is not a payment record.

Possible detection strategies:

- explicit table/range
- known total marker
- schema/type validation
- source-specific contract

Never rely only on row position if the producer can add rows.

## 14. Excel Tables

Excel can contain structured tables with a defined range and column names.

A named table can be a stronger extraction boundary than visually guessing the data region.

Prefer machine-defined boundaries when the producer can provide them.

## 15. Named Ranges

Named ranges can identify stable regions of a workbook.

Example:

    PaymentsExtract

They can be useful when sheet layout changes but the producer maintains the named range contract.

Do not assume named ranges are automatically correct. Validate their dimensions and columns.

## 16. Data Types

Excel stores values using spreadsheet-specific representations.

Potential values include:

- strings
- numbers
- booleans
- dates
- times
- formulas
- errors
- blanks

The extractor should map these into explicit pipeline types.

## 17. Dates and Times

Excel dates can be represented using serial values with workbook date-system rules.

Python libraries can convert recognized Excel date cells to Python date/datetime values.

Do not assume every numeric value is a date simply because a cell looks like a date in Excel.

Use cell metadata and the source contract.

## 18. Number Formatting vs Value

A cell may display:

    001234

while the underlying value is numeric:

    1234

If `001234` is a business identifier, converting it to an integer can lose meaningful leading zeros.

Distinguish:

    numeric measurement

from:

    formatted identifier

Identifiers should often remain strings.

## 19. Precision

Financial values should not be handled casually as binary floating-point values.

Define how spreadsheet numeric values map to pipeline types.

For monetary fields, a decimal representation is generally safer when exact decimal semantics are required.

## 20. Error Cells

Excel can contain formula/error values such as:

    #N/A
    #VALUE!
    #DIV/0!

These are not normal business values.

Define whether they should be:

- rejected
- converted to null
- quarantined
- preserved as error metadata

Do not silently turn calculation errors into valid values.

## 21. Hidden Sheets

Hidden worksheets may contain:

- helper calculations
- lookup data
- old exports
- internal information
- sensitive data

Do not process hidden sheets simply because they exist.

Explicitly define whether hidden sheets are allowed as extraction sources.

## 22. Very Hidden Sheets

Excel also supports stronger hidden states in some workbook models.

These can be used for implementation details rather than user-facing datasets.

Treat non-visible worksheets as non-contractual unless explicitly documented.

## 23. Workbook Metadata

Useful metadata can include:

- workbook filename
- file size
- checksum
- sheet names
- workbook properties
- extraction timestamp

Metadata can help diagnose producer changes.

Do not treat workbook metadata as business data unless the contract says so.

## 24. File Completeness

An Excel file may appear in a shared location before the producer has finished writing it.

Prefer:

    WRITE TEMP
       ↓
    CLOSE
       ↓
    VERIFY
       ↓
    ATOMIC RENAME
       ↓
    PUBLISHED XLSX

If no completion signal exists, stable-size checks are only heuristics.

A workbook that opens sometimes is not evidence that the publication process is correct.

## 25. File Identity

Track:

    source
    filename
    file_size
    checksum
    discovered_at

A SHA-256 checksum can identify exact file content.

File identity protects against duplicate delivery and accidental replacement.

## 26. Duplicate Files

The same workbook can arrive multiple times.

Possible causes:

- transfer retries
- scheduler retries
- manual resend
- partner re-upload

Use durable processed-file state.

Filename alone is insufficient if file contents can change.

## 27. Duplicate Records

Different workbooks can contain the same logical record.

Use a stable source or business identifier where available.

File-level idempotency does not replace record-level deduplication.

## 28. Checkpointing

Checkpoint state can contain:

    file_id
    checksum
    sheet_name
    row_number
    batch_number
    last_source_id

Always bind checkpoint state to the exact workbook identity.

If the workbook changes, do not continue using an old row position blindly.

## 29. Large Workbooks

Large Excel files require bounded memory.

Useful strategies include:

- read-only workbook mode
- row iteration
- bounded batches
- extracting only required sheets
- producer-side splitting

Do not load every worksheet when the pipeline needs one.

## 30. Workbook Limitations

Excel is a business-file format, not an ideal high-volume data interchange format.

Problems include:

- large file sizes
- complex formatting
- formulas
- hidden state
- human editing
- difficult schema contracts
- limited streaming compared with purpose-built data formats

If the source volume becomes large or frequent, consider whether the producer should provide CSV, JSONL, Parquet, database access, or another machine-oriented interface.

## 31. Macros and External Links

Workbooks may contain macros or external links.

Extraction systems should not execute arbitrary workbook code.

Do not enable macro execution merely to read data.

Treat external links as untrusted dependencies unless explicitly required and controlled.

The extraction worker should read workbook data without turning the ingestion process into an execution environment.

## 32. Malformed Workbooks

A corrupted workbook may fail during parsing.

Possible causes:

- incomplete transfer
- damaged ZIP container
- invalid XML parts inside XLSX
- producer software failure
- manual file corruption

Classify this as a file-level failure unless the parser can safely identify a valid subset according to the contract.

Do not silently continue with a partially readable workbook.

## 33. XLS vs XLSX

Excel files can use different formats.

Modern `.xlsx` files are based on Office Open XML.

Older `.xls` files use a different binary format.

Do not assume one parser handles every Excel format equally.

Identify the supported source formats explicitly.

## 34. Testing

Test at least:

1. Valid workbook.
2. Multiple worksheets.
3. Wrong sheet.
4. Renamed sheet.
5. Hidden sheet.
6. Header below row 1.
7. Merged cells.
8. Blank rows.
9. Totals row.
10. Named table.
11. Named range.
12. Formula cells.
13. Missing cached formula values.
14. Excel date.
15. Leading-zero identifier.
16. Numeric precision.
17. Error cell.
18. Missing required column.
19. Unexpected column.
20. Duplicate file.
21. Duplicate record.
22. Large workbook.
23. Incomplete workbook.
24. Corrupted workbook.
25. Checkpoint restart.
26. File mutation.
27. Persistence failure.
28. Macro/external-link presence.

## 35. Observability

Useful metrics:

    excel_files_discovered_total
    excel_files_processed_total
    excel_files_failed_total
    excel_files_quarantined_total
    excel_files_duplicated_total
    excel_sheets_inspected_total
    excel_rows_read_total
    excel_rows_persisted_total
    excel_rows_rejected_total
    excel_schema_errors_total
    excel_formula_errors_total
    excel_processing_duration_seconds
    excel_bytes_processed_total

Useful log fields:

    extraction_run_id
    file_id
    filename
    checksum
    sheet_name
    row_number
    batch_number
    workbook_version
    rows_processed
    error_type

Do not log complete worksheets or sensitive cell values unnecessarily.

## 36. Intentional Failure

### Failure drill 1 — Rename the expected sheet

Change Payments to Payment Export.

Verify the extractor fails according to its contract instead of silently selecting another sheet.

### Failure drill 2 — Move the header

Insert several title rows before the header.

Verify header detection behaves according to policy.

### Failure drill 3 — Add a total row

Add a subtotal or total below the dataset.

Verify it is not interpreted as a normal record.

### Failure drill 4 — Formula cache problem

Create formula cells without reliable cached values.

Verify the formula policy detects the condition.

### Failure drill 5 — Duplicate workbook

Deliver the same workbook twice.

Verify file-level idempotency.

### Failure drill 6 — Persistence failure

Fail persistence halfway through a worksheet.

Restart and verify checkpoint and downstream idempotency behavior.

### Failure drill 7 — Workbook mutation

Change the workbook after checkpointing.

Verify the old checkpoint cannot be reused blindly.

## 37. Recovery

When Excel extraction fails:

1. Identify the extraction run and workbook.
2. Verify filename, size, and checksum.
3. Confirm the workbook was completely published.
4. Inspect sheet names and visibility.
5. Validate the expected sheet and header.
6. Determine whether the problem is parsing, schema, formula, or record related.
7. Verify checkpoint state.
8. Confirm the workbook has not changed.
9. Resume only from a valid checkpoint.
10. Reconcile expected and persisted rows.
11. Mark the workbook complete only after verification.

Never silently choose a different worksheet as a recovery shortcut.

## 38. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **openpyxl** | Python library for reading and working with XLSX workbooks, including worksheets, cells, formulas, and workbook metadata. |
| **pandas** | Useful for tabular Excel extraction when workbook size and memory requirements are controlled. |
| **LibreOffice** | Useful to know for controlled spreadsheet recalculation/conversion workflows, not as an implicit dependency of ingestion. |

> These are reference tools for production vocabulary. The underlying Excel extraction mechanics should still be understood independently.

## 39. Production Runbook

### Workbook cannot be opened

Check:

1. File completeness.
2. File checksum.
3. File format.
4. Transfer corruption.
5. Producer software.

### Expected sheet cannot be found

Check:

1. Sheet names.
2. Hidden state.
3. Producer version.
4. Workbook contract.
5. Recent source changes.

Do not automatically select the first available sheet.

### Row count is unexpectedly high

Check:

1. Header detection.
2. Blank rows.
3. Total/subtotal rows.
4. Wrong sheet.
5. Duplicate records.

### Formula values are missing

Check:

1. Whether formulas are expected.
2. Cached values.
3. Producer recalculation.
4. Formula policy.

### What not to do

Do not:

- assume the active sheet is the data source
- assume row 1 contains headers
- treat visual formatting as a schema
- execute macros during extraction
- blindly trust cached formula values
- convert identifiers to numbers when formatting matters
- load huge workbooks entirely into memory
- reuse checkpoints against changed files
- silently switch to another sheet after a contract failure

## 40. Common Mistakes

### Mistake 1 — Treating Excel like CSV

Excel is a workbook container with sheets, formulas, formatting, and metadata.

### Mistake 2 — Selecting the first sheet

Workbook ordering is not a reliable data contract.

### Mistake 3 — Assuming row 1 is the header

Human reports often contain titles and metadata above the table.

### Mistake 4 — Trusting displayed values blindly

Formatting and underlying values can differ.

### Mistake 5 — Ignoring formulas

Cached values may be stale or unavailable.

### Mistake 6 — Ignoring hidden sheets

Hidden content may contain sensitive or irrelevant data.

### Mistake 7 — Unbounded memory

Large workbooks can exhaust extraction workers.

## 41. Definition of Done

You are done when you can:

- explain the difference between workbook and worksheet
- inspect workbook structure before extraction
- select a sheet using an explicit contract
- detect headers safely
- handle merged cells
- identify totals and non-record rows
- use Excel tables and named ranges deliberately
- understand formula versus cached values
- handle dates and numeric identifiers correctly
- preserve precision where required
- detect Excel error cells
- reason about hidden worksheets
- process large workbooks with bounded memory
- avoid executing macros
- identify duplicate workbooks
- identify duplicate records
- checkpoint safely
- detect workbook mutation
- test schema and workbook failures
- intentionally break Excel extraction
- recover from failed extraction
- operate Excel extraction with a production runbook

## 42. What You Learned

The central principle is:

> An Excel workbook is a structured business document, not automatically a clean table.

A production Excel pipeline follows:

    DISCOVER
       ↓
    VERIFY COMPLETENESS
       ↓
    IDENTIFY WORKBOOK
       ↓
    INSPECT SHEETS
       ↓
    SELECT CONTRACTUAL SHEET
       ↓
    VALIDATE HEADER / TABLE
       ↓
    READ + VALIDATE ROWS
       ↓
    PERSIST
       ↓
    CHECKPOINT
       ↓
    MARK COMPLETE

The key question is:

> What proves that I extracted the intended worksheet, from the intended workbook version, using the intended table boundary and value semantics?

That question drives sheet selection, header detection, formula handling, type preservation, file identity, checkpointing, testing, observability, and recovery.