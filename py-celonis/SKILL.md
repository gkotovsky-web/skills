---
name: py-celonis
description: Interact with the Celonis EMS platform using the pycelonis 2.0 Python SDK. Use when working with pycelonis, writing Python scripts for Celonis, uploading/downloading data to/from Celonis, querying with PQL, managing data pools/models, or automating Studio content. Triggers: pycelonis, Celonis EMS, PQL, data pool, data model, knowledge model, action flow.
---

# PyCelonis 2.0

Python SDK for the Celonis Execution Management System (EMS). Requires Python 3.8–3.10.

## Quick Start

```python
pip install pycelonis
```

```python
from pycelonis import get_celonis

# Via environment variables: CELONIS_URL, CELONIS_API_TOKEN
celonis = get_celonis()

# Or explicit credentials
celonis = get_celonis(
    celonis_url="demo.eu-1.celonis.cloud",
    api_token="your_api_token",
)
```

## Key Namespaces

| Namespace | Purpose |
|---|---|
| `celonis.data_integration` | Data pools, models, tables, jobs |
| `celonis.studio` | Spaces, packages, analyses, knowledge models |
| `celonis.apps` | App spaces, packages, content nodes |
| `celonis.team` | Users, permissions |

## Workflows

See [REFERENCE.md](REFERENCE.md) for detailed examples.

### Push Data to Celonis

```python
import pandas as pd

pool = celonis.data_integration.get_data_pools().find("My Pool")
df = pd.DataFrame({"CASE_ID": [1, 2], "ACTIVITY": ["A", "B"]})
pool.create_table(df, "MY_TABLE", drop_if_exists=True)
```

### Pull Data with PQL

```python
from pycelonis.pql import PQL, PQLColumn, PQLFilter

model = celonis.data_integration.get_data_models().find("My Model")
query = PQL()
query += PQLColumn(name="case_id", query='"TABLE"."CASE_ID"')
query += PQLFilter('FILTER "TABLE"."ACTIVITY" = \'A\';')
df = model.get_data_frame(query)
```

### Access Studio Content

```python
space = celonis.studio.get_spaces().find("My Space")
pkg = space.get_packages().find("My Package")
km = pkg.get_knowledge_models().find("My KM")
```

## Advanced Features

See [REFERENCE.md](REFERENCE.md) for:
- Data push jobs (Parquet, chunked uploads)
- Upsert / append patterns
- Data model management
- PQL DataFrame (SaolaPy)
- Action flows and automation
