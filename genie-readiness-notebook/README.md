# Genie Ontology Readiness Assessment

A single Databricks notebook that tells you how ready one or more Unity Catalog catalogs are for Genie, and what to fix first. It is the notebook edition of the Genie Ontology Readiness app (`databricks-solutions/genie-ontology-readiness`), for workspaces that can't run Databricks Apps.

It reads metadata only, never table data, and gives two independent scores:

| Score | Question it answers | Built from |
| --- | --- | --- |
| **Coverage** | Do the tables people rely on have the foundations Genie needs? | 7 weighted pillars, scored on a focus set of gold and most-used tables |
| **Semantic quality** | Is that documentation good enough for Genie to answer correctly? | 4 checks: description quality, metric consistency, column clarity, business vocabulary |

A catalog can score well on coverage (everything has a comment) and poorly on quality (the comments don't say what a row is, what currency an amount is in, or which of three "revenue" measures to use). The two scores are reported side by side for that reason.

## Quick start

1. Import or clone this folder into the workspace, and attach the notebook to serverless or any UC-enabled cluster (DBR 13.3+). There is nothing to install.
2. Click **Run all** once. The widgets appear and the run stops at a clear message asking for a catalog.
3. Enter one or more catalogs in **01. Catalog(s) to assess**, optionally set an output target (widgets 11-14), and click **Run all** again.

The report renders inline at the end. A run usually takes a few minutes; inspecting Genie Agents through the API is the slowest step.

## What you get

* **Headline:** coverage score and readiness stage, semantic quality score, and the focus set the scores are based on.
* **Top gaps:** the most important fixes on each axis, in priority order, each tagged with its pillar or check.
* **Detail cards:** one per pillar and check, with the signals behind the score, the gaps, and notes on anything that couldn't be measured.
* **Drill-downs:** the focus tables with their flags, plus per-pillar tables. These include the lowest-rated descriptions with a suggested fix, conflicting measure definitions, unclear columns per table, and uncovered business terms with an example question.
* **Saved outputs** (optional): Delta tables for tracking runs over time, plus JSON, Markdown, HTML and CSV files.

The Markdown or JSON report can be pasted into any assistant to draft a remediation plan.

## What it reads, writes and sends

* **Reads:** `<catalog>.information_schema`, tags and constraints, metric-view definitions, and Genie Agent settings. If enabled, it also reads `system.access.audit`, `system.access.table_lineage` and `system.query.history`. It never reads table rows.
* **Writes:** nothing, except to the output target you choose.
* **Sends to a model** (only when `use_fmapi = true`): through `ai_query`, to the endpoint in `llm_endpoint`. This means the names and descriptions of sampled tables and metric views, up to 30 column names, types and comments each, and for the vocabulary check, Genie conversation titles plus the words that didn't match. Conversation titles are users' questions and can contain business context. Set `use_fmapi = false` to send nothing to any model.

## Requirements

* Any UC-enabled compute: serverless, or a classic cluster on DBR 13.3+. No libraries to install.
* `BROWSE` (or `USE CATALOG` + `USE SCHEMA`) on each catalog you assess.
* Optional, for the activity signals: `SELECT` on `system.access`. Add `system.query` too if you set `adoption_scope = workspace`.
* Optional, for Genie `api` mode and the vocabulary check: `CAN_EDIT` on the Genie spaces you want inspected. Spaces you can't inspect are counted and reported.

Each part degrades on its own: anything you can't access is reported as unavailable, and its weight is re-spread, rather than failing the run.

| Feature | Permission needed | Without it |
| --- | --- | --- |
| Core scan (required) | `BROWSE`, or `USE CATALOG` + `USE SCHEMA`, on each catalog | That catalog is skipped and listed as unreadable |
| Business-user access check | Visibility of grants (catalog owner or `MANAGE`) | The access signal is skipped and UC Foundation re-weighted |
| Usage, adoption, top-used tables | `SELECT` on `system.access` (add `system.query` for `adoption_scope = workspace`) | Adoption is unavailable; the focus set uses the gold layer only |
| Genie Agents (API) | `CAN_EDIT` on the Genie spaces | Those spaces are counted but not inspected. If the API can't be used at all, `auto` falls back to the audit log |
| Vocabulary check | Ability to list conversations in those spaces | Vocabulary is scored on metric-view synonyms only |
| Description quality | Query access to a Foundation Model API chat endpoint | Not run; the other quality checks are re-weighted |

### Locked-down workspaces

| Situation | Set |
| --- | --- |
| No system tables | `include_system_tables = false`, `genie_source = api` |
| No Foundation Model API, or LLM calls not allowed | `use_fmapi = false` |
| No Genie access | `genie_source = skip` |
| Can't write anywhere | `output_mode = display_only` (the report still renders, and the JSON can be copied) |

## Reading the results

**Coverage stage** (from the coverage score):

| Score | Stage | What it means |
| --- | --- | --- |
| 0-34 | Foundation building | Establish Unity Catalog governance and a curated gold layer as the base for Genie |
| 35-54 | Core foundation in place | Core governance is in place; strengthen metadata and the semantic layer next |
| 55-71 | Semantics and Genie forming | Metadata and a semantic layer are forming; curate and tune Genie Agents |
| 72-84 | Curated and validating | Genie Agents are curated; validate accuracy and onboard business users |
| 85-100 | Ontology-ready | Mature semantics, domains and adoption |

**Level** (for every pillar, check and both headline scores): 0 Absent, 1-39 Initial, 40-64 Developing, 65-84 Established, 85-100 Optimized.

A few reading tips:

* **"Strong score, open gaps"** marks a pillar that scores 65+ but still has gaps worth closing. A high pillar score is not the same as finished.
* **Unavailable or Not run** means the signal couldn't be measured or was switched off. It is excluded, not scored as zero; the card and the notes say why.
* **Start with the top gaps lists.** They are ordered by impact (low score, high weight), and most have a matching drill-down table that names the specific tables, columns or measures.
* **LLM-based results vary slightly between runs** (typically a point or two on description quality). Compare trends across runs rather than single-point differences.

## How coverage is scored

Coverage measures how much of a **focus set** of tables has each capability, not whether it exists somewhere in the catalog. The focus set is:

* the **gold layer**: tables matching `gold_schema_pattern` or `gold_table_pattern`, tables tagged with a layer/tier tag of gold, curated, mart, semantic, presentation or platinum, or certified tables;
* plus the **25 tables used by the most distinct people** over 90 days (when system tables are on).

Schemas matching `exclude_schema_pattern` (scratch, dev, staging, bronze, and so on) are left out and counted as clutter. If no focus tables are found, all in-scope tables are used, and the report says so.

| Pillar | Weight | What is scored |
| --- | --- | --- |
| Unity Catalog Foundation | 15 | Managed share, freshness of focus tables (`stale_days`), clutter (excluded schemas, tmp/backup names, duplicate names), SELECT access for business users |
| Metadata Richness | 22 | Meaningful table and column descriptions on focus tables. Placeholders, comments under 20 (table) or 8 (column) characters, and comments that only repeat the name don't count |
| Relationships & Modeling | 12 | Gold layer present, % of gold tables with a primary key, % taking part in a foreign key (as child or parent) |
| Metrics | 20 | Metric views present and how many, their descriptions, comments on dimensions and measures, % queried by people |
| Genie Agents | 16 | Existence, activity, curation, benchmarks, overly broad agents (over 25 tables), use of gold tables and metric views, thumbs up/down |
| Domains & Stewardship | 10 | % of focus tables in a domain (table, schema or catalog tag), stewarded (group/service-principal owner or steward tag), certified, tagged; deprecated tables still in use |
| Adoption & Activity | 5 | People only (identities with an email); service principals are reported but not scored; % of focus tables read by people |

When a signal can't be read, its weight is spread across that pillar's other signals, and a note says so.

## How semantic quality is scored

| Check | Weight | What it looks for |
| --- | --- | --- |
| Description quality | 30 | An LLM scores up to `llm_sample_size` described focus tables and metric views (most-used first) on business meaning, grain (what one row is), units and currency, caveats, and column comments. Each gets one suggested fix |
| Metric consistency | 25 | From the metric-view definitions: the same measure name defined with a different source or formula, the same formula under different names, and generic names like `value` or `total` |
| Column clarity | 25 | On focus-table columns: cryptic or generic names with no explanation, code/flag/status columns whose values aren't explained, amount/price/duration columns with no unit or currency, and the same column name with different data types across tables |
| Business vocabulary | 20 | Words from Genie conversation titles that don't appear in any table or column name, comment or metric-view synonym; plus the share of metric-view fields with synonyms. IDs, codes and month/day names are set aside by rule; with FMAPI on, an LLM also sets aside other data values and everyday words |

Objects with no description at all are counted on the coverage axis (Metadata Richness), not here, so the two scores don't double-penalize.

## Parameters

**Scope**

| # | Widget | Default | Meaning |
| --- | --- | --- | --- |
| 01 | catalogs | (required) | Comma-separated catalog names |
| 06 | business_principals | (blank) | Business user groups, comma-separated, for the access check. Blank = any non-owner |
| 07 | exclude_schema_pattern | tmp, dev, staging, bronze, ... | Regex for schemas left out of scoring and counted as clutter |
| 08 | gold_schema_pattern | gold, mart, analytics, ... | Regex for gold-layer schema names |
| 09 | gold_table_pattern | `gold_`, `mart_`, `dim_`, `fact_` | Regex for gold-layer table names |
| 10 | stale_days | 90 | Focus tables not changed in this many days count as stale |

**Activity and Genie**

| # | Widget | Default | Meaning |
| --- | --- | --- | --- |
| 02 | include_system_tables | true | Use system tables for usage, adoption and Genie activity |
| 03 | genie_source | auto | `auto` = API (catalog-scoped), falling back to the audit log. `api` = only Genie spaces using this catalog. `audit` = all active spaces in the workspace. `skip` = leave Genie out |
| 04 | adoption_scope | catalog | `catalog` = people and events on this catalog (lineage). `workspace` = workspace-wide audit and query history |
| 05 | lookback_days | 30 | Activity window for Genie and adoption |
| 15 | max_genie_spaces | 200 | Maximum Genie spaces inspected through the API |

**Semantic quality**

| # | Widget | Default | Meaning |
| --- | --- | --- | --- |
| 16 | semantic_quality | true | Run the semantic quality axis |
| 17 | use_fmapi | true | Allow `ai_query` calls. False = no model calls at all |
| 18 | llm_endpoint | databricks-gemini-3-8-flash | Chat endpoint for the LLM checks. If it can't be queried, a few common hosted models are tried |
| 19 | llm_sample_size | 40 | Tables and metric views scored by the description rubric |
| 20 | max_genie_conversations | 300 | Genie conversation titles read for the vocabulary check |

**Output**

| # | Widget | Default | Meaning |
| --- | --- | --- | --- |
| 11 | output_mode | auto | `auto`, `table`, `volume`, `workspace` or `display_only` |
| 12 | output_schema | (blank) | `catalog.schema` for the Delta tables |
| 13 | output_volume_path | (blank) | `/Volumes/<catalog>/<schema>/<volume>[/subdir]` for the report files |
| 14 | output_workspace_dir | `~/genie_readiness_runs` | Workspace folder for the report files |

In `auto` mode the notebook writes to every target you configured. If none is configured, or all fail, it writes to your workspace home folder. If nothing can be written, it prints the report JSON so it can be copied. Invalid volume paths are rejected rather than written to local disk.

## Outputs

**Delta tables** (appended, one `run_id` per run), in `output_schema`:

| Table | One row per | Use it for |
| --- | --- | --- |
| `genie_readiness_runs` | Run | Coverage and semantic quality scores over time, parameters, and the full report JSON |
| `genie_readiness_pillars` | Pillar or check (`axis` = coverage or quality) | Trends per pillar, signals and gaps |
| `genie_readiness_drilldown` | Drill-down row (JSON) | The specific tables, columns, measures and terms behind each gap |
| `genie_readiness_tables` | Table in scope | Every flag per table: focus, gold, described, keyed, domain, certified, fresh, usage |

**Files**, under `<target>/<run_id>/`: `report.html` (the visual report, shareable as-is), `report.md`, `report.json`, `drilldown.csv` and `tables.csv`.

To track progress, schedule the notebook as a job with an `output_schema`, then chart `overall_score` and `semantic_quality_score` from `genie_readiness_runs` by `assessed_at`.

## Customizing

The heuristics are plain constants at the top of two cells, so they can be adapted to a customer's naming conventions without touching the logic:

* **Pillar definitions cell:** domain, steward, certification and layer tag keys; gold tag values; junk-table name pattern; placeholder comments; minimum comment lengths; how many top-used tables join the focus set; the "broad Genie Agent" threshold.
* **Semantic quality cell:** known abbreviations (`KNOWN_ABBR`), generic column and measure names, code/flag and measure column patterns, unit words (`UNIT_TOKENS`), vocabulary stopwords, and the LLM rubric and vocabulary prompts.

## Notebook layout

| Cells | Purpose |
| --- | --- |
| 1-3 | About, widgets, parameter validation |
| 4-5 | Pillar definitions and scoring rules; SQL, access and source helpers |
| 6 | Table inventory: focus set, quality flags, access, ownership, usage |
| 7-9 | Probes: coverage pillars 1-4, pillars 5-7, semantic quality checks |
| 10-11 | Run coverage; run semantic quality |
| 12-13 | Report; drill-downs |
| 14 | Persist outputs |

## Limitations

* Name and vocabulary heuristics assume English naming. Adjust the word lists for other languages.
* Metric consistency compares metric-view measures only; SQL expressions inside Genie Agents are not yet included.
* The vocabulary check uses conversation titles (usually the opening question), not every follow-up message.
* The business-user access check needs grant visibility (catalog owner or `MANAGE`); otherwise it is skipped.
* In `audit` mode, Genie signals are workspace-wide rather than scoped to the catalog.
