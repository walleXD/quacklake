# Quacklake

## Vision & Scope

| Item           | DuckLake (today)                         | quacklake-rs                 | quacklake-py                                                |
| -------------- | ---------------------------------------- | ---------------------------- | ----------------------------------------------------------- |
| Compute Engine | DuckDB                                   | Any (via Arrow + DataFusion) | Same                                                        |
| Metadata Store | DuckDB / Postgres / SQLite / MotherDuck  | Pluggable adapters           | Pluggable                                                   |
| Storage Layer  | Any object store (S3, GCS, Azure, local) | Same                         | Same                                                        |
| Client API     | SQL only                                 | Rust API + Arrow Flight/IPC  | Python: `scan(...) -> pandas.DataFrame \| polars.DataFrame` |

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Python Layer (quacklake-py)                                 │
│  • PyO3 bindings                                             │
│  • Arrow C Data → Pandas / Polars zero-copy                  │
│  • Thin wrapper around quacklake-rs                          │
└────────────────────┬─────────────────────────────────────────┘
│ Arrow Flight / IPC
┌────────────────────┴─────────────────────────────────────────┐
│  quacklake-rs (Rust)                                         │
│  • Catalog connection via SQLx (DuckDB/SQLite/Postgres)      │
│  • Object-store abstraction (opendal)                        │
│  • Arrow RecordBatches in/out                                │
│  • DataFusion integration (optional compute)                 │
└──────────────────────────────────────────────────────────────┘
```

## Component Breakdown

| Layer           | Crate                | Responsibilities                                                    |
| --------------- | -------------------- | ------------------------------------------------------------------- |
| **formats**     | `quacklake-format`   | Parquet footers, manifest schema, metadata protobuf/Arrow encodings |
| **catalog**     | `quacklake-catalog`  | Table, schema, snapshot, partition metadata in SQL tables           |
| **storage**     | `quacklake-storage`  | Object-store abstraction (S3, GCS, Azure, local) with async I/O     |
| **transaction** | `quacklake-transact` | ACID commit protocol, optimistic concurrency, snapshot isolation    |
| **scan**        | `quacklake-scan`     | Projection / filter push-down, file pruning, Arrow reader           |
| **python**      | `quacklake-py`       | PyO3 bindings, Arrow C Data bridge to Pandas & Polars               |
| **cli**         | `quacklake-cli`      | Thin CLI for quick checks, powered by the Rust library              |

### Phase 1 – Read-Only MVP (v0.1.0)

| Deliverable | Detail                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **catalog** | **Read-only** adapter that connects to an **already-initialized** DuckLake catalog via SQLx (DuckDB, Postgres, or SQLite).<br>- Expects tables `lakehouse.catalogs`, `lakehouse.schemas`, `lakehouse.tables`, `lakehouse.snapshots`, `lakehouse.manifests`, `lakehouse.partitions` to exist.<br>- Provide `Catalog::connect_readonly(url)` that returns a connection pool **without running migrations**. |
| **formats** | Decode DuckLake manifest (protobuf/Arrow) into Rust structs.                                                                                                                                                                                                                                                                                                                                              |
| **storage** | Integrate `opendal`; list objects, read Parquet.                                                                                                                                                                                                                                                                                                                                                          |
| **scan**    | Implement `LakehouseSession::scan(catalog_ref, schema_ref, table_name) -> DataFrame` that:<br>1. Queries existing catalog for latest snapshot.<br>2. Resolves manifest paths.<br>3. Prunes files & yields Arrow `RecordBatch`.                                                                                                                                                                            |
| **python**  | Expose `session = ducklake.connect("duckdb:///lake.db")` and `session.table("analytics.events").to_pandas()` via PyO3.                                                                                                                                                                                                                                                                                    |
| **tests**   | Spin up golden catalog fixture (DuckDB file) in CI; read-only integration test.                                                                                                                                                                                                                                                                                                                           |
|             |

Success metric: `cargo test` passes; Python wheel can open a DuckLake catalog DB created by DuckDB and return a Pandas frame.

### Phase 2 – Transactional Writes (v0.2.0)

| Deliverable     | Detail                                                                                                  |
| --------------- | ------------------------------------------------------------------------------------------------------- |
| **transaction** | Optimistic locking, commit conflict retry, rollback using SQLx transactions.                            |
| **python**      | `session.table("analytics.events").append(df)` & `session.table("analytics.events").merge(df, on="id")` |
| **tests**       | Concurrency chaos tests with `tokio` tasks.                                                             |
| **python**      | `session.table("analytics.events").append(df)` & `session.table("analytics.events").merge(df, on="id")` |
| **tests**       | Concurrency chaos tests with `tokio` tasks.                                                             |

### Phase 3 – Creating **new** catalogs (v0.3.0)

| Deliverable | Detail                                                                                                                                                                                                                                                                                                                            |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **catalog** | **Write & create** adapter:<br>- SQLx migrations embedded via `sqlx migrate` to create `lakehouse.*` tables.<br>- Provide `Catalog::create_catalog(url)` and `Catalog::create_schema(schema)` helpers.<br>- Transactional commit protocol (optimistic locking, commit conflict retry, rollback) piggy-backs on SQLx transactions. |
| **python**  | `session.create_catalog("lakehouse")`, `session.create_schema("analytics")`, `session.create_table("events", schema)`, `session.table("analytics.events").append(df)` & `session.table("analytics.events").merge(df, on="id")`                                                                                                    |
| **tests**   | End-to-end flow: create empty DuckDB file, run migrations, write a small DataFrame, read it back.                                                                                                                                                                                                                                 |

### Phase 4 – Pluggable Catalog Back-Ends (v0.4.0)

| Deliverable           | Detail                                                                                                                                                                     |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Trait `Catalog`**   | Define a single async trait `Catalog { async fn snapshots(...), async fn manifests(...) }` with SQLx implementations: `DuckDBCatalog`, `SqliteCatalog`, `PostgresCatalog`. |
| **Runtime selection** | Load back-end via `DUCKLAKE_CATALOG_URL` env var or `ducklake.toml` (`[catalog] url = "postgres://..."`).                                                                  |
