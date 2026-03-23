---
name: py-celonis
description: Interact with the Celonis EMS platform using the pycelonis 2.0 Python SDK. Always use this skill when the user is writing any Python code that touches Celonis, mentions pycelonis, wants to upload or download data to/from Celonis, query process data with PQL, manage data pools or data models, or automate anything in Celonis Studio — even if they don't say "pycelonis" explicitly. If the user is doing anything programmatic with Celonis in Python, this skill applies.
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

# Or explicit credentials — use base_url (NOT celonis_url)
celonis = get_celonis(
    base_url="https://team.celonis.cloud",
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

`celonis.data_integration` does NOT have `get_data_models()`. Data models live under a pool:

```python
from pycelonis.pql import PQL, PQLColumn, PQLFilter
import pycelonis.pql as pql

# Get pool first, then model
pool = celonis.data_integration.get_data_pool("<pool-id>")
model = pool.get_data_model("<model-id>")

# List available tables and their aliases
for t in model.get_tables():
    print(t.name, "→ alias:", t.alias)

query = PQL()
query += PQLColumn(name="case_id", query='"<table-alias>"."<column>"')
query += PQLFilter('FILTER "<table-alias>"."<column>" = \'value\';')

# Use SaolaPy (current recommended API)
df = pql.DataFrame.from_pql(query, data_model=model)
print(df.head(5))
```

**Important: use the table alias in PQL, not the table name.**
If content-cli shows a table as `t_o_custom_PurchaseOrder`, the PQL alias is `o_custom_PurchaseOrder` (no `t_` prefix). Always call `model.get_tables()` to confirm the correct alias and actual column names before writing PQL queries.

`model.export_data_frame(query)` still works but is deprecated — prefer `pql.DataFrame.from_pql(query, data_model=model)`.

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
