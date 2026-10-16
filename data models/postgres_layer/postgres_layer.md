# Postgres Layer

This document describes the last step of the pipeline: how the gold tables are published to Postgres, how they are laid out there, who can read them, and what changes if Redshift is used instead.

**Scope.** The serving database and the publish step. 

## Contents

- [1. Overview](#1-overview)
- [2. Postgres or Redshift](#2-postgres-or-redshift)
- [3. Type mapping](#3-type-mapping)
- [4. constraints ](#4-constraints)


## 1. Overview

Gold lives in Delta and is the source of truth. Postgres holds a read-only copy of the gold tables so that BI tools and analysts can use ordinary SQL against a familiar database. Every run publishes a full snapshot of every gold table. Nothing is edited in Postgres.

| Topic | Decision |
|---|---|
| Target | Postgres |
| Model published | star model|
| Content | The gold tables as they are. Same table names, same column names, no PII. |
| Refresh | Full snapshot each run, loaded to a staging schema and swapped in as one step. |
| Direction | One way, from Delta to Postgres. Postgres is never read back by the pipeline. |
| Access | A read-only role for analysts and BI tools. |


## 2. Postgres or Redshift

**Decision: Postgres.** The gold model has eight tables built for ten business questions, which is well inside what Postgres serves comfortably. I dont know postgres.


## 3. Type mapping

| Gold type (Delta) | Postgres |
|---|---|
| `STRING` | `text` | 
| `INT` | `integer` | 
| `DECIMAL(12,2)` | `numeric(12,2)` |
| `DECIMAL(12,4)` | `numeric(12,4)` | 
| `BOOLEAN` | `boolean` |
| `DATE` | `date` | 
| `TIMESTAMP` (UTC) | `timestamptz` | 
| `ARRAY<STRING>` | `text[]` |

Values are copied without change. No rounding, trimming or recasting happens during the publish.

## 4. constraints

| Constraint | Decision | Why |
|---|---|---|
| Primary keys | Created and enforced on every table. | A duplicate key is a bug, and the load should fail on it rather than publish it. |
| Not null | On all primary key columns. | Follows from the primary key. |
| Foreign keys | **Not created.** They are documented in the gold documents. | Some fact rows have no matching dimension row, by design (the gold design principle that facts keep their keys). An enforced foreign key would reject them and break the totals. |

