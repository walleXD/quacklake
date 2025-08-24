# Quacklake

## Vision & Scope

| Item           | DuckLake (today)                         | ducklake-rs                  | ducklake-py                                                 |
| -------------- | ---------------------------------------- | ---------------------------- | ----------------------------------------------------------- |
| Compute Engine | DuckDB                                   | Any (via Arrow + DataFusion) | Same                                                        |
| Metadata Store | DuckDB / Postgres / SQLite / MotherDuck  | Pluggable adapters           | Pluggable                                                   |
| Storage Layer  | Any object store (S3, GCS, Azure, local) | Same                         | Same                                                        |
| Client API     | SQL only                                 | Rust API + Arrow Flight/IPC  | Python: `scan(...) -> pandas.DataFrame \| polars.DataFrame` |

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Python Layer (ducklake-py)                                  │
│  • PyO3 bindings                                             │
│  • Arrow C Data → Pandas / Polars zero-copy                  │
│  • Thin wrapper around ducklake-rs                           │
└────────────────────┬───────────────────────────────────────────┘
│ Arrow Flight / IPC
┌────────────────────┴───────────────────────────────────────────┐
│  ducklake-rs (Rust)                                          │
│  • Catalog connection via SQLx (DuckDB/SQLite/Postgres)      │
│  • Object-store abstraction (opendal)                        │
│  • Arrow RecordBatches in/out                                │
│  • DataFusion integration (optional compute)                 │
└──────────────────────────────────────────────────────────────┘
```

## Component Breakdown

| Layer           | Crate               | Responsibilities                                                    |
| --------------- | ------------------- | ------------------------------------------------------------------- |
| **formats**     | `ducklake-format`   | Parquet footers, manifest schema, metadata protobuf/Arrow encodings |
| **catalog**     | `ducklake-catalog`  | Table, schema, snapshot, partition metadata in SQL tables           |
| **storage**     | `ducklake-storage`  | Object-store abstraction (S3, GCS, Azure, local) with async I/O     |
| **transaction** | `ducklake-transact` | ACID commit protocol, optimistic concurrency, snapshot isolation    |
| **scan**        | `ducklake-scan`     | Projection / filter push-down, file pruning, Arrow reader           |
| **python**      | `ducklake-py`       | PyO3 bindings, Arrow C Data bridge to Pandas & Polars               |
| **cli**         | `ducklake-cli`      | Thin CLI for quick checks, powered by the Rust library              |

### Phase 1 – Read-Only MVP (v 0.1.0)

| Deliverable | Detail                                                                                                                                                                                                                                                            |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **catalog** | Initialize minimal DuckLake catalog via SQLx in Postgres Using `sqlx`: `CREATE SCHEMA IF NOT EXISTS lakehouse;` plus tables `catalogs`, `schemas`, `tables`, `snapshots`, `manifests`, `partitions`. Provide `Catalog::connect()` that returns a connection pool. |
| **formats** | Decode DuckLake manifest (protobuf/Arrow) into Rust structs.                                                                                                                                                                                                      |
| **storage** | Integrate `opendal` crate; list objects, read Parquet.                                                                                                                                                                                                            |
| **scan**    | Implement `LakehouseSession::scan(table_ref) -> DataFrame` that:<br>1. Queries catalog for latest snapshot.<br>2. Resolves manifest paths.<br>3. Prunes files & yields Arrow `RecordBatch`.                                                                       |
| **python**  | Expose `session = ducklake.connect("duckdb:///lake.db")` and `session.table("analytics.events").to_pandas()` via PyO3.                                                                                                                                            |
| **tests**   | Spin up ephemeral DuckLake catalog in CI; add golden-file tests from DuckLake reference samples.                                                                                                                                                                  |

Success metric: `cargo test` passes; Python wheel can open a DuckLake catalog DB created by DuckDB and return a Pandas frame.

Phase 2 – Transactional Writes (v 0.2.0)
| Deliverable | Detail |
|-------------|--------|
| **transaction** | Optimistic locking, commit conflict retry, rollback using SQLx transactions. |
| **python** | `session.table("analytics.events").append(df)` & `session.table("analytics.events").merge(df, on="id")` |
| **tests** | Concurrency chaos tests with `tokio` tasks. |

### Phase 3 – Pluggable Catalog Back-Ends (v 0.3.0)

| Deliverable                                                                          | Detail |
| ------------------------------------------------------------------------------------ | ------ |
| Trait `Catalog` with SQLx impls: `MySQLCatalog`, `SqliteCatalog`, `PostgresCatalog`. |
| Config file or env-vars to switch back-ends.                                         |
