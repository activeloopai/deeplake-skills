# Deeplake Managed Service -- Reference

## pg_deeplake SQL Reference

Once data is ingested, use these SQL features:

### Vector Similarity Search (Cosine)

The `<#>` operator is overloaded by pg_deeplake -- it performs **vector cosine similarity** when applied to embedding columns and **BM25 text similarity** when applied to text columns. In both cases, higher scores = more similar, so use `ORDER BY ... DESC`.

**Cast the vector column:** write `embedding::float4[] <#> ...`, with a cast on
both sides. `EMBEDDING(n)` is a domain over `float4[]`, and `<#>` is overloaded
three ways (vector, BM25 text, hybrid record); spelling the left operand out is
the form that resolves in every client, rather than relying on the domain being
smashed to its base type. The cast costs nothing -- it is a relabel, not a copy
-- and it does not stop the index from being used.

**Note:** Unlike pgvector's `<=>` (distance, lower = closer), pg_deeplake's `<#>` returns **similarity** (higher = more similar). Always use `ORDER BY ... DESC`.

```sql
-- <#> on embedding column: cosine similarity (higher = more similar)
SELECT id, text, embedding::float4[] <#> $query_embedding AS similarity
FROM documents
ORDER BY similarity DESC
LIMIT 10;
```

> **⚠️ Through `client.query()` a vector cannot be a query parameter — inline
> it.** Passing a list as `$1` fails: the REST query API serializes it to a
> PostgreSQL array *string*, which reaches the DuckDB executor typed `VARCHAR`,
> and DuckDB refuses to convert that to `FLOAT4[]`:
>
> ```
> Conversion Error: Type VARCHAR with value '{0.55,0.03,…}' can't be cast to FLOAT4[]
> ```
>
> Adding `$1::FLOAT4[]` or `CAST($1 AS FLOAT4[])` does **not** help — the cast
> applies to the VARCHAR DuckDB already declined. Use `vector_literal()` /
> `vectorLiteral()` to inline the vector instead. Scalar and text parameters are
> unaffected (`WHERE id = $1` and BM25's `text <#> $1` both work); only
> array-valued parameters are.
>
> This is specific to the REST query API. A **direct** PostgreSQL connection
> (psql, asyncpg, psycopg) binds a real `float4[]`, so `emb <#> $1::float4[]`
> works there — which is why the SQL guides show that form.

**Python example:**
```python
from deeplake.managed import vector_literal

query_embedding = model.encode("search query").tolist()

results = client.query(f"""
    SELECT id, text, embedding::float4[] <#> {vector_literal(query_embedding)} AS similarity
    FROM embeddings
    ORDER BY similarity DESC
    LIMIT 10
""")

for row in results:
    print(f"{row['similarity']:.3f}: {row['text']}")
```

```typescript
import { vectorLiteral } from 'deeplake';

const rows = await client.query(`
    SELECT id, text, embedding::float4[] <#> ${vectorLiteral(queryEmbedding)} AS similarity
    FROM embeddings ORDER BY similarity DESC LIMIT 10
`);
```

`vector_literal()` renders each value with `repr()` so the vector round-trips
exactly — a self-similarity query returns `1.0`. A 1536-d vector produces
roughly a 30 KB literal, which the query API accepts without trouble.

### BM25 Text Search

> **Important:** Use the `<#>` operator for BM25 search. The `BM25_SIMILARITY()` function form is **not supported** on the managed query endpoint and will return a 400 error.

```sql
-- <#> on text column: BM25 text similarity (higher = more relevant)
SELECT id, text, text <#> 'search query' AS score
FROM documents
ORDER BY score DESC
LIMIT 10;
```

**Python example:**
```python
results = client.query("""
    SELECT id, text, text <#> $1 AS score
    FROM documents
    ORDER BY score DESC
    LIMIT 10
""", ("machine learning",))
```

### Full-Text Contains

```sql
-- Check if text contains keyword
SELECT * FROM documents
WHERE text @> 'important keyword';
```

### Hybrid Search (Vector + Text)

```sql
-- Combine vector and text search with weights
-- deeplake_hybrid_record(embedding, text, vector_weight, text_weight)
-- Weights: 0.7 = 70% vector similarity, 0.3 = 30% BM25 text similarity
SELECT id,
       (embedding, text)::deeplake_hybrid_record <#>
       deeplake_hybrid_record(ARRAY[0.1, 0.2, ...]::FLOAT4[], 'search text', 0.7, 0.3) AS score
FROM documents
ORDER BY score DESC
LIMIT 10;
```

**Python example:**
```python
from deeplake.managed import vector_literal

query_emb = model.encode("neural networks").tolist()
results = client.query(f"""
    SELECT id, text,
           (embedding, text)::deeplake_hybrid_record <#>
           deeplake_hybrid_record({vector_literal(query_emb)}, $1, 0.7, 0.3) AS score
    FROM documents
    ORDER BY score DESC
    LIMIT 10
""", ("neural networks",))
```

> The vector argument must be inlined here too — `deeplake_hybrid_record($1, $2, …)`
> fails with the same `Conversion Error` described under
> [Vector Similarity Search](#vector-similarity-search). The **text** argument
> can stay a parameter.

### Create Indexes for Performance

```sql
-- Vector index (speeds up similarity search)
CREATE INDEX ON documents USING deeplake_index (embedding);

-- Text index (speeds up BM25 search)
CREATE INDEX ON documents USING deeplake_index (text) WITH (index_type = 'bm25');
```

**Via SDK (recommended):**

```python
# Python — standalone
client.create_index("documents", "embedding", index_type="clustered")
client.create_index("documents", "text", index_type="bm25")

# Python — during ingestion
client.ingest("documents", data, index={"embedding": "clustered", "text": "bm25"})
```

```typescript
// Node.js — standalone
await client.createIndex("documents", "embedding", { indexType: "clustered" });
await client.createIndex("documents", "text", { indexType: "bm25" });

// Node.js — during ingestion
await client.ingest("documents", data,
    { index: { embedding: "clustered", text: "bm25" } });
```

### Index Types

`WITH (index_type = '…')` selects which index pg_deeplake builds. The SDK's
`index_type=` / `indexType:` argument maps 1:1 onto this option, and the names
are the same ones `Dataset.summary()` prints in its `index=` suffix — so an
index type read off one dataset can be replayed onto another verbatim.

| `index_type`          | Valid on            | Enables                                                  |
| --------------------- | ------------------- | -------------------------------------------------------- |
| `exact_text`          | text                | Equality only: `WHERE col = 'x'` **(text default)**       |
| `bm25`                | text                | Full-text ranking: `text <#> 'query'`, `BM25_SIMILARITY`  |
| `inverted_index`      | text, numeric, json | Keyword / containment lookups: `CONTAINS(col, 'x')`       |
| `clustered`           | embedding (1-D)     | Cosine similarity `<#>` **(embedding default)**           |
| `clustered_quantized` | embedding (1-D)     | Same, binary-quantized — faster, slightly less accurate   |
| `pooled_quantized`    | embedding (2-D)     | ColBERT-style `MAXSIM` over an embeddings matrix          |
| `unique`              | any                 | Column-level uniqueness enforced at commit time           |

Aliases: `inverted` → `inverted_index`, `exact` → `exact_text`,
`quantized` / `binary_quantized` → `clustered_quantized`.

**Defaults when `index_type` is omitted** — text `exact_text`, 1-D embedding
`clustered`, 2-D embedding `pooled_quantized`, numeric/json `inverted_index`.

> **⚠️ The text default is `exact_text`, which is rarely what a source dataset
> carries.** `exact_text` serves only `col = 'x'`; it returns nothing for
> `CONTAINS` or BM25 ranking. Most datasets built through the SDK carry
> `inverted_index` or `bm25` on their text columns. If you are reproducing an
> existing dataset, read the source's types with `client.describe_table()` and
> pass them explicitly — a bare `create_index` will silently produce a
> *different* index with no error.

**`WITH (dimension = N)`** — supplies the vector dimension when building a
clustered index over a plain `float4[]` column. Only needed when that column has
no rows to infer the length from (i.e. an empty skeleton table). A real
`EMBEDDING(dim)` column already carries its dimension.

> **An `EMBEDDING(N)` column arrives with a `clustered` index already attached.**
> `CREATE TABLE … emb EMBEDDING(1536)` alone yields
> `embedding(1536, clustered, index=clustered)` — no `CREATE INDEX` required.
> Re-requesting `clustered` is a no-op, but requesting `clustered_quantized` on
> that column is refused (`Conflicting index types for column 'emb'`), because
> the index type is baked into the column type. For a quantized index, declare
> the column as plain `float4[]` (SDK `"EMBEDDING"`) and pass the dimension when
> creating the index.

```sql
CREATE INDEX idx_emb ON memories USING deeplake_index (content_embedding)
    WITH (index_type = 'clustered', dimension = 1536);
```

**One index per column.** pg_deeplake rejects a second index on an
already-indexed column, and `CREATE INDEX IF NOT EXISTS` is skipped by
PostgreSQL before pg_deeplake ever sees the conflicting type. To change a
column's index type, drop it first:

```python
client.drop_index("documents", "text")
client.create_index("documents", "text", index_type="bm25")
```

### Reading back which index a column has

The built index type is not exposed on `ColumnDefinition` — `dataset_view::summary()`
splices it into the printed type string as an `index=` suffix, and `summary()`
prints rather than returns. `client.describe_table()` / `client.describeTable()`
parse it back out:

```python
client.describe_table("documents")
# {'text':      {'type': 'text (compression=lz4)',     'index': 'bm25'},
#  'embedding': {'type': 'embedding(1536, clustered)', 'index': 'clustered'}}
```

The SQL catalog is a cheaper but *partial* alternative — it shows the requested
`WITH` options, not the built index, and says nothing when the index was created
with the default:

```sql
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'documents';
-- CREATE INDEX idx_documents_text ON "default".documents
--     USING deeplake_index (text) WITH (index_type=bm25)
```

### Raw SQL: Inserting Image Bytes

When inserting image data via raw SQL (not `client.ingest()`), use `decode()` with the `IMAGE` cast:

```sql
INSERT INTO images (id, img)
VALUES ('img_1', decode('89504e470d0a1a0a...', 'hex')::IMAGE);
```

```python
image_hex = image_bytes.hex()
client.query(
    "INSERT INTO images (id, img) VALUES ($1, decode($2, 'hex')::IMAGE)",
    ("img_1", image_hex)
)
```

### Raw SQL: Vector Literals

When using vector values as SQL literals (e.g. for benchmarking without parameterized queries):

```sql
SELECT id, text, embedding::float4[] <#> ARRAY[0.1, 0.2, 0.3]::FLOAT4[] AS score
FROM documents
ORDER BY score DESC
LIMIT 10;
```

> **Tip:** For production use, prefer parameterized queries (`$1`) over literals.

---

## Data Types Reference

| Schema Type    | Python Type | JS/TS Type        | Postgres Type             | Example                 |
| -------------- | ----------- | ----------------- | ------------------------- | ----------------------- |
| `TEXT`         | str         | string            | text                      | `"hello"`               |
| `INT32`        | int         | number            | integer                   | `42`                    |
| `INT64`        | int         | number            | bigint                    | `9999999999`            |
| `FLOAT32`      | float       | number            | real                      | `3.14`                  |
| `FLOAT64`      | float       | number            | double precision          | `3.14159265359`         |
| `BOOL`         | bool        | boolean           | boolean                   | `True`                  |
| `JSONB`        | dict/list   | object/array      | jsonb                     | `{"k": "v"}`            |
| `BINARY`       | bytes       | Buffer/Uint8Array | bytea                     | `b"\x00\x01"`           |
| `IMAGE`        | bytes       | Buffer/Uint8Array | IMAGE (bytea)             | Image binary data       |
| `VIDEO`        | bytes       | Buffer/Uint8Array | bytea                     | Video binary data       |
| `EMBEDDING`    | list[float] | number[]          | float4[] (variable length) | `[0.1, 0.2, 0.3]`      |
| `EMBEDDING(N)` | list[float] | number[]          | EMBEDDING(N) → `embedding(N, float32)` | `[0.1, …]` (exactly N) |
| `SEGMENT_MASK` | bytes       | Buffer/Uint8Array | SEGMENT_MASK (bytea)      | Segmentation mask data  |
| `BINARY_MASK`  | bytes       | Buffer/Uint8Array | BINARY_MASK (bytea)       | Binary mask data        |
| `BOUNDING_BOX` | list[float] | number[]          | BOUNDING_BOX (float4[])   | `[x, y, w, h]`          |
| `CLASS_LABEL`  | int         | number            | CLASS_LABEL (int4)        | Label index             |
| `POLYGON`      | list[ndarray] | number[][][]    | DEEPLAKE_POLYGON (float4[][][]) | Per-row list of polygons, shaped (P x N x 2|3) — matches `deeplake.types.Polygon()`. Changed from `bytea` in 1.12. |
| `POINT`        | list[float] | number[]          | DEEPLAKE_POINT (float4[]) | `[1.0, 2.0]`            |
| `MESH`         | bytes       | Buffer/Uint8Array | MESH (bytea)              | 3D mesh data (PLY, STL) |
| `MEDICAL`      | bytes       | Buffer/Uint8Array | MEDICAL (bytea)           | Medical imaging (DICOM) |
| `LINK(target)` | bytes       | Buffer/Uint8Array | LINK (bytea)              | Reference to external data (see [LINK Types](#link-types-and-roundtrip)) |
| `FILE`         | str (path)  | string (path)     | N/A (processed)           | `"/path/to/file.mp4"`   |

> **Note:** `EMBEDDING` and `EMBEDDING(N)` are **not** interchangeable.
> Bare `EMBEDDING` maps to a plain `float4[]` — a *variable-length* float array
> per row, which pg_deeplake stores as a generic array rather than a vector
> column. `EMBEDDING(N)` maps to the `EMBEDDING` domain with a typmod, which
> pg_deeplake resolves to a real `embedding(N, float32)` column.
>
> Both support `<#>` and `deeplake_index`, but they differ where it matters:
> a clustered index on a bare `float4[]` column has to *infer* the dimension
> from existing rows, so on an **empty** table it fails unless you pass
> `dimension=`. Prefer `EMBEDDING(N)` whenever you know the dimension —
> especially when creating a table before loading data.
>
> **Unknown type names pass through to PostgreSQL verbatim.** `SMALLINT`,
> `TEXT[]`, `NUMERIC(10,2)` and friends all work in `schema=`. (Previously
> anything outside the table above silently became `TEXT`, so
> `"EMBEDDING(1536)"` produced a text column with no error.)
>
> **Note:** `FILE` is a schema directive, not a storage type. Columns marked as `FILE` are treated as file paths during ingestion -- the files are processed (chunked, etc.) and the resulting data is stored in generated columns. The `FILE` column itself is not stored in the dataset.
>
> **Note:** Domain types like `IMAGE`, `SEGMENT_MASK`, `BINARY_MASK`, `BOUNDING_BOX`, `CLASS_LABEL`, `DEEPLAKE_POLYGON`, `DEEPLAKE_POINT`, `MESH`, and `MEDICAL` are PostgreSQL domain types defined by pg_deeplake. They behave like their base types (bytea, float4[], int4) but carry semantic meaning for visualization, search, and type-aware processing.
>
> **Important -- IMAGE columns for UI display:** To display images correctly in the UI, explicitly set `schema={"image_col": "IMAGE"}` during ingestion. Without this, bytes columns are stored as generic `BINARY` and the UI will not render them as images.
>
> **Important -- IMAGE query results:** IMAGE columns returned via `client.query()` may come back as **base64-encoded strings** rather than raw bytes, depending on the backend serialization. Always handle both types:
> ```python
> import base64
> val = row["image"]
> if isinstance(val, str):
>     image_bytes = base64.b64decode(val)
> else:
>     image_bytes = val
> ```

**Schema inference:**
- `bool` / `boolean` -> BOOL
- `int` / `number` (integer) -> INT64
- `float` / `number` (decimal) -> FLOAT64
- `bytes` / `Buffer` -> BINARY
- `str` / `string` -> TEXT
- `list[float]` / `number[]` -> EMBEDDING (size auto-detected)

> **`JSONB` columns:** pass `schema={"col": "JSONB"}` explicitly (dict/list values
> are not inferred as `jsonb`). Querying is subject to the DuckDB dialect split —
> use `json_*` functions, not the PG `jsonb_*` aliases, and cast when needed:
> `json_array_length(col::json)`. See [SQL Dialect](#sql-dialect-postgres-wire-duckdb-execution).

---

## SQL Dialect: Postgres wire, DuckDB execution

Queries against deeplake (pg_deeplake) tables are **parsed as PostgreSQL but
executed by a DuckDB engine**. Most SQL is identical, but where the dialects
diverge the DuckDB layer shows through — sometimes in error text that names
DuckDB constructs. Known caveats:

| Area | PG form that fails | Use instead |
| ---- | ------------------ | ----------- |
| JSON length | `jsonb_array_length(col)` → *"did you mean json_array_length"* | `json_array_length(col::json)` |
| JSON aliases | `jsonb_*` family generally | `json_*` family (cast `jsonb`→`json`) |
| JSON access | `col->>'key'` works, but errors echo DuckDB `'$.key'` path syntax | `json_extract(col, '$.key')` for clarity |
| Column type change | `ALTER TABLE … ALTER COLUMN … TYPE …` → *"VACUUM FULL is not supported for deeplake tables"* | Create a new table with the target schema and `INSERT … SELECT` to copy |

**Rule of thumb:** advertise/write **`json_*`** (DuckDB-native) rather than
`jsonb_*`, and cast `jsonb` values to `json` before applying JSON functions.

---

## LINK Types and Roundtrip

`LINK` is a parameterized PostgreSQL domain over `bytea` that stores a **reference** to
external data (in S3/GCS/Azure) instead of the bytes themselves. The data is fetched
lazily at read time, with credentials resolved via the backend API
(see [Setting the Credentials Key](#setting-the-credentials-key)).

**Available since pg_deeplake 1.5.** Earlier versions (1.4) shipped a bare `LINK` domain only.

### Targets

`LINK(target)` declares what kind of data the reference points to. Valid targets:

```
IMAGE, VIDEO, FILE, SEGMENT_MASK, MEDICAL, AUDIO, MESH, BYTES
```

Bare `LINK` (no target) defaults to `BYTES` semantics. An unknown target (e.g.
`LINK(UNKNOWN)`) is rejected at `CREATE TABLE` / `ALTER TABLE` time.

> **Note:** `AUDIO` is valid **only** as a LINK target — there is no standalone `AUDIO`
> domain type. The other targets mirror the inline domain types of the same name.

```sql
CREATE TABLE media (
    id    INT,
    photo LINK(IMAGE),
    clip  LINK(VIDEO),
    scan  LINK(MEDICAL),
    raw   LINK            -- bare LINK == LINK(BYTES)
) USING deeplake;

-- Add a LINK column to an existing table
ALTER TABLE media ADD COLUMN sound LINK(AUDIO);
```

### Roundtrip behavior

LINK columns do **not** roundtrip as raw bytes. The write side and read side differ:

| Operation | Behavior |
|-----------|----------|
| **Write** (`INSERT`/`UPDATE`) | Cast bytes to the domain: `'\xDEADBEEF'::LINK`. NULL is allowed. |
| **Read** (`SELECT`) | Returns a serialized **data reference**, not the bytes: `DLREF:version\|column_id\|row_id` |
| **NULL** | A NULL link reads back as NULL. |

```sql
INSERT INTO media (id, photo) VALUES (1, '\x89504E47...'::LINK);

SELECT photo FROM media WHERE id = 1;
-- => 'DLREF:3|7|1'   (NOT the image bytes)
```

The `DLREF:` value encodes `version | column_id | row_id`. SDKs detect the `DLREF:`
prefix and fetch the underlying object directly from storage using the table's resolved
credentials. Within a single row all LINK columns share the same `version` and `row_id`
but each has a distinct `column_id`.

### How other types roundtrip

The `DLREF:` indirection is shared with the inline `VIDEO` domain. Other media types
return their bytes inline:

| Type | `SELECT` returns |
|------|------------------|
| `LINK(target)`, `VIDEO` | `DLREF:version\|column_id\|row_id` reference (SDK fetches bytes) |
| `IMAGE` | Image bytes inline — **may be base64-encoded string** depending on backend serialization (decode if `isinstance(val, str)`) |
| `SEGMENT_MASK`, `BINARY_MASK`, `MEDICAL`, `MESH`, `BINARY` | Bytes inline |
| `DEEPLAKE_POLYGON` | `float4[][][]` array inline — list of polygons, each (N x 2|3) |
| `EMBEDDING`, `BOUNDING_BOX`, `DEEPLAKE_POINT` | `float4[]` array inline |
| Scalars (`TEXT`, `INT*`, `FLOAT*`, `BOOL`, `CLASS_LABEL`) | Native value inline |

---

## Attach an Existing Dataset

A pg_deeplake table can be pointed at an existing on-disk deeplake dataset
created out-of-band by the SDK (`deeplake.create`). The pg table acts as a
SQL view over the existing typed columns — **no data rewrite, no codec
restriction**. Works for any dtype × compression combination the SDK
supports (`SegmentMask(UInt8())`, `SegmentMask("uint16", sample_compression="lz4")`,
`png`/`zlib`/`null`, etc.) because the SDK's decode pipeline runs first and
pg receives the decoded bytes.

### The two-step attach

```sql
-- 1. Register a pg table with the column types you want SQL to see.
--    Use the matching domain per column: SEGMENT_MASK / BINARY_MASK /
--    MEDICAL / MESH / DEEPLAKE_POLYGON / etc.
CREATE TABLE my_attached (
    smask      SEGMENT_MASK,
    lz4_smask  SEGMENT_MASK,
    zlib_smask SEGMENT_MASK,
    png_smask  SEGMENT_MASK
) USING deeplake;

-- 2. Retarget the table to the existing dataset's directory on storage.
--    physical_name is the *last segment* of the s3/gcs path; the rest
--    (bucket/org_id/workspace) is derived from the pg session.
SECURITY LABEL FOR deeplake ON TABLE my_attached
  IS 'physical_name=levon_testing_type_withour_compression_1';

-- 3. Read. pg returns the SDK-decoded sample bytes for every codec.
SELECT smask, lz4_smask, zlib_smask, png_smask FROM my_attached;
```

The first read after the retarget triggers an internal fall-back-to-open
that adopts the existing on-disk schema. The pg catalog still reports a
single domain per column (e.g. all four columns above show
`data_type=bytea / domain_name=segment_mask`) — codecs are an
on-disk implementation detail, not a wire-format distinction.

### What attaches cleanly today

| Pg domain          | SDK source                                                       |
|--------------------|------------------------------------------------------------------|
| `SEGMENT_MASK`     | `SegmentMask(UInt8/UInt16, sample_compression=null/lz4/zlib/png)` |
| `BINARY_MASK`      | `BinaryMask(sample_compression=null/lz4)` only                   |
| `MEDICAL`          | `Medical()` (DICOM/NIfTI pass-through)                           |
| `MESH`             | `Mesh()` (PLY/STL pass-through)                                  |
| `DEEPLAKE_POLYGON` | `Polygon()`                                                       |

For typed numeric columns (uint8/uint16 arrays the pg side declares as
`BYTEA`-family domains), reads work; writes through pg still expect the
BYTEA wire shape (raw bytes), so use the SDK for typed-array DML on the
same dataset.

### Limitations

- **BinaryMask is null/lz4 only.** zlib/png are rejected at the SDK
  layer (`InvalidBinaryMaskCompression`). If a client's "binary mask"
  data is PNG-compressed, it was actually declared as `SegmentMask`.
- **PNG channel limit.** PNG can't losslessly store 5-channel uint8
  arrays. A `SegmentMask(UInt8(), sample_compression="png")` over
  `(H, W, 5)` silently drops to `(H, W, 4)` on encode (PIL/PNG-format
  limitation). pg returns the same 4-channel result the SDK gives it.
- **BinaryMask reads back as `numpy.bool_`** through the SDK (not
  uint8). The underlying byte buffer matches uint8 0/1, so `pg → bytes`
  agrees with `SDK → arr.tobytes()`.

### `deeplake_resync_table_pointer(tbl regclass)`

Available in pg_deeplake 1.13+. Use after the underlying deeplake
dataset is mutated out-of-band (SDK `deeplake.create` / `add_column` on
the same path from a side process) and pg scans start failing with
`schema drift on table … pg projects column index N but deeplake
dataset has only M columns`. Evicts the cached per-backend snapshot so
the next access re-opens the dataset at HEAD:

```sql
SELECT deeplake_resync_table_pointer('my_attached'::regclass);
SELECT * FROM my_attached;   -- now sees the new HEAD
```

Refuses to evict when the table has unapplied DML — flush first via
`SELECT deeplake_flush_table('my_attached'::regclass)`. Refuses on
non-deeplake tables.

### Via the managed API

The `Client.create_table(...)` + `Client.query("SECURITY LABEL …")` pair from
`deeplake.managed.Client` is the canonical way to drive this from Python for a
fleet of client datasets:

```python
client = Client(token=TOKEN, api_url=API_URL,
                workspace_id="default", org_id=ORG_ID)
deeplake.client.endpoint = API_URL

client.create_table("my_attached", {
    "smask": "SEGMENT_MASK",
    "lz4_smask": "SEGMENT_MASK",
    "zlib_smask": "SEGMENT_MASK",
    "png_smask": "SEGMENT_MASK",
})
client.query("SECURITY LABEL FOR deeplake ON TABLE my_attached "
             "IS 'physical_name=levon_testing_type_withour_compression_1'")
rows = client.query("SELECT smask, lz4_smask, zlib_smask, png_smask "
                    "FROM my_attached")
```

> Older revisions of this guide reached for the private
> `client._create_table_via_api(...)` because no public equivalent existed.
> `create_table()` supersedes it — same REST call, plus schema-type resolution,
> alphabetical column ordering (which the DuckDB scanner requires), and
> optional index creation.

For migration scripts that loop over many existing client datasets, see
the dedicated skill `pg-deeplake-attach-typed-datasets`, which has the
full playbook (dataset listing → schema inspection → dtype-kind →
pg-domain mapping → registration → retarget → verify) plus the canonical
beta probe script.

---

## Managed Credentials

LINK columns and `al://` datasets read/write data in your own cloud bucket. Deeplake
resolves access through **managed credentials** configured in the platform UI
(**Workspace → Managed Credentials**) and applied per workspace (resolution order:
workspace credential → org default → environment default). The setup is a UI wizard
that alternates with a couple of cloud-CLI commands; full walkthroughs:

- **AWS** — https://docs.deeplake.ai/latest/guide/aws/
- **Azure** — https://docs.deeplake.ai/latest/guide/azure/
- **GCS** — https://docs.deeplake.ai/latest/guide/gcs/

### Supported credential types

`storage_type` is the discriminator on the credential record; it picks which generator
mints the temp creds returned to indra at read time. Pick the rightmost column when
production / no-stored-secret is the goal.

| `storage_type`     | Identity model                                                                                                          | Workload-identity / no-key? | When to use                                                                                |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- | --------------------------- | ------------------------------------------------------------------------------------------ |
| `s3`               | Long-term AWS access keys (or pre-issued STS triplet — passed through as-is).                                           | No                          | Fastest setup; paste keys.                                                                 |
| `s3:role`          | Customer IAM role ARN; deeplake-api uses stored AWS keys to `AssumeRole` (with optional session policy scoping path).   | Partial (role-based)        | Cross-account access without sharing the customer's user keys; key rotation still needed.  |
| `gcs`              | Service-account JSON key (full key in the credential record).                                                           | No                          | Fastest GCS setup; SA-key download → paste.                                                |
| `gcs_federated`    | **GCS Workload Identity Federation.** Per-cred GCP SA auto-provisioned in Deeplake's project; customer grants it bucket access. | **Yes**                     | Recommended for production GCS — no stored secret.                                         |
| `azure`            | Storage account + shared key.                                                                                           | No                          | Fastest Azure setup.                                                                       |
| `azure_federated`  | **Azure WI / OIDC federation.** Multi-step (Graph app install in customer tenant → tenant ID → storage account → access probe). | **Yes**                     | Recommended for production Azure — no stored secret; uses customer's existing SP/MI.       |

All six types resolve through the same `/internal/orgs/{org}/workspaces/{ws}/storage-creds`
path when a pg pod fetches workspace creds — `GenerateInternal` dispatches to the
matching generator (`gen_s3.go`, `gen_s3_role.go`, `gen_gcs.go`, `gen_gcs_federated.go`,
`gen_azure_federated.go`), so federated credentials work end-to-end without any
special-casing at the pod.

### Workload identity (customer-side, condensed)

Three of the six types above are **federation-based** — the customer never stores a
long-term secret with Deeplake. Use them in production:

| Type              | What customer creates                                                                | What Deeplake provisions / does                                                              |
| ----------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `gcs_federated`   | An IAM binding on the bucket granting the Deeplake-provisioned SA `objectAdmin`.    | A per-credential GCP SA in our project; impersonates via WIF when minting creds.            |
| `azure_federated` | Installs our Graph app in their tenant + grants the SP `Storage Blob Data Contributor`. | An AAD app registration with an OIDC federated credential; exchanges via tenant trust.       |
| `s3:role`         | An IAM role with a trust policy listing Deeplake's role ARN.                         | (today) Stored AWS keys do the `AssumeRole`; true OIDC-federated `s3_federated` is a planned addition. |

Full walkthroughs live in the per-cloud sections below.

### AWS (condensed)

Base path format: `s3://<bucket>/<optional-prefix>` (bucket must already
exist). Three paths, in increasing production-readiness:

**Path A — Long-term IAM user keys** (`s3` storage_type; fastest,
dev-only): create an IAM user with bucket-scoped policy, generate an
access key, and paste `access_key_id` + `secret_access_key` into the
wizard. Optional `session_token` field for pre-issued STS triplets
(rotate before the triplet expires).
```bash
aws iam create-user --user-name deeplake-storage
aws iam put-user-policy --user-name deeplake-storage \
  --policy-name deeplake-bucket-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::<BUCKET>", "arn:aws:s3:::<BUCKET>/*"]
    }]
  }'
aws iam create-access-key --user-name deeplake-storage
```

**Path B — AssumeRole** (`s3:role`, production-recommended today, no
long-term customer-account secret stored on Deeplake's side): create an
IAM role in your account with a trust policy listing Deeplake's runtime
role ARN, grant the role bucket-scoped permissions, and paste the role
ARN into the wizard. Deeplake-api uses its own stored credentials to
issue `sts:AssumeRole`; the resulting STS triplet is what the pg pods
receive. The `external_id` field (standard confused-deputy mitigation
for cross-account AssumeRole) is required.
```bash
# Trust policy — allow Deeplake's runtime role to assume yours.
# The wizard pre-fills <DEEPLAKE_ACCOUNT_ID>, <DEEPLAKE_RUNTIME_ROLE>,
# and the suggested <EXTERNAL_ID> so you copy-paste verbatim.
cat > trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::<DEEPLAKE_ACCOUNT_ID>:role/<DEEPLAKE_RUNTIME_ROLE>" },
    "Action": "sts:AssumeRole",
    "Condition": { "StringEquals": { "sts:ExternalId": "<EXTERNAL_ID>" } }
  }]
}
EOF
aws iam create-role --role-name deeplake-customer-access \
  --assume-role-policy-document file://trust-policy.json

# Same bucket-scoped policy as Path A, but attached to the role.
aws iam put-role-policy --role-name deeplake-customer-access \
  --policy-name deeplake-bucket-access \
  --policy-document file://bucket-policy.json
```
Optional session policy scoping in the wizard further restricts the
minted STS triplet to a sub-prefix of the bucket; useful when the same
role serves multiple Deeplake workspaces.

**Path C — `s3_federated` (OIDC, planned)**: when shipped, the model
will mirror `gcs_federated` / `azure_federated`: a true OIDC web
identity trust between your role and Deeplake's per-credential OIDC
issuer, with zero stored secrets on either side. Track readiness in the
Managed Credentials UI; no action needed until it appears as a wizard
option. Until then, `s3:role` is the no-stored-customer-secret path.

**Verify any credential** by creating a smoke-test table — data lands
under `<base_path>/<org_id>/<workspace_id>/<table>/`:
```python
client = deeplake.Client(token="<token>", workspace_id="default")
client.query('CREATE TABLE "smoke_test" ("id" BIGINT, "name" TEXT) USING deeplake', timeout=60)
client.query("INSERT INTO smoke_test VALUES (1, 'hello'), (2, 'world')", timeout=60)
print(client.query("SELECT * FROM smoke_test"))
```

### Azure (federated, condensed)

Base path format: `az://<storage_account>/<container>` (container must already exist).

1. In the UI, **Add credential → Azure**, set name + base path. The wizard generates an
   `<APP_ID>`.
2. Install the service principal in your tenant — **save the output's `id` (object ID)**:
   ```bash
   az ad sp create --id <APP_ID>
   ```
3. Submit your tenant ID in the wizard (UUID format).
4. Grant `Storage Blob Data Contributor` on the storage account scope:
   ```bash
   ACCOUNT_ID="$(az storage account show --name <ACCOUNT_NAME> --query id -o tsv)"
   az role assignment create \
     --assignee-object-id <SP_OBJECT_ID> \
     --assignee-principal-type ServicePrincipal \
     --role "Storage Blob Data Contributor" \
     --scope "${ACCOUNT_ID}"
   ```
5. Submit storage details (subscription ID, resource group, account, container) and
   **Verify access**. The credential moves to `verified`.

### GCS (condensed)

Base path format: `gs://<bucket>/<optional-prefix>`. Two paths:

**Path A — Service Account Key** (fastest): create an SA, grant
`roles/storage.objectAdmin` on the bucket, generate a JSON key, and paste it into the
wizard.
```bash
gcloud iam service-accounts create deeplake-storage --display-name="Deeplake managed storage"
gcloud storage buckets add-iam-policy-binding "gs://<BUCKET>" \
  --member="serviceAccount:deeplake-storage@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
gcloud iam service-accounts keys create deeplake-sa-key.json \
  --iam-account="deeplake-storage@<PROJECT_ID>.iam.gserviceaccount.com"
```

**Path B — Workload Identity Federation** (production, no stored key): select
**GCS Federated** in the wizard; Deeplake provisions a per-credential service account,
then run the wizard's pre-filled binding:
```bash
gcloud storage buckets add-iam-policy-binding gs://<BUCKET> \
  --member="serviceAccount:c-<id>@activeloop-saas-iam.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

**Verify any credential** by creating a smoke-test table — data lands under
`<base_path>/<org_id>/<workspace_id>/<table>/`:
```python
client = deeplake.Client(token="<token>", workspace_id="default")
client.query('CREATE TABLE "smoke_test" ("id" BIGINT, "name" TEXT) USING deeplake', timeout=60)
client.query("INSERT INTO smoke_test VALUES (1, 'hello'), (2, 'world')", timeout=60)
print(client.query("SELECT * FROM smoke_test"))
```

---

## Setting the Credentials Key

`deeplake_set_creds_key()` binds a table's LINK columns to a named managed credential
so reads can dereference the external data the links point to. It does two things: it
**resolves** the named credential to live storage credentials and injects them as the
table's links reader for the current session, and it **persists** the key on the dataset
so later opens re-resolve and re-inject automatically.

```sql
-- deeplake_set_creds_key(tbl regclass, creds_key text, token text DEFAULT NULL)
SELECT deeplake_set_creds_key('media'::regclass, 'prod-azure-bloblake');
```

```python
# via the SDK query interface
client.query("SELECT deeplake_set_creds_key($1::regclass, $2)", ("media", "prod-azure-bloblake"))
```

- `tbl` must be a DeepLake table (`USING deeplake`); calling it on a regular table
  errors with `not a DeepLake table`.
- `creds_key` is the name of a credential configured in **Managed Credentials** for the
  org. If it isn't bound to the org (or the backend endpoint isn't reachable) the call
  errors with `could not resolve creds_key '<key>'`.
- Returns `void`. Once set, reads that materialize LINK bytes resolve through that
  credential; without it the `DLREF:` reference still returns but the byte fetch fails.

**Managed tables resolve the key without a user token.** pg_deeplake tables open by raw
`s3://`/`az://`/`gs://` path and are *not* "connected to the Activeloop platform"
(no `ds_name`), so the legacy token-based path would throw
`not connected to Activeloop platform`. Instead the extension resolves the key from its
own pg context — **database name = org, schema = workspace** — against an internal,
service-authed backend endpoint, exchanges it for scoped storage credentials, and builds
a self-refreshing reader (it re-exchanges through the same endpoint when temporary STS
creds expire). The third `token` argument is **legacy and ignored** for managed tables;
no session token is required. This resolution requires the extension's `DEEPLAKE_API_URL`
to be configured (it is, in the managed deployment).

---

## Workload Identity Authentication

Distinct from the *storage* federation in the [Supported credential types](#supported-credential-types)
table — this section is about **authenticating to deeplake-api itself** with a
cloud-signed OIDC token instead of a Deeplake API token. Three layers cooperate.

### 1. Server-side default — `ACTIVELOOP_AUTH_PROVIDER`

A pod (pg_deeplake, indra-backed services) picks its own auth-provider class at
startup from this env var:

| Value                      | indra provider             | Token source                                                                  |
| -------------------------- | -------------------------- | ----------------------------------------------------------------------------- |
| *(unset)* / `activeloop`   | `activeloop_auth_provider` | Pass through whichever bearer the caller supplied.                            |
| `azure`                    | `azure_auth_provider`      | `ChainedTokenCredential` — `WorkloadIdentityCredential` first (`AZURE_FEDERATED_TOKEN_FILE`), then env, then MSI. |
| `aws`                      | `aws_auth_provider`        | AWS SDK default chain — picks up IRSA, instance profile, env.                |
| `gcp`                      | `gcp_auth_provider`        | GCP ADC / Workload Identity Federation.                                       |

Used for every outbound call indra makes to deeplake-api (`storage_creds`,
`get_dataset_credentials`, etc.). The Azure chain ordering is **intentional** — WI
goes before `EnvironmentCredential` because the latter errors loudly without
`AZURE_CLIENT_SECRET`; don't reorder it.

### 2. Org admin — register the workload identity

Once registered, the workload's OIDC token authenticates it *as* a member of the
org (no API token needed). Today: Azure SPs; AWS / GCP reserved.

```
POST   /organizations/{org_id}/workload-identities          (admin)  → 201
GET    /organizations/{org_id}/workload-identities          (member) → 200
GET    /organizations/{org_id}/workload-identities/{id}     (member) → 200
DELETE /organizations/{org_id}/workload-identities/{id}     (admin)  → 204
```

Register body:
```json
{
  "name": "prod-aks-cluster",
  "workload_identity_data": {
    "type": "azure",
    "azure_client_id": "<uuid>",
    "azure_tenant_id": "<uuid>"
  }
}
```

The middleware on every endpoint detects Azure-issued tokens by `iss`, verifies
signature against the issuer's JWKS, looks up the WI by `(appid, tid)`, and binds
the request to the registered identity's org. Registered WIs are also granted the
FGA `member` relation on the org so existing authz checks pass.

**The same Azure SP can be registered in more than one org** — uniqueness is
scoped per `(org_id, azure_client_id, azure_tenant_id)`, not global. When that
happens the auth middleware needs help picking which org to bind the request
to: the client sends `X-Activeloop-Org-Id: <uuid>` alongside the Azure token,
and the lookup matches the exact 3-tuple. When the header is absent and the SP
is registered in exactly one org, the unique row is returned automatically. When
the header is absent and >1 orgs claim it, the request fails:

```
HTTP 401
workload identity matches multiple orgs; X-Activeloop-Org-Id required
```

Same SP registered in the *same* org twice is still a `409 CONFLICT` — the
per-org unique index catches it.

#### SDK contract: when is `org_id` required?

To keep this from showing up as a runtime regression the day a customer
registers their SP in a second org, the Python managed `Client` /
`AsyncClient` requires `org_id` **at construction time** whenever
`auth_provider` is set to a workload-identity value. The split:

| `auth_provider`              | `org_id` source                                          | What happens if missing                |
| ---------------------------- | -------------------------------------------------------- | -------------------------------------- |
| default (API token)          | extracted from JWT `org` claim                           | nothing — works as before              |
| `azure` / `gcp` / `aws` (WI) | **`org_id=` kwarg** OR `ACTIVELOOP_ORG_ID` env var       | `AuthError` raised by `Client.__init__` |

```python
# WI mode — org_id required, either as kwarg…
c = deeplake.Client(auth_provider='azure', org_id='b7f45e10-…')

# …or via env (handy for shell-level multi-org switching)
# export ACTIVELOOP_ORG_ID=b7f45e10-…
c = deeplake.Client(auth_provider='azure')

# Missing both → AuthError("org_id required when auth_provider='azure'. …")
c = deeplake.Client(auth_provider='azure')  # AuthError
```

The Node/TypeScript `ManagedClient` (`typescript/node/src/client.ts`) enforces
the same contract:

```ts
// WI mode — orgId required, either as option…
const c = new ManagedClient({
  token: azureWiToken,
  authProvider: 'azure',
  orgId: 'b7f45e10-…',
});

// …or via env
// process.env.ACTIVELOOP_ORG_ID = 'b7f45e10-…';
const c = new ManagedClient({ token: azureWiToken, authProvider: 'azure' });

// Missing both → AuthError on construction.
```

The resolved value is sent two ways:
1. As `X-Activeloop-Org-Id` on every Python/JS HTTP call (handled in Python's
   `_api_request` and JS's `apiRequest`).
2. Pushed into the `ACTIVELOOP_ORG_ID` (and `ACTIVELOOP_AUTH_PROVIDER`)
   process env around `deeplake.open(...)` / `deeplake.open_async(...)` /
   `deeplakeSetEndpointAndOpen(...)` via `_push_org_id_env()` (Python) /
   `_withProviderEnv()` (Node) so the C++ HTTP layer
   (`cpp/backend/client.cpp` `get_dataset_credentials`) sends the same
   header on its outbound `/api/org/{ws}/ds/{ds}/creds` request. Without
   that bridge, the al:// path only carries the workspace name and the
   server can't disambiguate.

   Browser (no-Node) builds of the JS SDK skip the env-var push because
   `process.env` isn't available — they rely on the WASM client picking up
   any disambiguator from the JS-issued HTTP header path instead.

### 3. Client-side hint — `authProvider` / `X-Activeloop-Auth-Provider`

The SDK signals "this token isn't an API token, route it through provider X" so
the server can pick the matching auth path **per request** without changing the
pod's env. Resolution: explicit option → client-machine env var → none.

```python
# Python — explicit, or fall back to ACTIVELOOP_AUTH_PROVIDER env on the client.
client = DeepLakeBackendClient(token=azure_wi_token, auth_provider="azure")
```

```ts
// Node — same; browser builds skip the env fallback.
const client = new ManagedClient({
  token: azureWiToken,
  authProvider: "azure",
});
```

On every outbound request the SDK stamps:
```
X-Activeloop-Auth-Provider: azure
```
deeplake-api's `ExecuteQuery` handler reads the header and applies it to the
pg connection via `SELECT set_config('deeplake.auth_provider', 'azure', false)`
just before running the user's query (and resets in a deferred block so the
pooled conn doesn't leak the setting). pg_deeplake's `assign_hook` on that GUC
mirrors the value into indra's `auth_provider_type_override_slot` — a
thread-local override read by `backend::client`'s outbound calls for the
duration of the query, with `auth_provider_type_from_env()` as the fallback
when no hint arrives.

Net effect: pod startup is unchanged; the user's WI token routes through the
matching provider end-to-end. Empty/absent hint → server uses the pod's
`ACTIVELOOP_AUTH_PROVIDER` default.

---

## Thumbnail Table Schema

When a format declares `image_columns()`, the SDK auto-generates a shared `thumbnails` dataset at `{root_path}/thumbnails` with this schema:

| Column        | Type   | Description                                  |
| ------------- | ------ | -------------------------------------------- |
| `file_id`     | TEXT   | UUID of the source row (`_id` column value)  |
| `column_name` | TEXT   | Name of the IMAGE column (e.g. `"image"`)    |
| `dimension`   | TEXT   | Thumbnail size (e.g. `"32x32"`, `"256x256"`) |
| `content`     | BINARY | JPEG thumbnail bytes (quality 85)            |

**Sizes generated:** 32x32, 64x64, 128x128, 256x256 (aspect-preserving via `Image.thumbnail()` in Python, `sharp.resize({ fit: 'inside' })` in Node.js).

---

## Limits

| Resource                     | Limit           |
| ---------------------------- | --------------- |
| Video chunk duration         | 10 seconds      |
| Text chunk size (default)    | 1000 characters |
| Text chunk overlap (default) | 200 characters  |
| PDF rendering resolution     | 150 DPI (configurable via `pdf_dpi`) |
| Batch size (data ingest)     | 1000 rows       |
| Write buffer (flush_every)   | 200 rows        |
| Commit interval              | 2000 rows       |
| File normalization workers   | 4 threads       |
| Storage I/O concurrency      | 32 operations   |

---

## Performance Tuning

The SDK uses several optimizations to handle large-scale ingestion efficiently:

**Buffered writes:** Instead of calling `ds.append()` for each small batch (e.g., one image or text chunk), rows are accumulated in a memory buffer and flushed in larger batches. This reduces Python-to-C++ FFI overhead (or JS-to-WASM overhead in Node.js).

```python
# Default: flush every 200 rows, commit every 2000 rows
client.ingest("table", files)

# These are fixed internal defaults (not configurable via ingest()):
# flush_every=200   -- rows buffered before ds.append()
# commit_every=2000 -- rows between ds.commit() calls
```

**Periodic commits:** `ds.commit()` is called every 2000 rows (default) to:
- Free C++ memory buffers
- Enable crash recovery (partial progress is persisted)
- Bound peak memory usage for very large ingestions

A final `ds.commit()` is always called after all rows are written.

**Parallel file normalization (Python only):** When ingesting multiple files, normalization (ffmpeg for video, PyMuPDF for PDFs, file I/O for images/text) runs in a thread pool (up to 4 workers). Since these operations release the GIL, threads provide real parallelism.

**Storage concurrency (Python only):** The SDK sets `deeplake.storage.set_concurrency(32)` during ingestion to parallelize S3/GCS chunk uploads, significantly improving throughput for large datasets.

| Parameter             | Default | Description                        |
| --------------------- | ------- | ---------------------------------- |
| `flush_every`         | 200     | Rows buffered before `ds.append()` |
| `commit_every`        | 2000    | Rows between `ds.commit()` calls   |
| Normalization workers | 4       | Max threads for file processing    |
| Storage concurrency   | 32      | Parallel storage I/O operations    |

---

## Troubleshooting

**"Dataset already exists at path" on CREATE TABLE:**
Pre-pg_deeplake 1.13 this surfaced when a deeplake dataset already lived
on storage but no pg catalog row pointed at it (SDK-created dataset, a
retried CREATE, or a cross-pod race). Fixed in 1.13 — CREATE TABLE now
attaches to the existing deeplog via fall-back-to-open. Upgrade to 1.13+
if you see this on a fresh CREATE TABLE.

**"schema drift on table … pg projects column index N but deeplake dataset has only M columns":**
The on-disk deeplake schema changed under pg (SDK `deeplake.create` /
`add_column` from a side process) and this backend's cached snapshot is
stale.
```sql
SELECT deeplake_flush_table('my_table'::regclass);          -- if DML pending
SELECT deeplake_resync_table_pointer('my_table'::regclass); -- evict snapshot
SELECT * FROM my_table;                                     -- sees new HEAD
```
See [Attach an Existing Dataset](#attach-an-existing-dataset).

**"Expected vector of type UINT8, but found vector of type VARCHAR" on SELECT:**
Reading a typed-column SDK dataset through a BYTEA-domain pg table on a
pg_deeplake older than 1.13. The typed-bytea fill path (1.13+) emits the
SDK-decoded sample bytes through the BYTEA wire. Upgrade.

**"Token required" error:**
```bash
# Set the DEEPLAKE_API_KEY env var, or pass token= to Client()
export DEEPLAKE_API_KEY="dl_xxx"
```

**"Token does not contain org_id" error:**
```python
# Ensure your token contains an OrgID claim, or that the API /me endpoint
# is accessible to fall back on. All tokens should contain OrgID in their JWT payload.
```

**"ffmpeg not found" for video processing:**
```bash
# Install ffmpeg
sudo apt-get install ffmpeg
```

**"fitz not found" for PDF processing (Python):**
```bash
# Install PyMuPDF
pip install pymupdf
```

**Thumbnail generation skipped (Python):**
```bash
# Install Pillow
pip install Pillow
```

**Thumbnail generation skipped (Node.js):**
```bash
# Install sharp
npm install sharp
```

**Connection refused to API:**
```bash
# Check API server is running
curl https://api.deeplake.ai/health
```

**"workspace ID is required" when creating a workspace (HTTP 400):**
```
# POST /workspaces requires the "id" field in the request body.
# The "id" is used in API paths and al:// URLs.
# Example: { "id": "my-workspace", "name": "My Workspace" }
```

**"Not found: /workspaces/.../tables" during ingest:**
```
# The workspace must exist before calling ingest().
# Create it via the API or UI first, then ingest.
```

**Tables created via raw SQL not visible to ingest():**
```
# Tables created with raw SQL (CREATE TABLE ... USING deeplake) are not
# registered with the managed API. The SDK expects tables created via
# POST /workspaces/{id}/tables, which registers the al:// path.
# Use client.ingest() to create tables, or use client.query() for raw SQL
# operations on manually-created tables.
```

**Query timeout on large datasets:**
```python
# Python: increase timeout (default 60s)
results = client.query("SELECT COUNT(*) FROM big_table", timeout=300)
```
```typescript
// Node.js: increase timeout (default 60s)
const rows = await client.query("SELECT COUNT(*) FROM big", undefined, { timeoutMs: 300_000 });
```

**Note:** All database operations (query, list_tables, drop_table, create_table) go through the REST API. No direct PostgreSQL connection is needed from the Python or Node.js client.
