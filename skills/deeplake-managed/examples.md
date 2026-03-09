# DeepLake Managed Service -- Examples

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

# Search (uses deeplake_index automatically)
query_emb = [0.15]*384  # Placeholder

results = client.query("""
    SELECT text, embedding <#> $1 AS similarity
    FROM search_index
    ORDER BY similarity DESC
    LIMIT 5
""", (query_emb,))

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
ds = client.open_table("large_table")
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

### Workflow 7: Node.js -- Ingest and Query (TypeScript)

```typescript
import { ManagedClient, CocoPanoptic } from '@deeplake/node';

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

### Workflow 8: Node.js -- Ingest Structured Data (TypeScript)

```typescript
import { ManagedClient } from '@deeplake/node';

const client = new ManagedClient({
    token: process.env.DEEPLAKE_API_KEY!,
});

// Ingest structured data
await client.ingest("embeddings", {
    text: ["Hello world", "Goodbye world"],
    embedding: [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6]],
    score: [0.9, 0.8],
});

// Vector similarity search
const queryEmb = [0.15, 0.25, 0.35];
const results = await client.query(
    `SELECT text, embedding <#> $1 AS similarity
     FROM embeddings
     ORDER BY similarity DESC
     LIMIT 5`,
    [queryEmb],
);

for (const r of results) {
    console.log(`${r.similarity}: ${r.text}`);
}
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
