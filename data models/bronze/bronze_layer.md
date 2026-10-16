# Bronze Layer

This document describes the bronze layer: what it stores, how data gets into it, what is added to it, and what is deliberately not done to it. It is the reference for anyone loading or reading bronze tables.

**Scope.** What bronze does and how it is loaded. 
## Contents

- [1. Overview](#1-overview)
- [2. Diagram](#2-diagram)
- [3. What bronze does](#3-what-bronze-does)
- [4. Bronze tables](#4-bronze-tables)
- [5. How data gets into bronze](#5-how-data-gets-into-bronze)
- [6. Schema and added columns](#6-schema-and-added-columns)
- [7. Storage and maintenance](#7-storage-and-maintenance)
- [8. Security](#8-security)
- [9. Checks](#9-checks)
- [10. Assumptions and open items](#10-assumptions-and-open-items)

## 1. Overview

Bronze is the first copy of the data inside the lakehouse. Every document exported from MongoDB is stored exactly as received, in Delta format, and is never changed afterwards. Nothing is cleaned, typed, renamed or removed. If anything goes wrong downstream, bronze is the place to replay from.

| Topic | Decision |
|---|---|
| Storage format | Delta tables, one per source collection. |
| Load mode | **Batch.** A scheduled job reads all new files, loads them, and stops. Nothing runs between schedules. |
| Load method | Auto Loader with the available-now trigger (see [section 5.2](#52-ingestion-method)). |
| Write mode | **Append only.** Rows are added, never updated or deleted by the pipeline. |
| What a row is | One version of one source document. A document that changed three times appears three times. |
| Changes to source data | None. Values keep their original text, structure and field names. |
| Columns added | Four load columns, all prefixed `bronze_` (see [section 6](#6-schema-and-added-columns)). |
| Who can read it | The pipeline service identity only (see [section 8](#8-security)). |

## 2. Diagram

Click the image to open it at full size. The SVG zooms without losing sharpness. A PNG copy is in [bronze_ingestion_diagram.png](bronze_ingestion_diagram.png).

[![Bronze layer ingestion flow](bronze_ingestion_diagram.svg)](bronze_ingestion_diagram.svg)

How to read it:

- Grey is the landing zone (files), dark slate is the ingestion step, orange is bronze. The colours match the other layer diagrams.
- Arrows show the direction of the data. Files are read, never moved or edited.
- The two boxes at the bottom summarize section 3.

## 3. What bronze does 

| Bronze does |
|---|
| Keep every document version exactly as received.|
| Add load metadata so each row can be traced to its file and batch. 
| Add new fields automatically when the source schema grows. 
| Keep the full history so any run can be replayed. 
| Keep fields that do not fit the expected schema in a separate column instead of losing them.

## 4. Bronze tables

| Table | Source collection | Row meaning | Feeds silver table(s) |
|---|---|---|---|
| `bronze.users` | `users` | One user document version. | `USERS`, `USER_ITEM` |
| `bronze.orders` | `orders` | One order document version. | `ORDERS`, `ORDER_SUPPLIER_PO`, `ORDER_LINE` |
| `bronze.bills` | `bills` | One bill document version. | `BILLS`, `BILL_LINE` |
| `bronze.supplier_product` | To be confirmed | One supplier product source record version. | `SUPPLIER_PRODUCT` |

Total: 4 tables. The source of `bronze.supplier_product` is an open item (see [section 10](#10-assumptions-and-open-items)).

## 5. How data gets into bronze

### 5.1 Extract to the landing zone

MongoDB is not read directly by the bronze load. A scheduled export job first writes files to a landing zone in cloud object storage. This keeps the load independent of the live database and leaves a replayable copy of exactly what was extracted.

| Topic | Decision |
|---|---|
| Format | JSON files in MongoDB extended JSON, so wrappers such as `$oid` and `$date` are kept as they are. |
| Layout | One folder per collection, then one folder per run date. Files are written once and never edited. |
| What is exported | Documents whose `_meta.at` is later than the last successful export, with a small overlap window so late writes are not missed. |
| First run | A full export of every collection. |
| Full export later | Only when needed, for example to detect deleted documents or to recover from a gap. |
| Duplicates | The overlap window and full exports can deliver the same document version more than once. Bronze keeps them. Silver keeps one. |

Other ways to get the data in, and why they are not the default:

| Option | Why not the default |
|---|---|
| Read MongoDB directly into bronze with a connector | Simple, but there is no file copy to replay from and the load reads the live database. |
| Change data capture tool | Built for continuous replication. The data is batch in nature, so the extra moving parts are not needed. |

### 5.2 Ingestion method


| Criterion | Auto Loader 
|---|---|
| Tracks files already loaded | Yes, in a checkpoint |
| Handles schema growth | Yes, with a rescued data column | 
| Scales as files accumulate | Built for millions of files | 
| Reloading a chosen subset of files | Harder | 
| Runs as scheduled batch | Yes, with the available-now trigger |
| Needs a checkpoint to manage | Yes |

**Auto Loader with the available-now trigger.**

- **It is batch.** Auto Loader is built on Spark's streaming engine, but with the available-now trigger it behaves like a scheduled batch job. It starts, reads every file that arrived since the last run, writes them, and stops. No cluster stays up between runs, and no data is processed in real time.
- **MongoDB documents change shape.** New fields appear without warning. Auto Loader evolves the schema and keeps anything that does not fit in a rescued data column, so nothing is lost and the job does not fail.
- **It keeps working as data grows.** File discovery stays cheap as the landing zone fills up.


### 5.3 Run steps

Each batch run performs the same six steps for every collection.

| Step | What happens |
|---|---|
| 1. Find new files | Compare the landing folder with the checkpoint. Only files not read before are picked up. |
| 2. Read as they are | Values stay as text. Nested objects and arrays keep their shape. |
| 3. Add load metadata | `bronze_ingested_at`, `bronze_batch_id` and `bronze_source_file` are added to every row. |
| 4. Set aside misfits | Fields that do not fit the schema are kept in `bronze_rescued_data`. Nothing is dropped. |
| 5. Append to Delta | Rows are added to the bronze table. Existing rows are never changed. |
| 6. Record and check | The checkpoint is updated. File counts and row counts are logged and compared (see [section 9](#9-checks)). |

A re-run is safe. Files already loaded are skipped, so running the job twice does not create duplicate rows.

## 6. Schema and added columns

**How values are read.**

| Topic | Rule |
|---|---|
| Simple values | Read as text. A number such as `12.50` stays exactly `12.50`. Type conversion happens in silver. |
| Nested objects and arrays | Keep their original shape and field names, including `_id`, `_meta` and wrappers such as `$oid` and `$date`. |
| Dynamic-key fields | Fields whose keys are data, such as cart items keyed `SUP:SKU`, compared prices and the supplier map inside an order, are read as maps. A new key never becomes a new column. |
| New fields | Added to the table automatically. |
| Fields that do not fit | Kept in `bronze_rescued_data`. |

**Columns added to every bronze table.**

| Column | Type | Meaning |
|---|---|---|
| `bronze_ingested_at` | `TIMESTAMP` | When the row was loaded into bronze. Carried into silver under the same name. |
| `bronze_batch_id` | `STRING` | Identifier of the batch run. Used by silver to find rows it has not processed yet. |
| `bronze_source_file` | `STRING` | Path of the landing file the row came from. |
| `bronze_rescued_data` | `STRING` | Any fields that did not fit the schema. Normally empty. |

The `bronze_` prefix follows the silver naming convention for audit columns and cannot clash with source fields such as `_id` and `_meta`.

## 7. Storage and maintenance

| Topic | Decision |
|---|---|
| Partitioning | None to start. Tables of this size do not need it. Revisit if a table grows past about 1 TB. |
| Table history | Append-only, so full history is always available. |
| Compaction | Small files are compacted on a schedule, for example weekly. |
| Clean-up | Old file versions are removed after the retention window. The window must respect the erasure policy in the silver document. |
| Landing files retention | Proposed: 90 days after a successful load, then archived to cheaper storage. |
| Bronze table retention | Kept in full. Deletions happen only through the erasure process, keyed on `user_id`. |


## 8. Security

Bronze is unmasked. It can contain personal data and the `password` field, which never moves beyond bronze.

| Practice | How it is applied |
|---|---|
| Least privilege | Only the pipeline service identity can read or write bronze. Analysts and BI tools have no access. |
| Encryption | Encryption at rest on the landing zone and on bronze storage. |
| Credentials | `password` stays in bronze only and is never copied to silver or gold. |
| Erasure | Deletion requests reach bronze, silver and older Delta file versions within the retention policy. |
| Support access | Recovering a value from bronze, such as the `_meta.by` email, goes through a logged, approved request. |

## 9. Checks

Light checks only. Data quality rules belong to silver.

| # | Check | Action on failure |
|---|---|---|
| 1 | Every new landing file was loaded exactly once. | Fail the run. |
| 2 | Rows added to the table equal the records in the files loaded. | Fail the run. |
| 3 | A batch that loads zero files is reported, not silently accepted. | Warn. |
| 4 | The share of rows with a non-empty `bronze_rescued_data` is reported. A jump means the source schema changed. | Warn above a set limit. |
| 5 | New top-level fields seen in the batch are listed. | Warn. |
| 6 | Newest `bronze_ingested_at` per table is within the expected freshness. | Alert when late. |
| 7 | Duplicate versions of the same document in a batch are counted. | Report only. |

## 10. Assumptions and open items

| # | Item | Current assumption 
|---|---|---|
| B1 | Platform | Delta, Auto Loader and COPY INTO point to Databricks, with cloud object storage as the landing zone. 
| B2 | Source collections | Three known collections (`users`, `orders`, `bills`). The source of `bronze.supplier_product` is not settled (silver open item 1).
 B3 | Deleted documents | An incremental export cannot see deletions. An occasional full export is the proposed way to detect them. How deletions should appear in silver is not decided. 
| B4 | Data volume | Not known. 