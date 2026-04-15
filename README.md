# Migratex — Production-Grade ETL CLI Tool

[![Version](https://img.shields.io/badge/version-0.5.0-blue.svg)](#)
[![Python](https://img.shields.io/badge/python-3.8+-3776AB.svg?logo=python&logoColor=white)](https://www.python.org/)
[![Status](https://img.shields.io/badge/status-production--ready-success.svg)](#)
[![Tests](https://img.shields.io/badge/tests-98-brightgreen.svg)](#)
[![License](https://img.shields.io/badge/license-Proprietary-red.svg)](LICENSE)

**Tech Stack:**
![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?logo=pandas&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3FCF8E?logo=supabase&logoColor=white)
![Click](https://img.shields.io/badge/Click-000000?logo=python&logoColor=white)

---

> **This is a product showcase repository.** It contains documentation, architecture, and usage examples for Migratex, a proprietary ETL CLI tool developed by [ByteUp LLC](https://byteup.co). The source code is not included. For licensing inquiries, see [Licensing](#licensing) below.

## About

Migratex is a command-line ETL tool built in Python for extracting data from Excel workbooks, transforming it through configurable cleaning pipelines, and loading it into PostgreSQL databases including Supabase-hosted instances.

Originally developed as the migration framework for a client project that processed **247,709 records from 6 Excel files across 99 sheets**, it was later refactored into a standalone reusable product with a proper CLI interface, test suite, and documentation.

## Core Capabilities

### Extract

Reads Excel workbooks and converts each sheet to clean CSV files.

- Multi-sheet extraction with automatic naming
- Header normalization to `snake_case`
- Duplicate row removal
- Empty row and column cleanup
- Excel formula error handling (`#REF!`, `#N/A`, etc.)
- High-null column detection and removal

### Transform

Configurable data cleaning engine with multi-region support.

- **Phone normalization** across 6 countries (US, CO, MX, ES, AR, BR)
- Email validation and normalization
- Name splitting (full name to first/last)
- Spanish character encoding fixes
- Foreign key validation against the target database
- Per-table cleaning rules defined in configuration

### Load

Two import modes for different workload profiles.

- **Fast COPY mode** — PostgreSQL `COPY` for bulk imports (10–50× faster than standard `INSERT`)
- **Standard INSERT mode** — `ON CONFLICT` handling for upsert operations
- Batch processing with configurable batch size
- Transaction management with rollback support
- Pre-import and post-import validation

## Architecture

> For a full technical deep-dive — pipeline phases, component contracts, error categories, SQL safety implementation, and testing strategy — see [**docs/ARCHITECTURE.md**](docs/ARCHITECTURE.md). A [PDF version](docs/Migratex_Showcase.pdf) of the product brief is also included.

Modular pipeline architecture with clear separation of concerns.

- **CLI layer** handles argument parsing and command routing
- **Core layer** contains ETL pipeline components following a base component pattern — each component implements an `execute()` method returning status, statistics, and error information
- **Utility library layer** provides reusable functions for data cleaning, SQL safety, and schema validation
- **Configuration layer** centralizes cleaning rules, phone regions, and connection management

All components are configuration-driven through a single cleaning config file. Adding support for a new data source requires configuration changes only — no code modifications.

## Tech Stack

| Category        | Technologies                                         |
| --------------- | ---------------------------------------------------- |
| Language        | Python 3.8+                                          |
| Data Processing | Pandas, openpyxl                                     |
| Database        | PostgreSQL (psycopg2), Supabase                      |
| CLI Framework   | Click                                                |
| Configuration   | python-dotenv, `cleaning_config.py`                  |
| Testing         | pytest (98 tests)                                    |
| Security        | `psycopg2.sql` identifier escaping, parameterized queries |

## Engineering Highlights

**SQL Injection Prevention.** All dynamic SQL uses the `psycopg2.sql` module for identifier escaping. No string interpolation in any database query. User-provided table and column names are validated against a strict pattern before use.

**Configuration-Driven Cleaning.** Table-specific cleaning rules, validation patterns, column rename mappings, foreign key relationships, and import ordering are all defined in a single configuration file. Supporting a new data source requires configuration changes, not code changes.

**FK Dependency Ordering.** Import order respects foreign key dependencies automatically. Master data (locations, categories, vendors) imports first, followed by dependent tables (products, customers), then transactional data (orders, order items). Prevents integrity constraint violations during load.

**Dual Import Modes.** PostgreSQL `COPY` for maximum throughput on clean data (handles 100K+ rows efficiently), standard `INSERT` with `ON CONFLICT` for incremental updates and upsert operations. Mode selection is automatic or user-specified.

**Transaction Safety.** All import operations run within transactions. If any batch fails, the entire operation rolls back. No partial data states in the target database.

## By the Numbers

| Metric                  | Value                                            |
| ----------------------- | ------------------------------------------------ |
| Test Coverage           | 98 tests (CLI, data cleaning, extraction)        |
| Phone Regions Supported | 6 (US, CO, MX, ES, AR, BR)                       |
| Largest Production Run  | 247,709 records from 99 Excel sheets             |
| Import Success Rate     | 99.96% on production migration                   |
| Import Speed (COPY)     | ~3,000 rows/minute                               |
| Default Batch Size      | 1,000 rows per transaction                       |
| Core Components         | 10 (extractor, cleaner, importer, validators, reporter) |

## CLI Reference

The following examples illustrate the CLI interface as used in production.

### Extract Excel to CSV

```bash
# Extract single file
migratex extract ./data/inventory.xlsx

# Extract all Excel files in a directory
migratex extract ./data/

# Custom output directory
migratex extract ./data/inventory.xlsx -o ./output/

# Extract specific sheets only
migratex extract ./data/inventory.xlsx --sheets "Items,Vendors,Categories"

# Skip data cleaning
migratex extract ./data/inventory.xlsx --no-clean

# Set minimum rows threshold
migratex extract ./data/inventory.xlsx --min-rows 5
```

### Database Operations

```bash
# Check database connection
migratex db:status

# List all tables with row counts
migratex db:tables

# Show table schema
migratex db:schema products

# Validate database health
migratex db:validate
```

### Migrate CSV to Database

```bash
# Migrate single CSV to a specific table
migratex migrate ./output/items.csv --table products

# Auto-detect table names from filenames
migratex migrate ./output/ --auto

# Dry run (validate without importing)
migratex migrate ./output/items.csv --table products --dry-run

# Fast mode using PostgreSQL COPY
migratex migrate ./output/ --auto --mode fast
```

### Phone Region Support

```bash
# US phone format (default)
migratex extract ./data/customers.xlsx

# Mexican phone format
migratex --region mx extract ./data/customers.xlsx

# Colombian phone format
migratex --region co extract ./data/customers.xlsx
```

**Supported regions:**

| Code | Country   | Digits | Example          |
| ---- | --------- | ------ | ---------------- |
| us   | USA       | 10     | (555) 123-4567   |
| mx   | Mexico    | 10     | 55 1234 5678     |
| co   | Colombia  | 10     | 300 123 4567     |
| es   | Spain     | 9      | 612 34 56 78     |
| ar   | Argentina | 10     | 11 1234-5678     |
| br   | Brazil    | 11     | (11) 91234-5678  |

### Command Reference

```
migratex --help                    Show all commands
migratex --version                 Show version
migratex --region REGION           Set phone region (us, mx, co, es, ar, br)
migratex -q, --quiet               Suppress output except errors

migratex extract --help            Extract command help
migratex db:status --help          Database status help
migratex migrate --help            Migration help
```

## Sample Output

Extraction report produced by a typical run:

```
======================================================================
MIGRATEX EXTRACTION
======================================================================
Source: inventory.xlsx
Output: migratex_output/inventory/
Clean:  Yes
======================================================================

SHEETS FROM: inventory.xlsx
============================
+-------------------+--------+---------+--------+------------------+
|    Sheet Name     |  Rows  | Columns | Status |   Output File    |
+-------------------+--------+---------+--------+------------------+
| Items             | 10,451 |      24 |   OK   | Items.csv        |
| Vendors           |     30 |       7 |   OK   | Vendors.csv      |
| Categories        |     21 |       2 |   OK   | Categories.csv   |
+-------------------+--------+---------+--------+------------------+

+-------------------------------+
|       EXTRACTION SUMMARY      |
+-------------------------------+
| Total Sheets       :      3   |
| Extracted          :      3   |
| Total Rows         : 10,502   |
| Duplicates Removed :    150   |
+-------------------------------+
```

## Project Structure

```
migratex/
├── migratex/
│   ├── __init__.py           # Package init, version
│   ├── cli.py                # Main CLI entry point
│   ├── commands/             # CLI command handlers
│   │   ├── extract.py        # Extract command
│   │   ├── db.py             # Database commands
│   │   └── migrate.py        # Migrate command
│   ├── core/                 # Core ETL components
│   │   ├── excel_extractor.py
│   │   ├── migration_framework.py
│   │   ├── database_validator.py
│   │   ├── data_cleaner_component.py
│   │   ├── data_importer_component.py
│   │   ├── schema_validator_component.py
│   │   ├── fast_importer.py
│   │   └── report_generator.py
│   ├── lib/                  # Utility libraries
│   │   ├── data_cleaner.py        # Data cleaning engine
│   │   ├── sql_utils.py           # SQL injection protection
│   │   ├── pre_import_validator.py
│   │   └── schema_validator.py
│   └── config/               # Configuration
│       ├── cleaning_config.py
│       └── connection_config.py
├── tests/                    # Test suite (98 tests)
│   ├── conftest.py
│   ├── test_cli.py
│   ├── test_data_cleaner.py
│   └── test_excel_extractor.py
├── docs/                     # Architecture documentation
└── pyproject.toml
```

## Error Handling

Structured error handling across the pipeline. SQL injection attempts raise `SQLSafetyError`. Database connection issues trigger `OperationalError` with automatic retry logic. Foreign key violations during import are logged and skipped without stopping the batch, with a full report generated at completion. File not found and invalid parameter errors are caught at the CLI layer with clear user-facing messages.

All errors are categorized with specific recovery actions:

- Connection errors retry with exponential backoff
- FK violations log and continue
- Data type errors attempt conversion
- Duplicate keys are silently skipped

## Security

- All SQL queries use parameterized queries or `psycopg2.sql` for identifier escaping
- Database credentials are loaded exclusively from environment variables via `python-dotenv`
- No credentials appear in source code or log output
- User-provided file paths are validated against allowed base directories to prevent path traversal
- All dynamic table and column names are validated against a strict alphanumeric pattern before use in any query

## Origin

Migratex started as a custom migration framework built for a retail business in Mexico migrating from a legacy Google Sheets and Zapier infrastructure to PostgreSQL on Supabase. The original project processed 247,709 records from 6 Excel files containing 99 sheets, with database triggers, Edge Functions, and Shopify webhook integration. It delivered with a 99.96% success rate.

After delivering the client project, the framework was refactored into a standalone CLI tool with a proper package structure, 98 tests, multi-region support, and documentation. Migratex is now available as a reusable tool for any Excel-to-PostgreSQL migration workflow.

## Skills Demonstrated

| Area               | Details                                                                 |
| ------------------ | ----------------------------------------------------------------------- |
| Python Development | CLI tool design, package structure, OOP with base component pattern    |
| Data Engineering   | ETL pipeline architecture, data cleaning, validation, transformation   |
| Database           | PostgreSQL, Supabase, FK integrity, indexing, bulk operations          |
| Testing            | pytest with fixtures, 98 tests, unit and integration coverage          |
| Security           | SQL injection prevention, credential management, path validation       |
| Architecture       | Modular pipeline design, configuration-driven processing, SoC          |
| Documentation      | Architecture docs, README, inline documentation                        |

## Licensing

Migratex is proprietary software developed by ByteUp LLC. The source code is not included in this repository and is not available under any open-source license.

This repository contains documentation and marketing materials only. The content (README, architecture documentation, diagrams) is provided for informational and portfolio purposes.

For commercial licensing, custom migration projects, or integration inquiries, contact **contact@byteup.co**.

## About ByteUp

Migratex is a product of **[ByteUp LLC](https://byteup.co)**, a software studio building SaaS products for underserved markets and delivering contract development for clients who value clean architecture and reliable code.

### ByteUp products

| Product         | Status             | Link                                                |
| --------------- | ------------------ | --------------------------------------------------- |
| **Migratex**    | Production-ready   | You are here                                        |
| **TroveTrends** | Live               | [trovetrends.com](https://trovetrends.com)          |
| **FixDoc**      | In development     | `fixdoc.co` *(launching soon)* — live demo available on request |

### Author

Built and maintained by **Ivan Duarte**, Full-Stack Software Engineer and founder of ByteUp LLC. Based in Bogotá, Colombia.

- Company: [byteup.co](https://byteup.co) · contact@byteup.co
- GitHub: [@ivansing](https://github.com/ivansing)
- LinkedIn: [lance-dev](https://linkedin.com/in/lance-dev)

---

*Document Version: 1.0 · Product: Migratex · Status: Production-Ready*
