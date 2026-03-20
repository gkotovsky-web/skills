# PyCelonis 2.0 Reference

Full docs: https://celonis.github.io/pycelonis/2.0.1/

## Authentication

```python
from pycelonis import get_celonis

# Environment variables: CELONIS_URL, CELONIS_API_TOKEN
celonis = get_celonis()

# Explicit + options
celonis = get_celonis(
    celonis_url="demo.eu-1.celonis.cloud",
    api_token="your_token",
    timeout=30,
    total_retry=5,
    backoff_factor=1,
)
```

---

## Data Integration

### Data Pools

```python
pools = celonis.data_integration.get_data_pools()
pool = pools.find("Pool Name")          # by name
pool = pools.find("abc123")             # by ID

# Create a pool
new_pool = celonis.data_integration.create_data_pool("New Pool")
```

### Tables: Push (Upload)

```python
import pandas as pd

df = pd.DataFrame({"CASE_ID": [1, 2, 3], "ACTIVITY": ["A", "B", "C"]})

# Replace (drop and recreate)
pool.create_table(df, "MY_TABLE", drop_if_exists=True)

# Append rows
pool.get_table("MY_TABLE").append(df)

# Upsert (insert or update by primary key)
pool.get_table("MY_TABLE").upsert(df)
```

### Tables: Push via Data Push Job (Parquet)

```python
from pycelonis.ems.data_integration.data_pool import JobType

job = pool.create_data_push_job(target_name="MY_TABLE", type_=JobType.REPLACE)
with open("data.parquet", "rb") as f:
    job.add_file_chunk(f)
job.execute()
```

`JobType` options: `REPLACE`, `APPEND`, `UPSERT`, `DELETE`

### Column Configuration

```python
from pycelonis.ems.data_integration.data_pool_table import ColumnTransport, ColumnType

cols = [
    ColumnTransport(column_name="ID", column_type=ColumnType.INTEGER),
    ColumnTransport(column_name="NAME", column_type=ColumnType.STRING),
    ColumnTransport(column_name="CREATED", column_type=ColumnType.DATE),
]
pool.create_table(df, "MY_TABLE", drop_if_exists=True, column_config=cols)
```

`ColumnType` values: `INTEGER`, `LONG`, `FLOAT`, `DOUBLE`, `STRING`, `DATE`, `BOOLEAN`

### Data Models

```python
models = celonis.data_integration.get_data_models()
model = models.find("Model Name")

# Create from pool
model = pool.create_data_model("New Model")

# Reload model
model.reload()
```

---

## PQL: Querying Data

```python
from pycelonis.pql import PQL, PQLColumn, PQLFilter, PQLSort

model = celonis.data_integration.get_data_models().find("My Model")

query = PQL(distinct=True, limit=1000)
query += PQLColumn(name="case",     query='"TABLE"."CASE_ID"')
query += PQLColumn(name="activity", query='"TABLE"."ACTIVITY"')
query += PQLFilter('FILTER "TABLE"."ACTIVITY" = \'A\';')
query += PQLSort('"TABLE"."TIMESTAMP" ASC')

# Returns pandas DataFrame
df = model.get_data_frame(query)

# Chunked for large datasets
df = model.get_data_frame(query, chunksize=10_000)
```

### SaolaPy DataFrame (pandas-like)

```python
import pycelonis.pql as pql

df = pql.DataFrame.from_pql(query, data_model=model)
```

---

## Studio

### Navigate Content

```python
spaces = celonis.studio.get_spaces()
space = spaces.find("Space Name")

packages = space.get_packages()
pkg = packages.find("Package Name")

# Knowledge models
km = pkg.get_knowledge_models().find("KM Name")
content = km.get_content()
data_model_id = content.data_model_id

# Analyses
analysis = pkg.get_analyses().find("Analysis Name")

# Action flows
action_flow = pkg.get_action_flows().find("Flow Name")
```

### Apps Namespace (alternative)

```python
space = celonis.apps.get_spaces().find("Space Name")
pkg = space.get_packages().find("Package Name")
```

---

## Data Jobs

```python
jobs = pool.get_jobs()
for job in jobs:
    print(job.name, job.id)
    transformations = job.get_transformations()

# Execute a job
job.execute()
job.wait_for_execution()  # blocks until complete
```

---

## Team / Users

```python
users = celonis.team.get_users()
user = users.find("user@example.com")

groups = celonis.team.get_groups()
```

---

## Common Patterns

### Find or create table

```python
try:
    table = pool.get_table("MY_TABLE")
except Exception:
    pool.create_table(df, "MY_TABLE")
```

### Iterate all packages across spaces

```python
for space in celonis.studio.get_spaces():
    for pkg in space.get_packages():
        print(space.name, pkg.name)
```

### Export data model to DataFrame

```python
from pycelonis.pql import PQL, PQLColumn

query = PQL()
for col in ["CASE_ID", "ACTIVITY", "TIMESTAMP"]:
    query += PQLColumn(name=col.lower(), query=f'"MY_TABLE"."{col}"')

df = model.get_data_frame(query)
df.to_csv("export.csv", index=False)
```

---

## Migration from 1.x → 2.x

- `get_celonis()` replaces `Celonis()` constructor
- `celonis.data_integration` replaces direct pool access
- `celonis.studio` / `celonis.apps` replaces `celonis.workspaces`
- Collection `.find()` works by name or ID
- Full migration guide: https://celonis.github.io/pycelonis/2.0.1/tutorials/migration/
