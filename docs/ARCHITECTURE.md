# Migratex Architecture

Deep-dive technical documentation for the Migratex ETL pipeline. For an overview, see the [README](../README.md).

## Design Principles

Migratex was built around four principles, each a direct response to problems encountered during the original client migration:

1. **Configuration over code** — Adding a new data source should never require touching pipeline logic. All table-specific behavior is defined in configuration files.
2. **Fail loudly, recover gracefully** — Errors are categorized, logged, and handled per category. A batch failure rolls back the batch; a row failure logs and continues.
3. **SQL safety is non-negotiable** — No string interpolation in queries. No hardcoded credentials. Every identifier is escaped or validated before use.
4. **Deterministic ordering** — Import order respects foreign key dependencies so no constraint violation ever halts a migration mid-run.

## Pipeline Overview

```
Excel Workbook
      │
      ▼
┌───────────────┐
│   Extractor   │   openpyxl → pandas DataFrames
│   (per sheet) │   Header normalization, dedup, null pruning
└───────┬───────┘
        │  Clean DataFrames
        ▼
┌───────────────┐
│    Cleaner    │   Per-table rules from cleaning_config.py
│               │   Phone/email/name normalization
│               │   Encoding fixes, FK validation
└───────┬───────┘
        │  Cleaned CSV files
        ▼
┌───────────────┐
│   Validator   │   Schema check against target DB
│   (pre-import)│   Type coercion, null checks, FK preview
└───────┬───────┘
        │  Validated batch
        ▼
┌───────────────┐
│   Importer    │   COPY (fast) or INSERT ON CONFLICT
│               │   Batch transactions with rollback
└───────┬───────┘
        │
        ▼
┌───────────────┐
│   Reporter    │   Structured JSON + text report
│  (post-import)│   Success rate, failures, stats
└───────────────┘
```

## Component Contracts

Every pipeline component inherits from a base class and implements a uniform `execute()` method. This gives the CLI a single way to invoke any step and collect results.

```python
class BaseComponent:
    def execute(self, context: PipelineContext) -> ComponentResult:
        """
        Returns:
            ComponentResult with:
              - status: 'success' | 'partial' | 'failed'
              - stats: dict of metrics (rows processed, errors, timing)
              - errors: list of structured error records
              - artifacts: file paths produced (CSVs, reports)
        """
```

**Why this matters:** Any component can be swapped, tested in isolation, or chained into a new pipeline without touching the others. The extractor doesn't know the cleaner exists. The cleaner doesn't know about the importer.

## Pipeline Phases

### Phase 1 — Extraction

**Input:** Excel file path or directory
**Output:** One cleaned CSV per sheet in `migratex_output/<workbook>/`

Steps:

1. Open workbook with `openpyxl` (read-only mode for memory efficiency on large files).
2. For each sheet:
   - Read into pandas DataFrame
   - Normalize headers: strip whitespace, lowercase, replace spaces with underscores
   - Detect and remove empty columns (100% null)
   - Detect and flag high-null columns (>80% null) for removal unless whitelisted
   - Handle Excel error values (`#REF!`, `#N/A`, `#VALUE!`) by replacing with NULL
   - Drop exact duplicate rows (configurable)
3. Write cleaned CSV with UTF-8 encoding.
4. Emit per-sheet extraction stats (rows in, rows out, duplicates removed, columns dropped).

**Memory profile:** Peak usage ~2x the size of the largest sheet. A 50 MB Excel file processes in ~100 MB RAM.

### Phase 2 — Cleaning

**Input:** CSV file + table name
**Output:** Transformed CSV ready for import

The cleaner reads per-table rules from `cleaning_config.py`:

```python
CLEANING_RULES = {
    'customers': {
        'phone_columns': ['phone', 'mobile'],
        'email_columns': ['email'],
        'name_split': {'source': 'full_name', 'first': 'first_name', 'last': 'last_name'},
        'encoding_fixes': True,
        'required_fields': ['email', 'phone'],
    },
    ...
}
```

**Phone normalization** runs through `normalize_phone(raw, region)` which:
- Strips all non-digit characters
- Validates digit count against the region's expected length
- Reformats to the region's canonical format
- Returns `None` for invalid numbers (logged separately, not dropped)

**Email normalization** lowercases, strips whitespace, validates against a regex, and rejects multi-@ addresses.

**Name splitting** uses whitespace-based tokenization with a fallback for single-word names (first name only, last name null).

**Encoding fixes** target Spanish characters mangled by Excel's limited UTF-8 handling — á→a, ñ→n style corrections are explicitly *not* applied (destructive); instead, common mojibake patterns (`Ã¡` → `á`, `Ã±` → `ñ`) are reversed.

### Phase 3 — Validation

**Input:** Cleaned CSV + target table schema
**Output:** Validation report, pass/fail decision

The validator connects to the target database and fetches the schema for the destination table. For each CSV row it checks:

- Column count matches (with mappings applied)
- Type coercion is possible (date strings → date, numeric strings → int/float)
- NOT NULL constraints satisfied
- Foreign key references exist in master tables
- Unique constraints won't be violated (checked via staged SELECT)

Violations are categorized:

| Severity | Example                                  | Action              |
| -------- | ---------------------------------------- | ------------------- |
| **Fatal**   | Missing required column, wrong type      | Halt batch          |
| **Warning** | FK reference missing, will be skipped    | Log and continue    |
| **Info**    | Optional column missing, will be null    | Log only            |

### Phase 4 — Import

**Input:** Validated CSV + import mode
**Output:** Rows written to target database + import report

Two modes, automatic selection based on presence of `ON CONFLICT` requirements:

**COPY mode** (`--mode fast`):
- Uses PostgreSQL's `COPY FROM STDIN` via `psycopg2.cursor.copy_expert()`
- Streams CSV directly to the database (no intermediate INSERT statements)
- 10–50× faster than row-by-row INSERT
- Requires clean data — no duplicate keys, no FK violations
- Throughput: ~3,000 rows/minute on a Supabase hosted pooler

**INSERT mode** (default):
- `INSERT ... ON CONFLICT (key) DO UPDATE SET ...` for upsert semantics
- Batched (default 1,000 rows per transaction)
- Safe for incremental updates, re-runs, and dirty data
- Throughput: ~500–800 rows/minute depending on row size

**FK dependency ordering** is computed from `cleaning_config.py`:

```python
IMPORT_ORDER = [
    # Master data (no FK dependencies)
    'locations', 'categories', 'vendors',
    # Dependent data (FK to master)
    'products', 'customers',
    # Transactional (FK to dependent)
    'orders', 'order_items',
]
```

The importer walks this list in order so a master table is always fully loaded before any dependent table begins.

### Phase 5 — Reporting

**Input:** Accumulated stats from all phases
**Output:** JSON report + human-readable text summary

The reporter aggregates:

- Total rows processed at each phase
- Rows dropped (duplicates, validation failures)
- FK violations logged
- Timing breakdown per component
- Success rate percentage
- Output file manifest

Reports are written to `migratex_output/logs/` with timestamps. The JSON format is machine-readable for CI integration.

## Error Handling Strategy

Errors are handled at the category level, not the exception level. Each category has a defined recovery action:

| Category                    | Example                             | Action                        |
| --------------------------- | ----------------------------------- | ----------------------------- |
| `ConnectionError`           | Database unreachable                | Retry with exponential backoff (3 attempts) |
| `SQLSafetyError`            | Invalid identifier detected         | Halt immediately (never recover — indicates injection attempt) |
| `ForeignKeyViolation`       | FK reference missing in target     | Log, skip row, continue batch |
| `DataTypeError`             | Cannot coerce string to int        | Attempt automatic conversion; log if coercion fails |
| `DuplicateKeyError`         | Primary key collision              | Silently skip (idempotent re-runs) |
| `FileNotFoundError`         | Excel file missing                  | Halt, user-facing error at CLI layer |
| `SchemaValidationError`     | CSV columns don't match target    | Halt batch, emit full diff in report |

**Batch-level safety:** Every batch runs inside a single transaction. If any uncategorized exception bubbles up, the transaction rolls back — no partial data.

**Row-level safety:** Recoverable errors (FK violations, duplicates, type coercions) are logged to a separate `rejected_records/` directory for manual review without stopping the migration.

## Security Model

### SQL Injection Prevention

**Principle:** No string interpolation in any SQL query. Every identifier (table name, column name) and every value goes through a safe handler.

**Values** use psycopg2 parameterization:

```python
cursor.execute(
    "INSERT INTO %s (name, email) VALUES (%s, %s)",
    (table_name, name, email)
)
```

**Identifiers** use `psycopg2.sql`:

```python
from psycopg2 import sql

query = sql.SQL("SELECT * FROM {table} WHERE {col} = %s").format(
    table=sql.Identifier(table_name),
    col=sql.Identifier(column_name),
)
cursor.execute(query, (value,))
```

**Pre-validation** catches malicious input before it ever reaches `sql.Identifier`:

```python
IDENTIFIER_PATTERN = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]{0,62}$')

def validate_identifier(name: str) -> None:
    if not IDENTIFIER_PATTERN.match(name):
        raise SQLSafetyError(f"Invalid identifier: {name!r}")
```

### Credential Management

- All database credentials loaded from `.env` via `python-dotenv`
- No credentials in source code, configuration files, or log output
- `.env` is git-ignored and shipped with a `.env.example` template
- Log lines that include connection strings are scrubbed before writing

### Path Traversal Prevention

User-provided file paths (for extract, migrate, config) are resolved against an allowed base directory:

```python
def safe_path(user_input: str, base: Path) -> Path:
    resolved = (base / user_input).resolve()
    if not str(resolved).startswith(str(base.resolve())):
        raise PathTraversalError(f"Path outside base: {user_input}")
    return resolved
```

This prevents `../../../etc/passwd`-style attacks on any CLI command that accepts a path.

## Testing Strategy

**98 tests** organized by component:

| Module                | Tests | Coverage Focus                              |
| --------------------- | ----- | ------------------------------------------- |
| `test_cli.py`         | 24    | Argument parsing, command routing, exit codes |
| `test_data_cleaner.py`| 41    | Phone/email/name normalization, encoding fixes, regional formats |
| `test_excel_extractor.py` | 33 | Multi-sheet extraction, header handling, error values, edge cases |

**Fixtures** in `conftest.py` provide:

- Synthetic Excel files with controlled row counts, types, and edge cases
- A SQLite in-memory database for import tests (schema-compatible with PostgreSQL for non-PG-specific features)
- Mock configuration with minimal cleaning rules

**Integration tests** run against a real PostgreSQL instance in CI (not included in the unit count) to validate COPY mode, FK ordering, and transaction rollback.

## Configuration Reference

### `cleaning_config.py`

Central configuration for all table-specific behavior:

```python
CLEANING_RULES = {
    '<table_name>': {
        'phone_columns': [...],
        'email_columns': [...],
        'required_fields': [...],
        'column_renames': {'old': 'new'},
        'drop_columns': [...],
        'encoding_fixes': bool,
    }
}

IMPORT_ORDER = ['master_1', 'master_2', 'dependent_1', ...]

FK_RELATIONSHIPS = {
    'child_table': {
        'parent_table': 'parent_id_column'
    }
}
```

### `connection_config.py`

Database connection profiles. All values sourced from environment variables — this file contains only structural defaults.

### `.env` format

```env
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=mydb
DB_USER=postgres
DB_PASSWORD=<your_password>

# Optional: direct and pooled URLs for Supabase
DATABASE_URL_POOLING=postgresql://...
DATABASE_URL_DIRECT=postgresql://...
```

## Non-Goals

To keep the scope tight, Migratex intentionally does *not*:

- Write to non-PostgreSQL targets (MySQL, SQL Server, MongoDB)
- Provide a GUI or web dashboard (CLI only)
- Orchestrate scheduled runs (use cron, Airflow, or Prefect for scheduling)
- Perform real-time streaming ingestion (batch only)
- Manage schema migrations on the target database (use Alembic or Flyway)

These are deliberate boundaries. A tool that does one thing well beats a tool that does everything poorly.

---

*For the product overview, see the [README](../README.md). For licensing and contact, see the README's Licensing section.*
