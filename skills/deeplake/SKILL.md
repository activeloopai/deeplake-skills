---
name: deeplake
description: SDK and CLI for ingesting data into Deeplake managed tables, and mounting a cloud-backed filesystem for persistent agent memory. Use when users want to store, ingest, query data in Deeplake, or set up persistent memory/filesystem.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, WebFetch
metadata:
  openclaw:
    requires:
      env:
        - DEEPLAKE_API_KEY
    primaryEnv: DEEPLAKE_API_KEY
    homepage: https://github.com/activeloopai/deeplake-skills
---

# Deeplake Managed Service SDK

> Agent-friendly SDK for ingesting data into Deeplake managed tables.
> Use this skill when users want to store, ingest, or query data in Deeplake.
> Available in both **Python** and **Node.js/TypeScript**.

## DeepLake CLI (Filesystem Mode)

For persistent agent memory as a mounted filesystem, use the **deeplake CLI**:

```bash
curl -fsSL https://deeplake.ai/install.sh | bash   # Install CLI
deeplake init              # Interactive setup (auth + mount)
deeplake mount             # Mount all registered filesystems
deeplake umount --all      # Unmount all
deeplake list              # Show mount status
```

After `deeplake init`, a FUSE-mounted directory appears at your chosen path (e.g. `~/agent-memory`).
Files written there are cloud-synced to DeepLake in real-time and persist across sessions/reboots.

**When to use CLI vs SDK:**
- **CLI** — persistent filesystem for agent memory, config, and sandboxes (read/write files normally)
- **SDK** — programmatic data ingestion, vector search, ML training pipelines

---

## Quick Reference

### Python

```bash
pip install deeplake # uv add deeplake 
```

**Python import (primary):**
```python
from deeplake import Client

# Async variant (requires aiohttp: pip install aiohttp):
from deeplake.managed import AsyncClient
```


```python
from deeplake import Client

# Initialize -- token from DEEPLAKE_API_KEY env var, workspace defaults to "default"
client = Client()
client = Client(token="dl_xxx", workspace_id="my-workspace")

# Ingest files (FILE schema)
client.ingest("videos", {"path": ["video1.mp4", "video2.mp4"]}, schema={"path": "FILE"})

# Ingest structured data with indexes for search
client.ingest("embeddings", {
    "text": ["doc1", "doc2", "doc3"],
    "embedding": [[0.1, 0.2, ...], [0.3, 0.4, ...], [0.5, 0.6, ...]],
}, index=["embedding", "text"])

# Ingest from HuggingFace
client.ingest("cifar", {"_huggingface": "cifar10"})

# Ingest with format object (see formats.md for CocoPanoptic, Coco, LeRobot, custom)
client.ingest("table", format=my_format)

# Fluent query
results = client.table("videos").select("id", "text").where("file_id = $1", "abc").limit(10)()

# Raw SQL
results = client.query("SELECT * FROM videos LIMIT 10")

# Vector similarity search -- through client.query() the vector must be
# INLINED, not passed as $1 (see "Querying" below for why)
from deeplake.managed import vector_literal
results = client.query(f"""
    SELECT id, text, embedding::float4[] <#> {vector_literal(query_embedding)} AS similarity
    FROM embeddings ORDER BY similarity DESC LIMIT 10
""")

# Create an empty table with an explicit schema (no data yet)
client.create_table("docs", {
    "id": "TEXT",
    "embedding": "EMBEDDING(1536)",     # NOT bare "EMBEDDING" — keep the dimension
}, index={"id": "inverted_index", "embedding": "clustered"})

# Table management
client.list_tables()
client.drop_table("old_table")
client.create_index("embeddings", "embedding", index_type="clustered")
client.drop_index("embeddings", "embedding")
client.describe_table("embeddings")     # {col: {"type": ..., "index": ...}}

# Open a table without taking write access
ds = client.open_table("embeddings", read_only=True)
```

### Node.js / TypeScript

```
npm install deeplake
```

**TypeScript import:**
```typescript
import { ManagedClient, initializeWasm } from 'deeplake';
```

**WASM initialization (required before any operations):**
```typescript
await initializeWasm();
```
Call `initializeWasm()` once at startup before any `ManagedClient` operations (ingest, query, etc.). It initializes the underlying WASM module.


```typescript
import { ManagedClient, initializeWasm } from 'deeplake';

await initializeWasm();

const client = new ManagedClient({ token: 'dl_xxx', workspaceId: 'my-workspace' });

// Ingest files (FILE schema)
await client.ingest("videos", { path: ["video1.mp4"] }, { schema: { path: "FILE" } });

// Ingest structured data
await client.ingest("embeddings", {
    text: ["doc1", "doc2"],
    embedding: [[0.1, 0.2], [0.3, 0.4]],
});

// Ingest with format object (see formats.md)
await client.ingest("table", null, { format: myFormat });

// Fluent query (use .execute())
const results = await client.table("videos")
    .select("id", "text").where("file_id = $1", "abc").limit(10).execute();

// Raw SQL
const rows = await client.query("SELECT * FROM videos LIMIT 10");

// Create an empty table with an explicit schema (no data yet)
await client.createTable("docs", {
    id: "TEXT",
    embedding: "EMBEDDING(1536)",       // NOT bare "EMBEDDING" — keep the dimension
}, { index: { id: "inverted_index", embedding: "clustered" } });

// Table management
await client.listTables();
await client.dropTable("old_table");
await client.createIndex("embeddings", "embedding", { indexType: "clustered" });
await client.dropIndex("embeddings", "embedding");
await client.describeTable("embeddings");   // {col: {type, index}}

// Open a table without taking write access
const ds = await client.openTable("embeddings", true);
```

---

## Dependancies and Prerequisite

**Required services:**
- Deeplake API server running (default: `https://api.deeplake.ai`)

**Optional python dependencies (per file type):**
- Video ingestion: `ffmpeg` (`sudo apt-get install ffmpeg`)
- PDF ingestion: `pymupdf` (`pip install pymupdf`)
- Thumbnail generation: `Pillow` (`pip install Pillow`)
- COCO detection format: `pycocotools`, `Pillow`, `numpy` (`pip install pycocotools Pillow numpy`)
- LeRobot frames format: `pandas`, `numpy` (`pip install pandas numpy`)

**Optional typescript dependencies (per file type):**
- Video ingestion: `ffmpeg` (system binary)
- PDF ingestion: `pdfjs-dist` (`npm install pdfjs-dist`)
- Thumbnail generation: `sharp` (`npm install sharp`)
- COCO detection format: no external deps (pure JS mask rendering)

## Architecture

```
Python:  Client(token, workspace_id)
Node.js: ManagedClient({ token, workspaceId })
  |-- .ingest(table, data)       -> creates PG table via API, opens al://{ws}/{table}
  |                                 via deeplake SDK (auto credential rotation)
  |-- .create_table(table, schema) -> POST /workspaces/{id}/tables (empty table, no rows)
  |-- .query(sql)                -> POST /workspaces/{id}/tables/query -> list[dict] / QueryRow[]
  |-- .table(table)...           -> fluent SQL builder -> list[dict] / QueryRow[]
  |-- .create_index(table, col, index_type=) -> CREATE INDEX USING deeplake_index
  |-- .drop_index(table, col)    -> DROP INDEX (needed to change a column's index type)
  |-- .describe_table(table)     -> {col: {type, index}} read back from storage
  |-- .open_table(table, read_only=) -> deeplake.open[_read_only]("al://{ws}/{table}")
  |-- .list_tables()             -> GET /workspaces/{id}/tables -> list[str] / string[]
  `-- .drop_table(table)         -> DELETE /workspaces/{id}/tables/{name}
                    |
                    v
              REST API -> PostgreSQL + pg_deeplake
  - All DB operations go through the REST API (no direct PG connection)
  - Dataset access uses al:// paths with automatic credential resolution
  - Creds endpoint: GET /api/org/{workspace}/ds/{table}/creds
  - Vector similarity: embedding::float4[] <#> query_vec
  - BM25 text search:  text <#> 'search query'
  - Hybrid search:     (embedding, text)::deeplake_hybrid_record
```

---

## Client Initialization

### Python

```python
from deeplake import Client

client = Client(
    token: str = None,           # API token (falls back to DEEPLAKE_API_KEY env var)
    workspace_id: str = "default",  # Target workspace (default: "default")
    api_url: str = None,         # API URL (default: https://api.deeplake.ai)
    org_id: str = None,          # Target org UUID. REQUIRED when the token's embedded
                                 #   org differs from the org you want to act in (see below).
)
```

> **Cross-org calls — `org_id=` is easy to miss.** An API token embeds a **single**
> `org_id` claim, even for a user who belongs to many orgs. Every request the SDK
> makes is scoped to an org; if you omit `org_id`, that scope silently defaults to
> the **token's** embedded org — not necessarily the one you intended. Whenever the
> token-org ≠ the target-org, pass `org_id=` (Python/JS kwarg) or set
> `ACTIVELOOP_ORG_ID`; the SDK forwards it as the `X-Activeloop-Org-Id` HTTP header.
> The server only honors that exact header — other org-ish header names
> (e.g. `X-Org-Id`) are ignored, so a wrong/missing value creates resources in the
> token's org **with no error**. Map an org **name → UUID** via `GET /me` (returns
> the org IDs your token can reach). See [reference.md](reference.md) for the full
> auth-provider / `org_id` contract.

### Node.js / TypeScript

```typescript
import { ManagedClient, initializeWasm } from 'deeplake';

await initializeWasm();

const client = new ManagedClient({
    token: string,               // API token (required)
    workspaceId?: string,        // Target workspace (default: "default")
    apiUrl?: string,             // API URL (default: https://api.deeplake.ai)
});
```

**Token:** Create API tokens from the Deeplake platform at `https://app.deeplake.ai/<org_name>/workspace/<workspace>/apitoken`. The token is a JWT with `org_id` embedded. Falls back to the `DEEPLAKE_API_KEY` environment variable (Python only).

**Backend endpoint:** The client sets the C++ backend endpoint to `api_url` before each dataset open (not on initialization) so that `al://` path resolution (credential fetching) goes through deeplake-api instead of the legacy controlplane. This avoids global state clobbering when multiple clients use different API URLs. Python: `deeplake.client.endpoint = api_url`. Node.js: `deeplakeSetEndpoint(apiUrl)`.

**Connection lifecycle:**
```python
# Python: just create and use -- no connection to manage
client = Client()
client.ingest("table", {"path": ["file.txt"]}, schema={"path": "FILE"})
# No close() method -- client is stateless (REST API calls only)
```

---

## Ingestion

### Python: client.ingest()

```python
result = client.ingest(
    table_name: str,                    # Table name to create (must not already exist)
    data: dict[str, list] = None,       # Data dict (required unless format= is set).
                                        #   {"_huggingface": "name"} -> HuggingFace dataset
                                        #   schema has "FILE" cols -> file paths processed
                                        #   otherwise -> column data {col: [values]}
    *,
    format: Format = None,              # Format object (subclass of Format) with
                                        #   normalize() method. When set, data is ignored.
                                        #   e.g. CocoPanoptic(images_dir=..., ...)
    schema: dict[str, str] = None,      # Schema override {col: type}
                                        #   Use "FILE" for columns containing file paths
                                        #   See reference.md for all type names
    index: list[str] = None,            # Columns to create deeplake_index on after ingestion.
                                        #   Use for EMBEDDING (vector search) and TEXT (BM25) columns.
    on_progress: Callable = None,       # Progress callback(rows_written, total)
    chunk_size: int = 1000,             # Text chunk size (chars)
    chunk_overlap: int = 200,           # Text chunk overlap (chars)
    pdf_dpi: int = 150,                 # PDF render DPI (higher = sharper but slower)
) -> dict
```

### Node.js: client.ingest()

```typescript
const result = await client.ingest(
    tableName: string,                              // Table name
    data?: Record<string, unknown[]> | null,        // Data dict (or null when using format)
    options?: {
        format?: Format,                            // Format object with normalize()
        schema?: Record<string, string>,            // Schema override
        index?: string[],                           // Columns to create deeplake_index on
        onProgress?: (processed, total) => void,    // Progress callback
        chunkSize?: number,                         // Text chunk size (default 1000)
        chunkOverlap?: number,                      // Text chunk overlap (default 200)
    },
): Promise<IngestResult>
```

**Table existence:** If `table_name` already exists, `ingest()` appends data to the existing table — it does NOT drop and recreate it. To replace an existing table, call `client.drop_table(table_name)` first. The PG table schema must be compatible with the new data being appended.

**Returns:** `{"table_name": "videos", "row_count": 150, "dataset_path": "al://workspace_id/videos"}`

> **⚠️ Always check `row_count` against what you sent.** `ingest()` returning
> without raising is **not** proof the rows committed. During backend
> credential-refresh / provisioning windows a batch can be accepted at the HTTP
> layer yet land zero rows, and older servers reported `row_count: 0` for a write
> even when it succeeded. Treat `row_count` as the source of truth: if it doesn't
> equal `len(data)`, retry the missing rows (ingest is **append**, so use a
> deterministic primary key + de-dupe, or `INSERT … ON CONFLICT DO NOTHING`, to
> stay idempotent). After a large ingest, confirm with
> `SELECT COUNT(*) FROM <table>` — but mind the visibility lag (see
> [Querying → read-your-writes](#querying)).

**Both `data` and `format`:** If both are provided, `format` takes precedence and `data` is ignored. If neither is provided, an `IngestError` is raised.

**Thumbnails:** When a format object declares `image_columns()` (columns with pg_schema type `"IMAGE"`), thumbnails are auto-generated at 4 sizes (32x32, 64x64, 128x128, 256x256) and stored in a shared `thumbnails` dataset at `{root_path}/thumbnails`. Requires Pillow (Python) or sharp (Node.js).

### Chunking Strategy by File Type

| File Type | Extensions                           | Strategy                        | Columns Created                                                             |
| --------- | ------------------------------------ | ------------------------------- | --------------------------------------------------------------------------- |
| **Video** | .mp4, .mov, .avi, .mkv, .webm        | 10-second segments + thumbnails | id, file_id, chunk_index, start_time, end_time, video_data, thumbnail, text |
| **Image** | .jpg, .jpeg, .png, .gif, .bmp, .webp | Single chunk                    | id, file_id, image, filename, text                                          |
| **PDF**   | .pdf                                 | Page-by-page at 150 DPI (configurable via `pdf_dpi`) | id, file_id, page_index, image, text                                        |
| **Text**  | .txt, .md, .csv, .json, .xml, .html  | 1000 char chunks, 200 overlap   | id, file_id, chunk_index, text                                              |
| **Other** | *                                    | Single binary chunk             | id, file_id, data, filename                                                 |

### Key Examples

```python
# Ingest files (FILE schema)
client.ingest("videos", {"path": ["cam1.mp4", "cam2.mp4"]}, schema={"path": "FILE"})
client.ingest("photos", {"path": ["img1.jpg"]}, schema={"path": "FILE"})
client.ingest("manuals", {"path": ["manual.pdf"]}, schema={"path": "FILE"})

# Ingest structured data (dict = column data, schema inferred)
client.ingest("vectors", {
    "text": ["Hello", "Goodbye"],
    "embedding": [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6]],
})

# Ingest with explicit schema
client.ingest("data", {"name": ["Alice", "Bob"], "age": [30, 25]},
              schema={"name": "TEXT", "age": "INT64"})

# JSONB is a supported schema type — store arbitrary nested JSON per row.
# (Not obvious from the type table; confirmed working end-to-end.)
client.ingest("events", {"id": [1, 2], "payload": [{"k": "v"}, {"n": [1, 2]}]},
              schema={"id": "INT64", "payload": "JSONB"})

# Ingest from HuggingFace
client.ingest("mnist", {"_huggingface": "mnist"})

# Ingest with a format object (see formats.md for CocoPanoptic, Coco, LeRobot, custom)
client.ingest("table", format=my_format)

# Ingest with progress callback
def progress(rows_written, total):
    print(f"Written {rows_written} rows...")
client.ingest("docs", {"path": pdf_files}, schema={"path": "FILE"}, on_progress=progress)
```

> For custom format classes, see [formats.md](formats.md).
> For more ingestion examples, see [examples.md](examples.md).

---

## Training / Streaming

### client.open_table()

Open a managed table as a `deeplake.Dataset` for direct access -- bypasses PostgreSQL and returns the native dataset object with built-in ML framework integration.

```python
ds = client.open_table(table_name: str, read_only: bool = False) -> deeplake.Dataset
```

```typescript
const ds = await client.openTable(tableName: string, readOnly?: boolean);
```

**When to use:** Training loops, batch iteration, PyTorch/TensorFlow DataLoaders, async prefetch.

> **Pass `read_only=True` for anything that only reads.** The default opens
> **read/write**, which takes a writer handle on the dataset and leaves a stray
> `append` / `commit` free to mutate a table you only meant to inspect. With
> `read_only=True` you get a `ReadOnlyDataset` whose mutating methods raise.
> Reading, querying, `summary()`, batch iteration and dataloaders all work
> unchanged.

```python
# Inspect without taking write access
ds = client.open_table("videos", read_only=True)
ds.summary()

# Batch iteration (training reads only — open read-only)
ds = client.open_table("videos", read_only=True)
for batch in ds.batches(32):
    train(batch)

# PyTorch DataLoader
from torch.utils.data import DataLoader
ds = client.open_table("training_data", read_only=True)
loader = DataLoader(ds.pytorch(), batch_size=32, shuffle=True, num_workers=4)

# TensorFlow tf.data.Dataset
ds = client.open_table("training_data", read_only=True)
tf_ds = ds.tensorflow().batch(32).prefetch(tf.data.AUTOTUNE)

# Writing still needs the default read/write handle
ds = client.open_table("training_data")
ds.append({"id": ["x"]})
ds.commit()
```

> **Node.js note:** `openTable(name, true)` needs a WASM build carrying the
> `deeplake_open_read_only` binding, and `describeTable()` needs one carrying
> `Dataset.summary()`. Older bundles raise a clear error naming the missing
> binding; rebuild the WASM module (`python3 scripts/build_wasm_node.py dev`),
> or on the Python side, where both have shipped for longer.

---

## Querying

### Fluent Query API (Recommended)

`client.table(table)` returns a chainable `ManagedQueryBuilder`:

```python
# Python: supports both .execute() and () to run the query
results = (
    client.table("videos")
        .select("id", "text", "start_time")
        .where("file_id = $1", "abc123")
        .where("start_time > $2", 60)
        .order_by("start_time ASC")
        .limit(10)
        .offset(0)
)()  # or .execute()
```

```typescript
// Node.js: use .execute() only (no () shorthand)
// (assumes initializeWasm() already called at startup)
const results = await client.table("videos")
    .select("id", "text", "start_time")
    .where("file_id = $1", "abc123")
    .where("start_time > $2", 60)
    .orderBy("start_time ASC")
    .limit(10)
    .offset(0)
    .execute();
```

| Method                  | Python                | Node.js               | Description                |
| ----------------------- | --------------------- | --------------------- | -------------------------- |
| `.select(*cols)`        | `.select("id", "t")`  | `.select("id", "t")`  | Set columns (default `*`)  |
| `.where(cond, *params)` | `.where("id=$1","x")` | `.where("id=$1","x")` | Add WHERE (multiple = AND) |
| `.order_by(clause)`     | `.order_by("col")`    | `.orderBy("col")`     | Add ORDER BY               |
| `.limit(n)`             | `.limit(10)`          | `.limit(10)`          | Set LIMIT                  |
| `.offset(n)`            | `.offset(20)`         | `.offset(20)`         | Set OFFSET                 |
| Run query               | `.execute()` or `()`  | `.execute()`          | Execute, return results    |

**How `.where()` parameters work:** Each `.where("col = $N", value)` call adds an AND condition. The `$1`, `$2`, etc. placeholders are filled by the extra arguments, numbering across all `.where()` calls sequentially.

### Raw SQL: client.query()

```python
# Python
rows = client.query(
    sql: str,
    params: tuple = None,
    timeout: int = 60,       # HTTP timeout in seconds (increase for slow queries)
) -> list[dict]

# Examples
rows = client.query("SELECT * FROM videos LIMIT 10")
rows = client.query("SELECT * FROM documents WHERE file_id = $1", ("abc123",))
rows = client.query("SELECT COUNT(*) FROM big_table", timeout=300)  # 5-minute timeout
```

```typescript
// Node.js
const rows = await client.query(
    sql: string,
    params?: unknown[],
    options?: { timeoutMs?: number },  // default 60000 (60s)
) -> Promise<QueryRow[]>

// Examples
const rows = await client.query("SELECT * FROM videos LIMIT 10");
const rows = await client.query("SELECT * FROM documents WHERE file_id = $1", ["abc123"]);
const rows = await client.query("SELECT COUNT(*) FROM big", undefined, { timeoutMs: 300_000 });
```

Queries are sent to the API via `POST /workspaces/{id}/tables/query`. Use `$1`, `$2`, ... for parameterized queries.

> **Timeout:** The default query timeout is 60 seconds. For long-running queries (large aggregations, index builds), increase it via the `timeout` / `timeoutMs` parameter. Non-default timeouts are forwarded to the backend as `timeout_ms` so the server can also apply a deadline.

> For pg_deeplake SQL features (vector search, BM25, hybrid search, indexes), see [reference.md](reference.md).

### Read-your-writes: the ingest → query visibility lag

A successful `ingest()` / `INSERT` is **not immediately visible** to a following
read. Writes flush to object storage asynchronously, so a `COUNT(*)` issued
milliseconds after a commit can return the *old* count (often 0 on a fresh
table) — this is a **timing lag, not data loss**. When chaining an ingest into a
query, either:

- poll `SELECT COUNT(*) FROM <table>` until it reaches the expected count (allow
  a few seconds — empirically ~5s), or
- if a side process mutated the same dataset path out-of-band and reads then
  fail with `schema drift` / stale results, evict the cached snapshot:

```sql
SELECT deeplake_resync_table_pointer('my_table'::regclass);  -- re-open at HEAD
SELECT COUNT(*) FROM my_table;                                -- now current
```

`deeplake_resync_table_pointer` (pg_deeplake 1.13+) refuses to run with unflushed
DML — call `deeplake_flush_table('my_table'::regclass)` first. See
[reference.md](reference.md#deeplake_resync_table_pointertbl-regclass).

### SQL dialect — Postgres wire, DuckDB execution

Queries against deeplake tables are parsed as PostgreSQL but **executed by a
DuckDB engine** underneath, so the two dialects don't fully overlap and the split
occasionally leaks into error messages:

- **`jsonb` functions:** DuckDB exposes `json_*`, not the PG `jsonb_*` aliases.
  `jsonb_array_length(col)` fails ("did you mean json_array_length"); even
  `json_array_length(jsonb_col)` can fail. The reliable form is
  `json_array_length(col::json)`. Prefer `json_*` names and cast `jsonb` → `json`.
- **JSON access operators:** the PG `->` / `->>'key'` operators work, but under
  the hood they map to DuckDB path syntax (`'$.key'`) — which is what you'll see
  echoed in errors. Extracting with `json_extract(col, '$.key')` avoids surprises.
- **DDL limits:** in-place column type migration is **not** supported —
  `ALTER TABLE … ALTER COLUMN … TYPE …` fails with
  "VACUUM FULL is not supported for deeplake tables". To change a column type,
  create a new table with the target schema and copy the data
  (`INSERT INTO new SELECT … FROM old`).

---

## Table Management

```python
# Python
tables = client.list_tables() -> list[str]
client.drop_table(table_name: str, if_exists: bool = True) -> None
client.create_table(table_name: str, schema: dict, *,
                    index=None, if_not_exists: bool = True) -> dict
client.create_index(table_name: str, column: str, *,
                    index_type: str = None, dimension: int = None) -> None
client.drop_index(table_name: str, column: str, if_exists: bool = True) -> None
client.describe_table(table_name: str) -> dict
```

```typescript
// Node.js
const tables = await client.listTables();
await client.dropTable(tableName: string, ifExists?: boolean); // default true
await client.createTable(tableName: string, schema: SchemaMap,
                         options?: { index?: IndexSpec, ifNotExists?: boolean });
await client.createIndex(tableName: string, column: string,
                         options?: { indexType?: string, dimension?: number });
await client.dropIndex(tableName: string, column: string, ifExists?: boolean);
await client.describeTable(tableName: string);
```

### Creating a Table

`ingest()` creates a table as a side effect of writing rows. `create_table()` /
`createTable()` creates an **empty** one with an explicit schema. Use it when the
table must exist before any data lands: building a skeleton that mirrors an
existing dataset, provisioning a target for `INSERT INTO … SELECT`, or
pre-declaring domain types that inference can't guess from values.

```python
result = client.create_table("docs", {
    "id": "TEXT",
    "chunk_id": "TEXT",
    "created_at": "INT64",
    "embedding": "EMBEDDING(1536)",
}, index={
    "id": "inverted_index",
    "chunk_id": "inverted_index",
    "embedding": "clustered",
})
# {"table_name": "docs", "created": True, "dataset_path": "al://ws/docs"}
```

> **⚠️ Write `EMBEDDING(dim)`, not bare `EMBEDDING`.** Bare `EMBEDDING` maps to
> an unparameterized `float4[]`, which pg_deeplake stores as a *variable-length*
> float array — not an `embedding(dim, float32)` column. On an empty skeleton
> there are no rows to infer a dimension from, so a later `create_index` has
> nothing to work with (you'd have to pass `dimension=` by hand). With
> `EMBEDDING(1536)` the column comes back as `embedding(1536, clustered)`,
> matching a real vector dataset.
>
> Any type name the map doesn't recognise is now passed through to PostgreSQL
> verbatim, so `SMALLINT`, `TEXT[]`, or `LINK(IMAGE)` all work — and a typo
> surfaces as a server error instead of silently becoming a `TEXT` column.

> **`EMBEDDING(N)` already carries a `clustered` index.** Declaring the column
> is enough — `describe_table` reports `embedding(N, clustered)` with
> `index=clustered` before you call `create_index` at all. Passing
> `index={"col": "clustered"}` is therefore redundant (harmless — same type, so
> it's a no-op), but requesting a **different** type on such a column fails:
>
> ```
> Conflicting index types for column 'emb': existing index type 'clustered'
> ```
>
> To build a `clustered_quantized` index, declare the column as bare
> `"EMBEDDING"` (a plain `float4[]`, which carries no index) and pass the
> dimension at index time:
>
> ```python
> client.create_table("vecs", {"emb": "EMBEDDING"})
> client.create_index("vecs", "emb",
>                     index_type="clustered_quantized", dimension=1536)
> ```

`create_table` returns `created=False` when the table already exists (pass
`if_not_exists=False` to make that an error instead). Unlike `ingest`, an index
failure here **raises** — there is no committed data to protect.

### Index Creation

`create_index()` / `createIndex()` creates a `deeplake_index` on a column. Use it for:
- **EMBEDDING columns** — enables vector cosine similarity search via `<#>`
- **TEXT columns** — enables BM25 ranking or keyword `CONTAINS` lookups

The method executes `CREATE INDEX IF NOT EXISTS ... USING deeplake_index (column)
WITH (index_type = '…')`.

#### Index types — and why the default is usually wrong for a clone

| Column kind        | Accepted `index_type`                                | Default when omitted |
| ------------------ | ---------------------------------------------------- | -------------------- |
| text               | `exact_text`, `bm25`, `inverted_index`               | **`exact_text`**     |
| embedding (1-D)    | `clustered`, `clustered_quantized`                   | `clustered`          |
| embedding (2-D)    | `pooled_quantized`                                   | `pooled_quantized`   |
| numeric            | `inverted_index`                                     | `inverted_index`     |
| json / dict        | `inverted_index`                                     | `inverted_index`     |
| any                | `unique`                                             | —                    |

Aliases accepted: `inverted` → `inverted_index`, `exact` → `exact_text`,
`quantized` / `binary_quantized` → `clustered_quantized`.

> **⚠️ The text default does not match most real datasets.** A bare
> `create_index(table, "text")` builds `exact_text`, which serves only `col = 'x'`
> equality. Source datasets are typically `inverted_index` (keyword `CONTAINS`)
> or `bm25` (full-text ranking). Cloning a dataset's indexes means passing the
> type explicitly — read it from `describe_table()` rather than guessing.

```python
# Python — standalone
client.create_index("documents", "text", index_type="bm25")       # full-text ranking
client.create_index("documents", "keywords", index_type="inverted_index")
client.create_index("embeddings", "embedding", index_type="clustered")

# A clustered index over a bare float4[] column that has no rows yet needs the
# dimension supplied — there is nothing to infer it from.
client.create_index("skeleton", "embedding",
                    index_type="clustered", dimension=1536)

# Python — during ingestion (creates indexes after data is committed)
client.ingest("search_index", {
    "text": documents,
    "embedding": embeddings,
}, index={"text": "bm25", "embedding": "clustered"})
```

```typescript
// Node.js — standalone
await client.createIndex("documents", "text", { indexType: "bm25" });
await client.createIndex("embeddings", "embedding", { indexType: "clustered" });
await client.createIndex("skeleton", "embedding",
                         { indexType: "clustered", dimension: 1536 });

// Node.js — during ingestion
await client.ingest("search_index", {
    text: documents,
    embedding: embeddings,
}, { index: { text: "bm25", embedding: "clustered" } });
```

The `index=` argument accepts three shapes, in both `ingest()` and `create_table()`:

| Form | Meaning |
| ---- | ------- |
| `["embedding", "text"]` | default index type per column kind |
| `{"embedding": "clustered", "text": "bm25"}` | explicit type per column |
| `{"embedding": {"index_type": "clustered", "dimension": 1536}}` | full control |

#### Changing an existing index type

pg_deeplake allows **one index per column**. `CREATE INDEX IF NOT EXISTS` on a
column that is already indexed is skipped by PostgreSQL before pg_deeplake ever
sees it — so re-running with a corrected `index_type` would silently do nothing.
The client checks for this and raises instead:

```python
client.create_index("docs", "text", index_type="bm25")
# TableError: Column 'text' of table 'docs' already has a 'exact_text' index
# (idx_docs_text), which does not match the requested 'bm25'. CREATE INDEX
# would skip it silently. Resolution: client.drop_index('docs', 'text') first.

client.drop_index("docs", "text")
client.create_index("docs", "text", index_type="bm25")   # now takes effect
```

Re-requesting the **same** type is a no-op, so `create_table(..., index=...)` is
safe to re-run.

> **⚠️ Changing a type does not reliably take on the managed service yet.**
> `DROP INDEX` removes the PostgreSQL catalog entry immediately, but each
> backend pod holds its own copy of the deeplake index and drops it on its own
> schedule. A `create_index` with the new type issued straight afterwards is
> rejected by a pod that has not caught up — as "already exists" or as a type
> conflict — often for longer than the client's ~10s retry budget. Measured
> against api-beta: succeeds roughly 1 run in 5.
>
> The client absorbs the short cases and, when it runs out of attempts, **raises
> rather than reporting an index it did not create** — so you never end up
> believing a column is indexed when it is not. If you hit it, wait and retry,
> or create the table with the index type you want from the start. Verify with
> `describe_table()`, which reads storage rather than the catalog.

### Reading a table's real schema and indexes

`describe_table()` opens the table read-only and reports what storage actually
holds — not what was requested at creation time. The built index type is not
exposed anywhere else in the Python/JS API (it is spliced into the type string
that `Dataset.summary()` prints), so this is the only way to read it.

```python
client.describe_table("embedding_noquantized")
# {'chunk_id':   {'type': 'text (compression=lz4)',      'index': 'inverted_index'},
#  'created_at': {'type': 'int64',                       'index': None},
#  'embedding':  {'type': 'embedding(1536, clustered)',  'index': 'clustered'},
#  'id':         {'type': 'text (compression=lz4)',      'index': 'inverted_index'}}
```

Feed that straight back into `create_table` to clone a schema — see
[examples.md](examples.md) for the full skeleton recipe.

> **Caveat:** a clone reproduces column types, dimensions and index types, but
> **not chunk compression** — pg_deeplake hardwires a PG `TEXT` column to
> `text(compression=null)`, so an SDK-created source showing `compression=lz4`
> will differ there. Storage size only; semantics and queries are unaffected.

> **Note:** a freshly created table takes a moment to become visible on every
> backend pod. `create_index` absorbs that window automatically (bounded retry
> on "does not have DeepLake access method" / "does not exist"), so
> `create_table(..., index=...)` is safe to call in one shot.

---

## Workspace Management

Workspaces are created via the REST API. The SDK clients don't have a built-in `createWorkspace()` method — use `apiRequest` directly.

**Important:** The `id` field is **required** when creating a workspace. Omitting it returns an error.

> **⚠️ Which org does the workspace land in?** `POST /workspaces` creates the
> workspace in the org the request is **scoped** to. That scope comes from the
> `X-Activeloop-Org-Id` header — and **only** that header. If you belong to
> multiple orgs and pass no org header (or a mistyped one like `X-Org-Id`), the
> workspace is silently created in the **token's embedded org**, not the one you
> meant — with **no error**. Always set `X-Activeloop-Org-Id: <target-org-uuid>`
> (or construct the client with `org_id=`, which adds it for you) when the target
> org differs from the token's org. Verify afterward with
> `GET /workspaces` (the `org_id` field on each row) before ingesting.

```typescript
// Node.js — create workspace via API
import { apiRequest, extractOrgId } from 'deeplake';

// extractOrgId reads the token's embedded org. Pass a DIFFERENT org UUID here to
// target another org — apiRequest sends it as X-Activeloop-Org-Id.
const orgId = targetOrgId ?? extractOrgId(token);
await apiRequest(apiUrl, token, orgId, {
    method: 'POST',
    path: '/workspaces',
    body: { id: 'my-workspace', name: 'My Workspace' },
    timeoutMs: 30_000,
});
```

```python
# Python — create workspace via API
import requests

resp = requests.post(
    f"{api_url}/workspaces",
    headers={"Authorization": f"Bearer {token}"},
    json={"id": "my-workspace", "name": "My Workspace"},
)
resp.raise_for_status()
```

**List workspaces:**

```typescript
// Node.js
const resp = await apiRequest(apiUrl, token, orgId, {
    method: 'GET',
    path: '/workspaces',
});
// resp.data = [{ id, org_id, name, type, created_at }, ...]
```

| Field    | Required | Description                                       |
| -------- | -------- | ------------------------------------------------- |
| `id`     | **Yes**  | Workspace identifier (used in API paths and `al://` URLs) |
| `name`   | No       | Display name (defaults to `id` if omitted)        |

> **Note:** Omitting `id` returns HTTP 400 Bad Request with the message "workspace ID is required".

---

## Error Handling

Both Python and Node.js share the same error hierarchy:

```
ManagedServiceError          # Base class for all errors
├── AuthError                # Token invalid/expired
│   └── TokenError           # Token parsing failed
├── CredentialsError         # DB credentials fetch failed
├── IngestError              # File ingestion failed
├── TableError               # Table operation failed
└── WorkspaceError           # Workspace not found or inaccessible
```

```python
# Python imports
from deeplake.managed import (
    ManagedServiceError, AuthError, CredentialsError,
    IngestError, TableError, TokenError, WorkspaceError,
)
```

```typescript
// Node.js imports
import {
    ManagedServiceError, AuthError, CredentialsError,
    IngestError, TableError, TokenError, WorkspaceError,
} from 'deeplake';
```

| Error                                      | Cause                     | Solution                                                    |
| ------------------------------------------ | ------------------------- | ----------------------------------------------------------- |
| `AuthError: Token required`                | No token provided         | Pass `token=` to Client() or set `DEEPLAKE_API_KEY` env var |
| `AuthError: Token does not contain org_id` | Token missing OrgID claim | Ensure token has OrgID claim or API `/me` is accessible     |
| `IngestError: File not found`              | Invalid file path         | Check file exists at given path                             |
| `TableError: table creation failed`        | API table creation failed | Check API server is running and workspace is accessible     |
| `WorkspaceError: No storage path`          | API returned no path      | Check workspace exists and has storage configured           |

> For troubleshooting details, see [reference.md](reference.md).

---

## Reliability & Retries

The managed backend is a shared, autoscaling pool. Individual pods restart,
scale up under load, and refresh storage credentials — so **transient errors are
normal** and a robust client must retry them. Build these behaviors into any
bulk copy / ETL loop.

### Retry transient HTTP statuses (with backoff)

| Status | Meaning | Client action |
| ------ | ------- | ------------- |
| **503** (often with `Retry-After`) | Backend scaling / under load / connect failure — the query never ran | Honor `Retry-After` (default ~2s); retry with exponential backoff |
| **504** | Query exceeded its deadline | Retry, or raise `timeout` / add a `LIMIT` |
| **429** | Rate limited | Honor `Retry-After` |
| **502 / 500** on a query | Should be rare; treat as transient and retry a bounded number of times | Retry with backoff; alert if it persists |

Reads are always safe to retry. For **writes**, retry only if the write is
idempotent (deterministic primary key + `ON CONFLICT DO NOTHING`, or a de-dupe
pass) — a 503 means the write didn't run, but a dropped connection *after* the
server committed is ambiguous, so idempotency is the safe default.

### ⚠️ Silent truncation: an empty page is NOT always end-of-data

The single most dangerous failure mode for a **paginating** client
(`LIMIT/OFFSET` or keyset loops): during a backend credential-refresh window a
page can come back **HTTP 200 with zero rows** even though more data exists. A
naive loop that treats "empty page ⇒ done" will **silently truncate** — e.g.
stop at 137k of 249k rows and report success. Guard against it:

- **Don't infer completion from a single empty page.** Cross-check the total
  first (`SELECT COUNT(*)`) and keep paging until you've read that many rows.
- Prefer **keyset pagination** (`WHERE id > $last ORDER BY id LIMIT n`) over
  `OFFSET`, and verify the last key advances.
- If a page is unexpectedly empty mid-stream, **re-request it** after a short
  backoff before concluding you're done — a genuine end-of-data page stays empty;
  a credential-refresh blip fills in on retry.
- After the copy, assert `rows_copied == COUNT(*)` and fail loudly on a mismatch.

### After ingest, verify the committed count

See [Ingestion](#ingestion) — `ingest()` returning is not proof of a commit.
Compare the returned `row_count` to what you sent, and reconcile a bulk load with
a final `COUNT(*)` (accounting for the visibility lag above).

---

## Agent Decision Trees

### Decision: How to Initialize Client

```
Need to create a Client
|
|-- Python?
|   |-- DEEPLAKE_API_KEY env var is set?
|   |   `-- client = Client()                          # defaults: token from env, workspace="default"
|   |-- Have explicit token?
|   |   `-- client = Client(token="dl_xxx")            # workspace defaults to "default"
|   |-- Need specific workspace?
|   |   `-- client = Client(workspace_id="my-ws")      # token from env
|   `-- Need custom API URL?
|       `-- client = Client(api_url="http://custom:8080")
|
`-- Node.js?
    `-- import { ManagedClient, initializeWasm } from 'deeplake';
        await initializeWasm();
        const client = new ManagedClient({
           token: process.env.DEEPLAKE_API_KEY!,
           workspaceId: "my-ws",        // optional, default "default"
           apiUrl: "http://custom:8080", // optional
        });
```

### Decision: How to Ingest Data

```
User wants to ingest data
|
|-- Is it local files? -> use FILE schema
|   |-- Python:
|   |   `-- client.ingest("table", {"path": ["f1.mp4", "f2.mp4"]},
|   |          schema={"path": "FILE"})
|   `-- Node.js:
|       `-- await client.ingest("table", { path: ["f1.mp4"] },
|              { schema: { path: "FILE" } })
|
|-- Is it structured data (dict/lists)? -> pass a dict directly
|   |-- Python:
|   |   `-- client.ingest("table", {"col1": [...], "col2": [...]})
|   `-- Node.js:
|       `-- await client.ingest("table", { col1: [...], col2: [...] })
|
|-- Is it a HuggingFace dataset? -> use _huggingface key
|   `-- client.ingest("table", {"_huggingface": "dataset_name"})
|
|-- Is it a LeRobot robotics dataset? -> 3-table design (tasks + frames + episodes)
|   |-- from deeplake.managed.formats import LeRobotTasks, LeRobotFrames, LeRobotEpisodes
|   |-- client.ingest("tasks", format=LeRobotTasks(dataset_dir))
|   |-- client.ingest("frames", format=LeRobotFrames(dataset_dir, chunk_start=0, chunk_end=3))
|   `-- client.ingest("episodes", format=LeRobotEpisodes(dataset_dir, chunk_start=0, chunk_end=3))
|   Note: chunk_end is inclusive. Episodes requires git lfs. See examples.md Workflow 7.
|
|-- Is it a domain-specific format (COCO, etc.)? -> use format object
|   |-- Python:  client.ingest("table", format=my_format)
|   `-- Node.js: await client.ingest("table", null, { format: myFormat })
|   See formats.md for built-in formats: CocoPanoptic, Coco, LeRobot, custom
|
|-- Need custom chunking for text?
|   `-- client.ingest("table", {"path": ["doc.txt"]},
|          schema={"path": "FILE"},
|          chunk_size=500, chunk_overlap=100)
|
`-- Need explicit schema?
|   `-- client.ingest("table", {...}, schema={
|          "name": "TEXT",
|          "count": "INT64",
|          "vector": "EMBEDDING",
|      })
|
|
|-- Need the table to exist BEFORE any data? (skeleton, INSERT…SELECT target)
|   `-- client.create_table("table", {"id": "TEXT", "embedding": "EMBEDDING(1536)"},
|          index={"id": "inverted_index", "embedding": "clustered"})
|      Use EMBEDDING(dim), not bare EMBEDDING — an empty table has no rows to
|      infer the dimension from.
|
`-- Need indexes for search performance?
    `-- client.ingest("table", {...}, index=["embedding", "text"])
       Explicit types:  index={"text": "bm25", "embedding": "clustered"}
       Or standalone:   client.create_index("table", "embedding",
                                            index_type="clustered")
       The text default is exact_text (equality only) — pass bm25 or
       inverted_index if that is what you actually want.
```

### Decision: Reproduce Another Dataset's Schema

```
Need a table with the same columns AND indexes as an existing dataset
|
|-- 1. Read the source (read-only — never take write access to inspect)
|      src = client.describe_table("source_table")
|      -> {col: {"type": "embedding(1536, clustered)", "index": "clustered"}}
|
|-- 2. Translate deeplake types -> schema types
|      "embedding(1536, ...)"  -> "EMBEDDING(1536)"   # keep the dimension!
|      "text (compression=…)"  -> "TEXT"
|      "int64"                 -> "INT64"
|
|-- 3. Carry each column's index type across verbatim
|      index = {c: i["index"] for c, i in src.items() if i["index"]}
|
`-- 4. client.create_table("skeleton", schema, index=index)
       Verify:  client.describe_table("skeleton") == src
       See examples.md Workflow 10 for the complete script.
```

### Decision: How to Query Data

```
User wants to query data
|
|-- Simple SELECT (small result)?
|   |-- Python fluent: client.table("table").select("id", "text").limit(100)()
|   |-- Python raw:    client.query("SELECT * FROM table LIMIT 100")
|   |-- Node fluent:   await client.table("table").select("id", "text").limit(100).execute()
|   `-- Node raw:      await client.query("SELECT * FROM table LIMIT 100")
|
|-- Large result set (streaming)?
|   `-- Use client.open_table("table", read_only=True) for direct dataset access
|      with batch iteration, PyTorch/TF DataLoaders, etc.
|
|-- Need semantic/vector search?
|   `-- from deeplake.managed import vector_literal
|      client.query(f"""
|          SELECT *, embedding::float4[] <#> {vector_literal(query_embedding)} AS score
|          FROM table ORDER BY score DESC LIMIT 10
|      """)
|      Through client.query() a vector CANNOT be passed as $1 -- the REST API
|      serializes it to a string that arrives at the DuckDB executor as
|      VARCHAR. Inline it. Scalars/text as $1 are fine. (On a direct psql /
|      asyncpg / psycopg connection $1::float4[] binds a real array and works.)
|
|-- Need text search?
|   `-- client.query("""
|          SELECT * FROM table
|          WHERE text @> $1
|      """, ("keyword",))
|
`-- Need hybrid search (vector + text)?
    `-- client.query(f"""
           SELECT *, (embedding, text)::deeplake_hybrid_record <#>
           deeplake_hybrid_record({vector_literal(query_emb)}, $1, 0.7, 0.3) AS score
           FROM table ORDER BY score DESC LIMIT 10
       """, ("search text",))
       Vector inlined; the text argument can stay a parameter.
```

### Decision: User Wants to Train on Data

```
User wants to train / iterate over data
|
|-- Fast native batch iteration (RECOMMENDED for large datasets)?
|   |-- ds = client.open_table("table", read_only=True)
|   |   for batch in ds.batches(256):  # dict of numpy arrays
|   |       states = torch.tensor(np.stack([batch[c] for c in cols], axis=1))
|   |-- For column subsets, use query first:
|   |   view = ds.query("SELECT col1, col2 WHERE episode_index < 100")
|   `   for batch in view.batches(256): ...
|
|-- Need PyTorch DataLoader (small datasets or need shuffle)?
|   `-- ds = client.open_table("table", read_only=True)
|      loader = DataLoader(ds.pytorch(), batch_size=32, shuffle=True)
|      NOTE: Slower on large remote datasets — prefer ds.batches() above
|
|-- Need TensorFlow tf.data?
|   `-- ds = client.open_table("table", read_only=True)
|      tf_ds = ds.tensorflow().batch(32).prefetch(AUTOTUNE)
|
`-- Training on LeRobot data?
    |-- Behavior cloning (state->action):
    |   ds = client.open_table("droid_frames", read_only=True)
    |   view = ds.query("SELECT state_x, ..., action_x, ... WHERE episode_index < 100")
    |   for batch in view.batches(256):
    |       states = torch.tensor(np.stack([batch[c] for c in STATE_COLS], axis=1))
    |       actions = torch.tensor(np.stack([batch[c] for c in ACTION_COLS], axis=1))
    |       # train(model, states, actions)
    |
    `-- Video-conditioned training:
        ds = client.open_table("droid_episodes", read_only=True)
        # Access video bytes via ds[i]["exterior_1_video"], etc.
```

### Decision: Error Recovery

```
Operation failed with error
|
|-- AuthError?
|   |-- "Token required" -> Set DEEPLAKE_API_KEY env var or pass token= to Client()
|   |-- "Token does not contain org_id" -> Ensure token has OrgID claim
|   `-- "Token expired" -> Get new token
|
|-- IngestError?
|   |-- "data must be a dict" -> Pass a dict, not list/str/int
|   |-- "data must not be empty" -> Dict must have at least one key
|   |-- "File not found" -> Check file path exists
|   |-- "ffmpeg not found" -> Install ffmpeg for video processing
|   `-- "fitz not found" / "pdfjs-dist not found" -> Install pymupdf (Python) or pdfjs-dist (Node.js)
|
|-- TableError?
|   |-- "create_deeplake_table failed" -> Check pg_deeplake extension
|   |-- "Table already exists" -> Use drop_table() first or different name
|   |-- "Index creation failed" -> Check column exists and is EMBEDDING or TEXT type
|   |-- "already has index ... does not match the requested" ->
|   |      One index per column. client.drop_index(table, col), then recreate.
|   |-- "Invalid index type 'X' for text column" ->
|   |      Use exact_text | bm25 | inverted_index (embeddings: clustered |
|   |      clustered_quantized | pooled_quantized)
|   |-- "does not have DeepLake access method" right after create_table ->
|   |      Backend pod hasn't caught up. create_index already retries; if you
|   |      hit it elsewhere, wait a few seconds and retry.
|   |-- "not float32 array, hence not supported yet for indexing" ->
|   |      Column is not an embedding. Declare it EMBEDDING(dim), or pass
|   |      dimension= when indexing a bare float4[] column.
|   `-- "Query timed out" -> Increase timeout: client.query(sql, timeout=300)
|
|-- ValueError: "Unknown index_type"?
|   `-- Valid: inverted_index, bm25, exact_text, clustered,
|      clustered_quantized, pooled_quantized, unique
|
|-- Skeleton table's column came back as TEXT instead of a vector?
|   `-- Schema said "EMBEDDING" (no dimension). Use "EMBEDDING(1536)".
|
|-- ReadOnlyDatasetModificationError / AttributeError on append?
|   `-- The handle came from open_table(..., read_only=True). Reopen without it.
|
|-- WorkspaceError / "workspace ID is required"?
|   `-- POST /workspaces requires the "id" field in the request body.
|      Use: { id: "my-ws", name: "My Workspace" }
|
|-- "Not found: /workspaces/.../tables"?
|   `-- Workspace must exist before ingest(). Create it via the API or UI first.
|
|-- Tables created via raw SQL not visible to ingest()?
|   `-- Raw SQL tables (CREATE TABLE ... USING deeplake) are not registered with
|      the managed API. Use client.ingest() to create tables. Use client.query()
|      for raw SQL operations on manually-created tables.
|
|-- Thumbnail generation failed? (logged as warning, non-fatal)
|   |-- Python: Install Pillow (`pip install Pillow`)
|   `-- Node.js: Install sharp (`npm install sharp`)
|
`-- ManagedServiceError?
    `-- Check API server is running at the configured api_url
```

---

## Supporting Files

- **[reference.md](reference.md)** -- pg_deeplake SQL reference (vector search, BM25, hybrid search, indexes), data types, LINK types & roundtrip, **attaching an existing SDK dataset (SECURITY LABEL physical_name + `deeplake_resync_table_pointer`)**, managed credentials (Azure/GCS) & `deeplake_set_creds_key()`, limits, performance tuning, troubleshooting
- **[examples.md](examples.md)** -- Complete end-to-end workflow examples and detailed ingestion examples
- **[formats.md](formats.md)** -- Format base class, custom format classes, normalize()/schema()/image_columns() rules
