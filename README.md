# Semantic Extraction Utility

**v0.6.0** — A Cortex Code skill for extracting semantic definitions from **Tableau**, **Looker**, **Power BI**, **Denodo**, and **SAP Business Objects**, producing professional assessment reports, and converting them to **Snowflake Semantic View YAML**.

## What It Does

This utility extracts, classifies, and reports on semantic definitions from BI tools:

1. **Parse** source files — `.pbit`, `.pbip`, `.bim` (Power BI), `.lkml` (Looker), `.twb`/`.tds`/`.twbx` (Tableau), `.vql` (Denodo), `.json` (Business Objects)
2. **Classify** every field as `simple` (auto-convertible), `needs_translation` (known SQL equivalent), or `manual_required` (requires human attention)
3. **Analyze** dashboard pages for overlap, purpose, and consolidation opportunities
4. **Generate** a branded HTML executive report + 5-tab Excel workbook — the customer-facing deliverable
5. **Produce** Snowflake Semantic View YAML ready for deployment
6. **Compare** inventories across runs or source types with the built-in diff tool

### Relationship to Snowflake Semantic View Autopilot

Snowflake's Semantic View Autopilot can ingest Tableau `.twb`/`.tds`/`.twbx` files directly in Snowsight — but it is manual (one workbook at a time), does not support Looker, Power BI, Denodo, or Business Objects, and does not produce an audit/mapping artifact.

This skill **complements** Autopilot:

- **Audit everything first**, then qualify what needs to migrate
- **Looker/LookML** — generates YAML directly (no Autopilot path exists)
- **Power BI / DAX** — generates YAML directly (no Autopilot path exists)
- **Denodo / VQL** — generates YAML directly (no Autopilot path exists); infers dim/measure via heuristics
- **SAP Business Objects** — generates YAML directly (no Autopilot path exists); leverages native Dimension/Measure types
- **Tableau at scale** — programmatic extraction across hundreds of workbooks
- **Mapping file** — the Excel deliverable that Autopilot doesn't produce

## Installation

### Option A: Install from GitHub (Recommended)

```bash
# Clone into your global skills directory
git clone https://github.com/sfc-gh-HMAHAKALI/SemanticExtratcor.git ~/.snowflake/cortex/skills/semantic-extraction

# Install Python dependencies
pip install -r ~/.snowflake/cortex/skills/semantic-extraction/requirements.txt
```

### Option B: Copy to Global Skills

```bash
# If you received the skill as a zip archive
unzip semantic-extraction.zip -d semantic-extraction

# Copy to global skills directory
cp -r semantic-extraction ~/.snowflake/cortex/skills/
```

### Option C: Copy to Project Skills

```bash
# For use in a specific project only
cp -r semantic-extraction .cortex/skills/
```

### Option D: Add via skills.json

Add to `~/.snowflake/cortex/skills.json`:

```json
{
  "paths": [
    "/path/to/semantic-extraction"
  ]
}
```

### Post-Install: Dependencies

After placing the skill, install Python dependencies:

```bash
cd ~/.snowflake/cortex/skills/semantic-extraction
pip install -r requirements.txt
```

This installs `pbixray` (for .pbix parsing), `lkml` (for Looker), and other required packages. All packages use prebuilt wheels — no C compiler or build tools are needed.

### Verify Installation

In a Cortex Code session:

```
> $$
```

You should see `semantic-extraction` in the skill listing. Then invoke with:

```
> $semantic-extraction
```

## Prerequisites

| Requirement | Purpose | Install |
|-------------|---------|---------|
| Python 3.11+ | Runs extraction scripts inline | — |
| `openpyxl` | Generates Excel mapping file | `pip install openpyxl` |
| `pyyaml` | YAML generation | `pip install pyyaml` |
| `lkml` | Parses LookML files (Looker only) | `pip install lkml` |
| `pbixray` | Parses .pbix files (Power BI only) | `pip install pbixray` |
| `jaydebeapi` | JDBC access to Denodo VDP (Denodo only) | `pip install jaydebeapi` + Denodo JDBC driver JAR |
| Java runtime | Required for BO universe extractor (Business Objects only) | JDK 8+ |
| Snowflake connection | Target resolution + deployment | Configured in `~/.snowflake/connections.toml` |

Snowflake connection is optional — the skill can generate the mapping file and YAML without one.

## Inputs

| Source | Input Type | What to Provide |
|--------|-----------|-----------------|
| **Tableau** (files) | `.twb`, `.tds`, `.twbx` | File paths on disk |
| **Tableau** (API) | GraphQL | Server URL + REST API token |
| **Looker** (files) | `.lkml` project | Full project directory path (model + view + manifest files) |
| **Looker** (API) | REST API | Instance URL + client_id/client_secret |
| **Power BI** (files) | `.pbix`, `.pbit`, `.pbip`, `.bim` | File paths or project folder |
| **Power BI** (API) | REST API / XMLA | Tenant ID + service principal credentials, or XMLA endpoint |
| **Denodo** (VQL export) | `.vql` text files | VQL export from `DESC VQL DATABASE` |
| **Denodo** (JDBC) | JDBC connection | VDP host, port (9999), database, credentials + JDBC driver JAR |
| **Denodo** (API) | Data Catalog REST API | Catalog URL + credentials (Basic/Kerberos/OAuth) |
| **Business Objects** (files) | `.unx`, `.unv`, JSON export | JSON from starschema extractor (recommended), or universe files |
| **Business Objects** (SDK) | Semantic Layer Java SDK | CMS server + credentials (requires BI Platform installation) |
| **Business Objects** (API) | REST API | BO REST base URL + credentials (inventory only — no SELECT expressions) |

## Outputs

| Artifact | Format | Description |
|----------|--------|-------------|
| **HTML Executive Report** | `.html` | Snowflake-branded assessment with executive summary, complexity breakdown, and action items |
| **Excel Workbook** | `.xlsx` | 5-tab professional workbook — the customer-facing governance deliverable |
| **Extraction Inventory** | `.json` | Structured inventory for programmatic use and cross-run comparison |
| **Semantic View YAML** | `.yaml` | Ready for `SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML` (via `generate` command) |

### HTML Executive Report

Three-section layout: **Executive Summary** (field counts, complexity funnel, auto-convert %, overlap callout, DAX remapping summary) → **Inventory Overview** (per-table breakdowns, relationship map) → **Items to Resolve** (flagged fields with effort estimates and expression excerpts). Light Snowflake branding, printable to PDF.

### Excel Workbook Tabs

| Tab | Contents |
|-----|----------|
| **Summary** | Extraction stats, complexity breakdown, conversion readiness score, usage instructions |
| **Full Inventory** | Every dimension, fact, and metric with source table, expression, complexity, effort |
| **Items to Resolve** | Flagged fields only — grouped by type (complex_dax, translatable_dax, inactive_relationship), with DAX patterns, categories, expression excerpts, and recommended actions |
| **Dashboard Inventory & Overlap** | Per-page purpose inference, field counts, pairwise Jaccard similarity, consolidation candidates (color-coded: red ≥80%, amber ≥50%) |
| **Reference** | Field type definitions, complexity tier explanations, DAX pattern glossary |

## CLI Usage

The utility runs as a Python module with 8 commands:

```bash
cd ~/.snowflake/cortex/skills/semantic-extraction

# Parse a source file or directory into inventory JSON
python3 -m modules.cli parse <path> --type powerbi --output inventory.json

# Generate HTML report + Excel workbook from inventory
python3 -m modules.cli report inventory.json --output ./output --customer "Acme Corp"

# Compare two inventory files
python3 -m modules.cli compare old_inventory.json new_inventory.json

# Generate Semantic View YAML from inventory
python3 -m modules.cli generate inventory.json --output ./yaml

# Generate YAML from a curated Excel workbook
python3 -m modules.cli generate-from-workbook workbook.xlsx --output ./yaml

# Other commands
python3 -m modules.cli crawl <dir> --type tableau    # Discover source files
python3 -m modules.cli classify inventory.json        # Classify complexity
python3 -m modules.cli test-connection --type looker   # Test API credentials
```

### Report Command Options

```
report <input.json> [options]
  --output DIR           Output directory (default: auto-named ~/Downloads/)
  --customer NAME        Customer name for report header
  --source-label TEXT    Source description (e.g. "Sharp LookML - 31 Projects")
  --ai                  Enable AI-powered rationalization via Snowflake Cortex
  --ai-model MODEL      Cortex LLM model (default: llama3.1-70b)
```

### Supported Parse Types

| `--type` | Formats | Notes |
|----------|---------|-------|
| `powerbi` | `.pbit`, `.pbip` folder, `.bim` | Also supports `.pbix` via PBIXRay |
| `looker` | `.lkml` directory | Resolves `extends` and `refinements` |
| `tableau` | `.twb`, `.tds`, `.twbx` | Extracts dashboard field usage |
| `denodo` | `.vql` | Requires `CREATE OR REPLACE VIEW` statements |
| `businessobjects` | `.json` (businessLayer format) | From starschema extractor |

## Interactive Workflow (via SKILL.md)

When invoked as `$semantic-extraction` in Cortex Code, the skill also provides a guided 9-step workflow with checkpoints at each step. The CLI commands above are used under the hood.

## Skill Structure

```
semantic-extraction/
├── README.md                              ← You are here
├── SKILL.md                               ← Guided workflow (invoked by $semantic-extraction)
├── requirements.txt                       ← Python dependencies
├── modules/
│   ├── cli.py                             ← CLI entry point (8 commands)
│   ├── common/                            ← Shared utilities (errors, logging, retry, file crawler)
│   ├── output/
│   │   ├── analysis.py                    ← Narrative engine, overlap analysis, purpose inference
│   │   ├── html_report.py                 ← Branded HTML executive report
│   │   ├── excel_report.py                ← 5-tab Excel workbook
│   │   ├── inventory.py                   ← Inventory normalizer
│   │   ├── yaml_generator.py              ← Semantic View YAML output
│   │   ├── diff.py                        ← Inventory comparison
│   │   └── excel_reader.py                ← Workbook-to-YAML reader
│   ├── adapters/                          ← discovery_context.json adapter
│   ├── powerbi/                           ← PBIT/PBIP/BIM parser, DAX classifier, M resolver
│   ├── looker/                            ← LookML parser, extends resolver, API client
│   ├── tableau/                           ← TWB/TDS/TWBX parser, API client
│   ├── denodo/                            ← VQL parser, JDBC/Catalog clients
│   └── businessobjects/                   ← Universe parser, @function resolver
└── references/
    ├── concept_mapping.md                 ← Tableau/Looker/PBI/Denodo/BO→SF mapping tables
    ├── tableau_extraction.md              ← XML parsing patterns, GraphQL queries
    ├── looker_extraction.md               ← lkml parser patterns, API usage
    ├── powerbi_extraction.md              ← TOM/BIM parsing, DAX, REST API, XMLA
    ├── denodo_extraction.md               ← VQL parsing, JDBC, Data Catalog API
    ├── businessobjects_extraction.md      ← Universe parsing, @function resolution
    └── semantic_view_yaml_spec.md         ← YAML structure, constraints, deployment
```

## Known Limitations

- **Tableau LOD expressions** (`{FIXED}`, `{INCLUDE}`, `{EXCLUDE}`) cannot be auto-converted — flagged for manual SQL
- **Looker Liquid SQL templates** (`{% if ... %}`) cannot be auto-converted — flagged for manual SQL
- **DAX CALCULATE/ALL patterns** (filter context manipulation) cannot be auto-converted — flagged for manual SQL
- **Tableau table calculations** (`RUNNING_SUM`, `LOOKUP`, `WINDOW_*`) are not supported in semantic views
- **Power BI calculated tables** (DAX-generated, e.g. `Calendar`) have no physical source — must create Snowflake views first
- **Denodo has no native dimension/measure** — classification is inferred via heuristics (data type, naming, aggregation, catalog tags). User override JSON recommended for precision.
- **Denodo FLATTEN/NEST/UNNEST** — complex type operations cannot be auto-converted; require Snowflake `LATERAL FLATTEN` views
- **BO .unv files** (Universe Design Tool, legacy) are proprietary binary — cannot be parsed without UDT or the starschema Java extractor
- **BO .unx files** (Information Design Tool) require the SAP Semantic Layer Java SDK or manual IDT export
- **BO @Script macros** (VBScript/JavaScript) have no SQL equivalent — flagged for manual rewrite
- **BO Contexts** (multiple join paths) may require generating separate semantic views per Context
- **BO REST API** provides inventory only — does not expose SELECT expressions or join conditions
- **Semantic views work best with ~50-100 columns** — large BI sources are split into focused views
- **No bulk CREATE SEMANTIC VIEW API** — deployment is sequential, one view at a time
- **Power BI .pbix Xpress9 compression** — `.pbix` data model cannot be unzipped with standard tools. Use `pbi-tools`, `PBIXRay`, or extract from `.pbit`/`.pbip` instead

## Related Skills

- `$semantic-view` — Create, audit, debug, and optimize Snowflake Semantic Views (use after initial deployment for ongoing refinement)
- `$cortex-agent` — Build Cortex Agents that query semantic views

## Reference Materials

The function translation tables, complexity classifiers, and extraction patterns in this skill were validated against the following official documentation:

| Source | Reference | Date Accessed |
|--------|-----------|---------------|
| **Tableau** | [Tableau Functions (by Category)](https://help.tableau.com/current/pro/desktop/en-us/functions_all_categories.htm) — Number, String, Date, Logical, Aggregate, Table Calculation, Type Conversion, Additional (Regex/RAWSQL) functions | March 2026 |
| **Tableau** | [Tableau Table Calculation Functions](https://help.tableau.com/current/pro/desktop/en-us/functions_functions_tablecalculation.htm) — RUNNING_*, WINDOW_*, RANK*, INDEX, FIRST, LAST, SIZE, TOTAL, LOOKUP, PREVIOUS_VALUE, MODEL_*, SCRIPT_* | March 2026 |
| **Looker** | [LookML Measure Types](https://cloud.google.com/looker/docs/reference/param-measure-types) — sum, count, average, median, percentile, list, sum_distinct, average_distinct, median_distinct, percentile_distinct, percent_of_total, percent_of_previous, running_total, period_over_period, number, string, yesno, date | March 2026 |
| **Looker** | [LookML Dimension/Filter/Parameter Types](https://cloud.google.com/looker/docs/reference/param-dimension-filter-parameter-types) — string, number, yesno, tier, bin, case, location, distance, duration, zipcode, date, date_time, time, custom_calendar, unquoted | March 2026 |
| **Looker** | [LookML Explore Parameters](https://cloud.google.com/looker/docs/reference/param-explore) — sql_always_where, always_filter, access_filter, conditionally_filter, sql_always_having, required_access_grants, aggregate_table, symmetric_aggregates | March 2026 |
| **Power BI** | [DAX Reference (Microsoft)](https://learn.microsoft.com/en-us/dax/) — 250+ functions across 13 categories. Also validated against the 1,412-page official PDF (`dax-reference.pdf`) | March 2026 |
| **Snowflake** | [Snowflake SQL Function Reference](https://docs.snowflake.com/en/sql-reference/functions-all) — target function mappings for all translations | March 2026 |
| **Snowflake** | [Semantic Views Documentation](https://docs.snowflake.com/en/sql-reference/sql/create-semantic-view) — YAML spec, SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML, constraints | March 2026 |
| **Denodo** | [Denodo VQL Reference](https://community.denodo.com/docs/html/browse/9.1/vdp/vql/) — VQL syntax, stored procedures (CATALOG_VDP_METADATA_VIEWS, GET_VIEW_COLUMNS, VIEW_DEPENDENCIES), data types, function reference | April 2026 |
| **Denodo** | [Denodo Data Catalog REST API](https://community.denodo.com/docs/html/browse/9.1/en/platform/data-catalog/) — Authentication, synchronization, browse/search endpoints, metadata export | April 2026 |
| **SAP BO** | [SAP BusinessObjects BI Platform SDK](https://help.sap.com/docs/BOBI_PLATFORM) — Semantic Layer Java SDK, RESTful Web Service SDK, universe structure (Classes, Objects, Joins, Contexts) | April 2026 |
| **SAP BO** | [starschema/business-objects-universe-extractor](https://github.com/starschema/business-objects-universe-extractor) — Java tool for extracting .unx/.unv universe metadata to JSON | April 2026 |

## Changelog

### 0.6.0 — April 2026

**Report Layer & Assessment Tooling** — Professional customer-facing output, dashboard intelligence, and multi-source validation.

- **HTML Executive Report** — Snowflake-branded 3-section layout (Executive Summary → Inventory Overview → Items to Resolve) with complexity funnel, auto-convert %, and effort estimates. Printable to PDF.
- **5-Tab Excel Workbook** — Summary, Full Inventory, Items to Resolve, Dashboard Inventory & Overlap, Reference. Color-coded headers, auto-sized columns, conditional formatting for severity.
- **Dashboard overlap analysis** — Pairwise Jaccard similarity between dashboard pages, consolidation candidate detection (red ≥80%, amber ≥50%), moved to Excel tab with brief callout in HTML executive summary.
- **Purpose inference** — `_infer_page_purpose()` generates question-oriented statements from field names and types (e.g., "Tracks cancellations by region and department").
- **3-tier DAX classification** — `simple` (direct SQL equivalent), `needs_translation` (known pattern with SQL mapping), `manual_required` (human attention needed). New `translatable_dax` flag category surfaces the middle tier.
- **DAX semantic interpretation** — Pattern-based categorization of DAX expressions (time intelligence, filter context, iterator, statistical, text, logical) with recommended Snowflake SQL equivalents.
- **AI-powered rationalization** — `--ai` flag on `report` command uses Snowflake Cortex (`llama3.1-70b`) for natural-language field descriptions and migration recommendations.
- **`report` CLI command** — New command: `python3 -m modules.cli report inventory.json --output ./output --customer "Acme Corp"` generates both HTML and Excel in one pass.
- **`compare` CLI command** — New command for diffing two inventory files with added/removed/changed field detection.
- **`generate-from-workbook` CLI command** — Generate Semantic View YAML from a curated Excel workbook (post-review workflow).
- **Multi-source validation** — Tested against 8 real-world projects across Power BI, LookML, and Tableau. Head-to-head comparison with reference implementation showed exact match on core extraction with improvements in flagging granularity and LookML metric coverage.

### 0.5.1 — April 2026

**Code Quality Review** — Remediation from Sr Staff Engineer review.

- Fixed temp directory leak in Tableau TWBX/TDSX extraction
- Extracted shared `retry_request()` helper — centralized retry-with-backoff replacing ~270 lines of duplicate logic
- Removed dead code in Looker topological sort; module-level import cleanup
- Net change: 11 files, +410 / -555 lines

### 0.5.0 — April 2026

**Python Automation Modules** — 34 modules, 10,000+ lines providing programmatic extraction, classification, and YAML generation for all 5 source types.

- CLI entry point (`python3 -m modules.cli`) with `crawl`, `parse`, `classify`, `generate`, `test-connection` commands
- **Tableau** — TWB/TDS/TWBX XML parser, TWBX ZIP extraction, GraphQL API client, LOD/table calc flagging
- **Looker** — LookML project walker, `extends` resolution with topological sort, Liquid template flagging, REST API client
- **Power BI** — Multi-format parser (PBIT/PBIP/BIM/PBIXRay), DAX classifier (~85 patterns), M source resolver, Scanner API client
- **Denodo** — VQL block parser, 5-priority dim/measure inference chain, JDBC + REST clients, view dependency graph
- **SAP Business Objects** — BIAR/JSON parser, @Function resolver (5-level nesting), Context splitting, REST API client
- Unified inventory normalizer, YAML generator with auto-split at 100+ columns
- Structured logging, graceful failure (one bad file never aborts extraction)

### 0.4.0 — March 2026

**Multi-Source Expansion** — Added Power BI, Denodo, and SAP Business Objects support alongside existing Tableau and Looker parsers.

### 0.3.0 — March 2026

**Looker Support** — LookML parsing, extends/refinements resolution, Liquid template flagging, REST API extraction.

### 0.2.0 — March 2026

**Tableau Support** — TWB/TDS/TWBX file parsing, GraphQL API extraction. 9-step guided workflow in SKILL.md.

### 0.1.0 — March 2026

**Initial Release** — Reference files (`concept_mapping.md`, `semantic_view_yaml_spec.md`), SKILL.md workflow skeleton, mapping file concept.
