# Deeplake Managed Service -- Examples

## Complete Workflow Examples

### Workflow 1: Ingest Videos and Search (Python)

```python
from deeplake import Client

# Initialize -- token from DEEPLAKE_API_KEY env var, workspace defaults to "default"
client = Client()

# Ingest video files (FILE schema)
result = client.ingest("security_videos", {
    "path": ["/path/to/camera1.mp4", "/path/to/camera2.mp4"],
}, schema={"path": "FILE"})
print(f"Ingested {result['row_count']} video segments")

# Fluent query for segments
segments = (
    client.table("security_videos")
        .select("id", "file_id", "start_time", "end_time", "text")
        .where("start_time > $1", 60)
        .limit(10)
)()

for seg in segments:
    print(f"Segment {seg['id']}: {seg['start_time']}s - {seg['end_time']}s")
```

### Workflow 2: Build Semantic Search Index (Python)

```python
from deeplake import Client

client = Client()

# Prepare documents with embeddings
documents = ["Doc about AI", "Doc about ML", "Doc about databases"]
embeddings = [[0.1]*384, [0.2]*384, [0.3]*384]  # Placeholder

# Ingest with indexes for fast search
client.ingest("search_index", {
    "text": documents,
    "embedding": embeddings,
}, index=["embedding", "text"])

# Search (uses deeplake_index automatically).
# Through client.query() the query vector must be INLINED — the REST API
# serializes an array parameter to a string, and passing it as $1 fails with
# "Conversion Error: Type VARCHAR ... can't be cast to FLOAT4[]". (On a direct
# psql / asyncpg / psycopg connection $1::float4[] binds a real array and works.)
from deeplake.managed import vector_literal

query_emb = [0.15]*384  # Placeholder

results = client.query(f"""
    SELECT text, embedding::float4[] <#> {vector_literal(query_emb)} AS similarity
    FROM search_index
    ORDER BY similarity DESC
    LIMIT 5
""")

for r in results:
    print(f"{r['similarity']:.3f}: {r['text']}")
```

### Workflow 3: Process PDF Documents (Python)

```python
from deeplake import Client

client = Client()

# Ingest PDFs (each page becomes a row)
result = client.ingest("manuals", {
    "path": ["/path/to/manual1.pdf", "/path/to/manual2.pdf"],
}, schema={"path": "FILE"})
print(f"Processed {result['row_count']} pages")

# Search within PDFs
pages = client.query("""
    SELECT file_id, page_index, text
    FROM manuals
    WHERE text @> 'installation'
""")

for page in pages:
    print(f"Found in file {page['file_id']}, page {page['page_index']}")
```

### Workflow 4: Iterate Over Large Datasets (Python)

```python
from deeplake import Client

client = Client()

# Use open_table() for large-scale iteration (bypasses PostgreSQL)
ds = client.open_table("large_table", read_only=True)
for batch in ds.batches(1000):
    process(batch)
```

### Workflow 5: Ingest COCO Panoptic and Query Segments (Python)

```python
from deeplake import Client
from deeplake.managed.formats import CocoPanoptic

client = Client()

# Ingest panoptic dataset using format object
# Thumbnails auto-generated for IMAGE columns
result = client.ingest("panoptic_train", format=CocoPanoptic(
    images_dir="/data/coco/train2017",
    masks_dir="/data/coco/panoptic_train2017",
    annotations="/data/coco/annotations/panoptic_train2017.json",
))
print(f"Ingested {result['row_count']} images")

# Query for images with specific categories
rows = client.query("""
    SELECT coco_image_id, filename, segments_info
    FROM panoptic_train
    LIMIT 10
""")

import json
for row in rows:
    segments = json.loads(row["segments_info"])
    print(f"Image {row['filename']}: {len(segments)} segments")
```

### Workflow 6: Ingest COCO Detection and Query (Python)

```python
from deeplake import Client
from deeplake.managed.formats import Coco

client = Client()

# Ingest detection dataset — auto-detects annotation files from split name
result = client.ingest("coco_val", format=Coco(
    images_dir="/data/coco/val2017",
    max_images=100,  # limit for testing
))
print(f"Ingested {result['row_count']} images")

# Query for images with many objects
rows = client.query("""
    SELECT coco_image_id, filename, num_objects, captions
    FROM coco_val
    WHERE num_objects > 10
    ORDER BY num_objects DESC
    LIMIT 5
""")

import json
for row in rows:
    caps = json.loads(row["captions"])
    print(f"Image {row['filename']}: {row['num_objects']} objects, captions: {caps}")
```

### Workflow 7: Ingest LeRobot Robotics Dataset and Connect to Training (Python)

```python
from deeplake import Client
from deeplake.managed.formats import LeRobotTasks, LeRobotFrames, LeRobotEpisodes

client = Client()

DATASET_DIR = "/path/to/lerobot_dataset"  # HuggingFace LeRobot v2.0 format

# LeRobot uses a 3-table design: tasks, frames, episodes.
# Ingest them in order (tasks first — it's the lookup table).

# 1. Tasks — fast, reads meta/tasks.jsonl (~31K rows for DROID)
client.ingest("droid_tasks", format=LeRobotTasks(DATASET_DIR))

# 2. Frames — per-frame state/action scalars from parquet files
#    chunk_start/chunk_end control which chunks to process (inclusive)
#    Each chunk = 1000 episodes. DROID has 93 chunks total.
client.ingest("droid_frames", format=LeRobotFrames(
    DATASET_DIR, chunk_start=0, chunk_end=3,  # first 4 chunks
))

# 3. Episodes — video data + metadata, pulls LFS videos lazily per chunk
#    Each chunk's videos are pulled via git-lfs, processed, then freed.
#    ~40s per chunk (LFS pull + read 3 camera MP4s per episode).
client.ingest("droid_episodes", format=LeRobotEpisodes(
    DATASET_DIR, chunk_start=0, chunk_end=3,
))

# Query frame data
frames = client.query("""
    SELECT episode_index, frame_index, state_x, state_y, action_x, action_y
    FROM droid_frames
    WHERE episode_index = 0
    ORDER BY frame_index
    LIMIT 10
""")
for f in frames:
    print(f"Frame {f['frame_index']}: state=({f['state_x']:.3f}, {f['state_y']:.3f})")

# Connect to training — open_table() bypasses SQL, returns native dataset
ds = client.open_table("droid_frames", read_only=True)
print(f"Dataset: {len(ds)} frames")

# RECOMMENDED: Use ds.query() + ds.batches() for fast training on large datasets.
# ds.pytorch() with DataLoader works but is slower on remote data due to per-row access.
import torch
import torch.nn as nn
import numpy as np

STATE_COLS = ["state_x","state_y","state_z","state_roll","state_pitch","state_yaw","state_pad","state_gripper"]
ACTION_COLS = ["action_x","action_y","action_z","action_roll","action_pitch","action_yaw","action_gripper"]

# Get a training subset (query runs server-side, fast)
train_view = ds.query("SELECT state_x, state_y, state_z, state_roll, state_pitch, state_yaw, state_pad, state_gripper, action_x, action_y, action_z, action_roll, action_pitch, action_yaw, action_gripper WHERE episode_index < 100")

# Simple behavior cloning model
model = nn.Sequential(nn.Linear(8, 64), nn.ReLU(), nn.Linear(64, 64), nn.ReLU(), nn.Linear(64, 7))
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

for epoch in range(3):
    total_loss = 0
    n = 0
    for batch in train_view.batches(256):  # dict of numpy arrays, very fast
        states = torch.tensor(np.stack([batch[c] for c in STATE_COLS], axis=1), dtype=torch.float32)
        actions = torch.tensor(np.stack([batch[c] for c in ACTION_COLS], axis=1), dtype=torch.float32)
        loss = nn.MSELoss()(model(states), actions)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        total_loss += loss.item()
        n += 1
    print(f"Epoch {epoch}: loss={total_loss/n:.6f}")
```

**Important notes for LeRobot ingestion:**
- `chunk_end` is **inclusive** — `chunk_start=0, chunk_end=3` processes chunks 0, 1, 2, 3
- Videos require `git lfs` installed and the dataset cloned (not just downloaded)
- Episodes ingestion pulls ~40s per chunk for LFS, then restores pointers to save disk
- For large datasets, ingest in stages: do chunks 0-9 first, verify, then continue
- Tables support **append**: call `ingest()` again with the same table name and different chunk range to add more data
- Dependencies: `pandas`, `numpy` (for frames); `git lfs` (for episodes)

**Training performance notes:**
- `ds.batches(N)` returns dicts of numpy arrays — ~10x faster than `DataLoader(ds.pytorch())`
- Use `ds.query("SELECT ... WHERE ...")` to filter columns/rows server-side before training
- 3 epochs over 19K frames completes in ~10s with `ds.batches(256)`
- For PyTorch DataLoader compatibility (e.g., shuffling), use on smaller subsets only

### Workflow 8: Node.js -- Ingest and Query (TypeScript)

```typescript
import { ManagedClient, CocoPanoptic, initializeWasm } from 'deeplake';

await initializeWasm();

const client = new ManagedClient({
    token: process.env.DEEPLAKE_API_KEY!,
    workspaceId: 'default',
});

// Ingest COCO panoptic
const result = await client.ingest("panoptic", null, {
    format: new CocoPanoptic({
        imagesDir: "coco/train2017",
        masksDir: "coco/panoptic_train2017",
        annotations: "coco/annotations/panoptic_train2017.json",
    }),
});
console.log(`Ingested ${result.rowCount} images`);

// Fluent query
const rows = await client.table("panoptic")
    .select("coco_image_id", "filename", "segments_info")
    .limit(10)
    .execute();

for (const row of rows) {
    const segments = JSON.parse(row.segments_info as string);
    console.log(`Image ${row.filename}: ${segments.length} segments`);
}

// Table management
const tables = await client.listTables();
console.log("Tables:", tables);
```

### Workflow 9: Node.js -- Ingest Structured Data (TypeScript)

```typescript
import { ManagedClient, initializeWasm } from 'deeplake';

await initializeWasm();

const client = new ManagedClient({
    token: process.env.DEEPLAKE_API_KEY!,
});

// Ingest structured data
await client.ingest("embeddings", {
    text: ["Hello world", "Goodbye world"],
    embedding: [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6]],
    score: [0.9, 0.8],
});

// Vector similarity search — through client.query() the vector must be
// inlined; the REST API serializes an array parameter to a string.
import { vectorLiteral } from 'deeplake';

const queryEmb = [0.15, 0.25, 0.35];
const results = await client.query(
    `SELECT text, embedding::float4[] <#> ${vectorLiteral(queryEmb)} AS similarity
     FROM embeddings
     ORDER BY similarity DESC
     LIMIT 5`,
);

for (const r of results) {
    console.log(`${r.similarity}: ${r.text}`);
}
```

---

### Workflow 10: Clone a Dataset's Schema and Indexes (Python)

Build an empty "skeleton" table whose columns **and index types** match an
existing dataset — the shape you want before a migration, a backfill, or an
`INSERT INTO … SELECT`.

Two things make a naive attempt silently wrong:

1. **`EMBEDDING` without a dimension** becomes a variable-length `float4[]`,
   not an `embedding(1536, …)` column. An empty table has no rows to infer the
   dimension from, so this is unrecoverable later.
2. **`create_index` without `index_type`** uses pg_deeplake's default, which for
   text is `exact_text` — while real datasets usually carry `inverted_index` or
   `bm25`. Nothing errors; the clone just behaves differently under search.

```python
from deeplake import Client

client = Client(workspace_id="my-workspace")

SOURCE = "embedding_noquantized"
TARGET = "embedding_skeleton"

# 1. Read the source's real column types AND built index types.
#    describe_table opens read-only — inspecting must never take write access.
src = client.describe_table(SOURCE)
for col, info in src.items():
    print(f"{col:12s} {info['type']:32s} index={info['index']}")
# chunk_id     text (compression=lz4)           index=inverted_index
# created_at   int64                            index=None
# embedding    embedding(1536, clustered)       index=clustered
# id           text (compression=lz4)           index=inverted_index


# 2. Translate deeplake type strings -> SDK schema types.
def to_schema_type(deeplake_type: str) -> str:
    if deeplake_type.startswith("embedding("):
        # "embedding(1536, clustered)" -> "EMBEDDING(1536)".  Keeping the
        # dimension is the whole point: bare EMBEDDING loses it.
        dim = deeplake_type[len("embedding("):].split(",")[0].rstrip(")")
        return f"EMBEDDING({dim.strip()})"
    if deeplake_type.startswith("text"):
        return "TEXT"
    if deeplake_type in ("int64", "int32", "float32", "float64", "bool"):
        return deeplake_type.upper()
    if deeplake_type.startswith("image"):
        return "IMAGE"
    raise ValueError(f"add a mapping for {deeplake_type!r}")


schema = {col: to_schema_type(info["type"]) for col, info in src.items()}
# Carry each column's index type across verbatim — describe_table reports the
# same names create_index accepts, so no translation table is needed.
index = {col: info["index"] for col, info in src.items() if info["index"]}

client.create_table(TARGET, schema, index=index)


# 3. Verify the clone matches. Index types are compared, not just column names.
dst = client.describe_table(TARGET)
mismatches = {
    col: (src[col]["index"], dst.get(col, {}).get("index"))
    for col in src
    if src[col]["index"] != dst.get(col, {}).get("index")
}
assert not mismatches, f"index type mismatch: {mismatches}"
print("skeleton matches the source")
```

> **What a clone can and cannot reproduce.** Column types, dimensions and index
> types all carry across. **Chunk compression does not.** pg_deeplake hardwires
> a PG `TEXT` column to `text(compression=null)`, so if the source was created
> directly with the SDK (`deeplake.create(..., Text(chunk_compression="lz4"))`)
> its text columns read back as `compression=lz4` while the clone reads
> `compression=null`. This affects storage size, not semantics or query
> behaviour — compare the `index` field for correctness, and treat a `type`
> difference that is only `compression=` as expected. If you need byte-identical
> storage, create the dataset with the SDK and attach a pg table to it instead
> (see [reference.md](reference.md#attach-an-existing-dataset)).

**Copying the data in.** The skeleton is a normal managed table, so `ingest()`
appends to it — the pre-declared schema wins over type inference:

```python
client.ingest(TARGET, {
    "id": ids, "chunk_id": chunk_ids,
    "created_at": timestamps, "embedding": vectors,
})

# Reads lag writes by a few seconds — poll rather than assuming.
import time
expected = len(ids)
for _ in range(15):
    n = client.query(f'SELECT COUNT(*) AS n FROM "{TARGET}"')[0]["n"]
    if n >= expected:
        break
    time.sleep(2)
```

**If a column already has the wrong index**, `create_index` reports it rather
than silently no-op'ing (`CREATE INDEX IF NOT EXISTS` would skip it):

```python
client.drop_index(TARGET, "id")
client.create_index(TARGET, "id", index_type="inverted_index")
```

**Node.js equivalent:**

```typescript
const src = await client.describeTable(SOURCE);

const schema: Record<string, string> = {};
const index: Record<string, string> = {};
for (const [col, info] of Object.entries(src)) {
    if (info.type.startsWith('embedding(')) {
        const dim = info.type.slice('embedding('.length).split(',')[0].replace(')', '');
        schema[col] = `EMBEDDING(${dim.trim()})`;
    } else if (info.type.startsWith('text')) {
        schema[col] = 'TEXT';
    } else {
        schema[col] = info.type.toUpperCase();
    }
    if (info.index) index[col] = info.index;
}

await client.createTable(TARGET, schema, { index });
```

---

### Workflow 11: Read-Only Access for Training and Inspection (Python)

`open_table()` defaults to **read/write**, which takes a writer handle on the
dataset. Anything that only reads should say so — it prevents a stray
`append`/`commit` from mutating a table you meant to leave alone.

```python
# Inspect a table without write access
ds = client.open_table("training_data", read_only=True)
ds.summary()
print(len(ds))

# Training reads only — open read-only
ds = client.open_table("training_data", read_only=True)
for batch in ds.batches(256):
    train_step(batch)

from torch.utils.data import DataLoader
loader = DataLoader(ds.pytorch(), batch_size=32, shuffle=True, num_workers=4)

# Mutating a read-only handle raises instead of silently succeeding
ds.append({"id": ["x"]})        # raises

# Writing needs the default handle
ds_rw = client.open_table("training_data")
ds_rw.append({"id": ["x"]})
ds_rw.commit()
```

```typescript
// Node.js — second argument is readOnly
const ds = await client.openTable("training_data", true);
```

---

## Detailed Ingestion Examples

### Ingest Video Files (FILE schema)

```python
client.ingest("security_footage", {
    "path": ["camera1_2025-01-15.mp4", "camera2_2025-01-15.mp4"],
}, schema={"path": "FILE"})
# Creates ~10-second segments with thumbnails
# Each segment has: id, file_id, chunk_index, start_time, end_time, video_data, thumbnail, text
```

### Ingest Text Documents

```python
client.ingest("documents", {
    "path": ["report.txt", "notes.md", "data.json"],
}, schema={"path": "FILE"})
# Text is chunked into ~1000 char pieces with 200 char overlap
# Each chunk has: id, file_id, chunk_index, text
```

### Ingest Images

```python
client.ingest("photos", {
    "path": ["image1.jpg", "image2.png"],
}, schema={"path": "FILE"})
# Each image stored as single row
# Columns: id, file_id, image (binary), filename, text
```

### Ingest PDFs

```python
client.ingest("manuals", {"path": ["manual.pdf"]}, schema={"path": "FILE"})
# Each page rendered at 300 DPI as PNG
# Columns: id, file_id, page_index, image (binary), text (extracted)
```

### Ingest with Progress Callback

```python
def progress(rows_written, total):
    print(f"Written {rows_written} rows...")

client.ingest("documents", {"path": pdf_files}, schema={"path": "FILE"}, on_progress=progress)
```

### Ingest Structured Data (dict = column data)

```python
client.ingest("vectors", {
    "id": ["doc1", "doc2", "doc3"],
    "text": ["Hello world", "Goodbye world", "Another doc"],
    "embedding": [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6], [0.7, 0.8, 0.9]],
    "score": [0.9, 0.8, 0.7],
})
# Schema inferred from data types
# Embeddings auto-detected as float arrays
```

### Ingest with Explicit Schema

```python
client.ingest("data", {"name": ["Alice", "Bob"], "age": [30, 25]},
              schema={"name": "TEXT", "age": "INT64"})
```

### Ingest from HuggingFace

```python
client.ingest("mnist", {"_huggingface": "mnist"})
client.ingest("cifar", {"_huggingface": "cifar10"})
client.ingest("squad", {"_huggingface": "squad"})
```

### Ingest COCO Panoptic Data (format object)

```python
from deeplake.managed.formats import CocoPanoptic

client.ingest("panoptic", format=CocoPanoptic(
    images_dir="coco/train2017",
    masks_dir="coco/panoptic_train2017",
    annotations="coco/annotations/panoptic_train2017.json",
))
# Each image becomes one row with columns:
# coco_image_id (int), image (IMAGE), mask (SEGMENT_MASK), width (int), height (int),
# filename (str), mask_filename (str), segments_info (JSON str), categories (JSON str)
# Thumbnails auto-generated for IMAGE columns at 32x32, 64x64, 128x128, 256x256
```

### Ingest COCO Detection/Captions Data (format object)

```python
from deeplake.managed.formats import Coco

client.ingest("coco_train", format=Coco(
    images_dir="coco/train2017",
    annotations_dir="coco/annotations",
))
# Each image becomes one row with columns:
# coco_image_id (int), image (IMAGE), mask (SEGMENT_MASK), width (int), height (int),
# filename (str), captions (JSON str), annotations (JSON str),
# categories (JSON str), num_objects (int)
# Requires: pycocotools, Pillow, numpy
```
