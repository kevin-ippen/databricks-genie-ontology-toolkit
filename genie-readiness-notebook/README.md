# Genie Ontology Readiness: notebook edition

This is a single-notebook version of the Assess tab from the Genie Ontology Readiness app (`databricks-solutions/genie-ontology-readiness`). It is for workspaces where Databricks Apps aren't available. It scores the catalog(s) you choose on two axes:

* **Coverage**: the 7-pillar scorecard (weights, maturity levels, readiness stages), scored on a focus set of tables.
* **Semantic quality**: whether the documentation is good enough for Genie to answer correctly.

## Contents

* `Genie Ontology Readiness Assessment`: the notebook. Fill in the widgets and click Run all.

## Requirements

* Any UC-enabled compute: serverless, or a classic cluster on DBR 13.3+. No libraries to install.
* `BROWSE` (or `USE CATALOG` + `USE SCHEMA`) on each catalog you assess.
* Optional, for the activity signals: `SELECT` on `system.access`. Add `system.query` too if you set `adoption_scope = workspace`.
* Optional, for Genie `api` mode and the vocabulary check: `CAN_EDIT` on the Genie spaces you want inspected. Spaces you can't inspect are counted and reported.
* Optional, for the LLM checks: permission to query a Foundation Model API chat endpoint through `ai_query`.

## Parameters

| # | Widget | Default | Meaning |
| --- | --- | --- | --- |
| 01 | catalogs | (required) | Comma-separated catalog names |
| 02 | include_system_tables | true | Set to false if system tables are locked down. Adoption becomes unavailable, and Genie needs `api` mode |
| 03 | genie_source | auto | `auto` = API first (catalog-scoped), falling back to the audit log. `api` = only Genie spaces whose tables live in the catalog. `audit` = the app's method (workspace-wide). `skip` = leave Genie out |
| 04 | adoption_scope | catalog | `catalog` = users and events on the catalog, from `table_lineage`. `workspace` = the app's audit + query history method |
| 05 | lookback_days | 30 | Activity window for Genie and adoption |
| 06 | business_principals | | Comma-separated business user groups. Blank = any non-owner |
| 07 | exclude_schema_pattern | tmp/dev/staging/bronze/... | Regex for schemas left out of scoring and counted as clutter |
| 08 | gold_schema_pattern | gold/mart/analytics/... | Regex for gold-layer schema names |
| 09 | gold_table_pattern | `gold_`, `mart_`, `dim_`, `fact_` | Regex for gold-layer table names |
| 10 | stale_days | 90 | Focus tables not changed in this many days count as stale |
| 11 | output_mode | auto | `auto`, `table`, `volume`, `workspace`, or `display_only` |
| 12 | output_schema | | `catalog.schema` for the Delta tables (append) |
| 13 | output_volume_path | | `/Volumes/<catalog>/<schema>/<volume>[/subdir]` for the report files |
| 14 | output_workspace_dir | `~/genie_readiness_runs` | Workspace folder for the same files |
| 15 | max_genie_spaces | 200 | Maximum number of spaces inspected in `api` mode |
| 16 | semantic_quality | true | Run the semantic quality axis |
| 17 | use_fmapi | true | Allow `ai_query` calls. Set to false and the notebook makes no LLM calls |
| 18 | llm_endpoint | databricks-gemini-3-8-flash | Chat endpoint for the LLM checks. If it can't be queried, a few common hosted models are tried |
| 19 | llm_sample_size | 40 | Tables and metric views scored by the description rubric |
| 20 | max_genie_conversations | 300 | Genie conversation titles read for the vocabulary check |

## Outputs

* **Delta tables** in `output_schema`: `genie_readiness_runs` (one row per run, coverage and semantic quality scores), `genie_readiness_pillars` (one row per pillar or check, with an `axis` column), `genie_readiness_drilldown`, and `genie_readiness_tables` (one row per table with all flags).
* **Files** under `<run_id>/`: `report.json`, `report.md`, `report.html`, `drilldown.csv` and `tables.csv`.

In `auto` mode the notebook writes to every target you configured. Volume paths that aren't `/Volumes/<catalog>/<schema>/<volume>` are rejected. If you configured no target, or all of them fail, it writes to your workspace home folder. If nothing can be written, it prints the full report JSON so it can be copied. The report is always shown inline as well.

## How coverage is scored (differs from the app)

The app mostly scores whether a capability exists. This notebook scores how much of a **focus set** of tables it covers. The focus set is the gold layer (found by name pattern, a `layer`-style tag, or certification) plus the 25 tables used by the most distinct people over 90 days. Schemas matching `exclude_schema_pattern` are left out and counted as clutter. If no focus tables are found, all in-scope tables are used, and the report says so.

| Pillar | What is scored |
| --- | --- |
| UC Foundation | Managed share, freshness of focus tables (`stale_days`), clutter (excluded schemas, tmp/backup names, duplicate names), SELECT access for `business_principals` |
| Metadata | Meaningful table and column descriptions on focus tables. Placeholders, comments under 20/8 characters, and name-only comments don't count |
| Relationships | Gold layer present, % of gold tables with a PK, % taking part in an FK (as child or parent) |
| Metrics | Metric views present and how many, descriptions, comments on dimensions and measures, % queried by people |
| Genie Agents | Existence, activity, curation, benchmarks, broad agents (over 25 tables), use of gold tables and metric views, thumbs up/down |
| Domains | % of focus tables in a domain (table, schema, or catalog tag), stewarded (group/SP owner or steward tag), certified, tagged; deprecated tables still in use |
| Adoption | People only (identities with an email); service principals reported but not scored; % of focus tables read by people |

When a signal can't be read (for example, grants aren't visible), its weight is spread across that pillar's other signals, and a note says so. Edit the tag keys and thresholds in the *Pillar definitions* cell.

## Semantic quality (second axis)

This is reported next to the coverage score, not blended into it. Checks that can't run are left out and the rest re-weighted.

| Check | Weight | How it works | Needs |
| --- | --- | --- | --- |
| Description quality | 30 | An LLM rubric (business meaning, grain, units, caveats, column comments) on `llm_sample_size` described focus tables and metric views, most-used first, each with a suggested fix | `use_fmapi = true` and a queryable `llm_endpoint` |
| Metric consistency | 25 | Parses metric-view YAML. Flags the same measure name with a different source or formula, the same formula under different names, and generic names | Read access to the metric views |
| Column clarity | 25 | On focus-table columns: cryptic or generic names with no explanation, code/flag/status columns whose values aren't explained, amount/price/duration columns with no unit or currency, and the same column name with different types across tables | Metadata only |
| Business vocabulary | 20 | Words from Genie conversation titles compared against names, comments and metric-view synonyms. IDs, codes and month/day names are set aside by rule; with FMAPI on, an LLM also sets aside other data values and everyday words. Also measures synonym coverage | Genie API access to conversations |

Set `semantic_quality = false` to skip the whole axis.

Set `use_fmapi = false` for workspaces without Foundation Model API access, or where LLM calls aren't allowed; the notebook then never calls `ai_query`. Description quality shows as *Not run* and its weight is spread over the other three checks. Without the LLM, names, places and everyday words may show up as uncovered vocabulary terms. Heuristic word lists (`KNOWN_ABBR`, `UNIT_TOKENS`, `CODE_COL_RE`, and so on) are at the top of the semantic quality cell.

## Other differences from the app

* Scope is the selected catalog(s), read from `<catalog>.information_schema`. The notebook never reads `system.information_schema`.
* The legacy `hive_metastore` coverage signal is left out, because it is workspace-wide.
* Domains are always scored with the governed-tag proxy, not the native Domains API.
* Genie `api` mode is catalog-scoped. If audit activity can't be read, the activity points are awarded for curation instead. The run notes say when this happened.
* There is no LLM Plan tab. The JSON or Markdown report can be pasted into any assistant to draft a plan.
