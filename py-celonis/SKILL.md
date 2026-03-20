# PyCelonis Skill (v2.0)

## What this skill does
Guides Claude on how to use PyCelonis — the official Python SDK for the Celonis EMS — to interact programmatically with data pools, data models, Studio packages, and analyses. Used for data ingestion, extraction via PQL, write-back, and automation.

**Docs:** https://celonis.github.io/pycelonis/2.0.1/

---

## Installation

### Inside ML Workbench (pre-installed)
```python
import pycelonis
```

### Outside ML Workbench
```bash
pip install --extra-index-url=https://pypi.celonis.cloud/ pycelonis
```

### Pin to major.minor version (recommended)
```bash
pip install pycelonis=="2.0.*"
```

---

## Authentication & Connection

### Inside ML Workbench (auto-credentials from env vars)
```python
from pycelonis import get_celonis
celonis = get_celonis()
```

### Outside ML Workbench (explicit credentials)
```python
celonis = get_celonis(
    base_url="https://<team>.<realm>.celonis.cloud/",
    api_token="<your-token>",
    key_type="APP_KEY"  # or "USER_KEY"
)
```

### Key types
| Key Type | Description |
|---|---|
| `APP_KEY` | App-level key, custom permissions — preferred for automation |
| `USER_KEY` | User-specific key, inherits user permissions |

### Optional flags
```python
celonis = get_celonis(connect=False)      # skip sanity check
celonis = get_celonis(permissions=False)  # suppress permissions output
```

---

## Object Model

All interactions follow the pattern:
```
celonis.<api_service>.<object_path>.<method>()
```

### API Services
| Service | Accessible Objects |
|---|---|
| `celonis.data_integration` | Data pools, data models, data jobs, tables, tasks |
| `celonis.studio` | Spaces, packages, analyses, action flows, knowledge models, skills, views |
| `celonis.apps` | Same as studio but read-only |

### Standard Methods (available on most objects)
| Method | Description |
|---|---|
| `create_<object>()` | Create new object in EMS |
| `get_<object>(id)` | Retrieve object by ID |
| `get_<object>s()` | Retrieve all objects as CelonisCollection |
| `.update()` | Push local changes to EMS |
| `.sync()` | Pull EMS state into local object |
| `.delete()` | Delete object from EMS (cascades to children) |
| `.dict()` | Print all properties of an object |

### CelonisCollection search
```python
collection = data_pool.get_data_models()
model = collection.find("My Model")                            # by name
model = collection.find("some-id", search_attribute="id")     # by attribute
models = collection.find_all("My Model", search_attribute="name")
```

---

## Data Integration

### Create a Data Pool and Data Model
```python
data_pool = celonis.data_integration.create_data_pool("My Pool")
data_model = data_pool.create_data_model("My Model")
```

### Retrieve existing
```python
data_pool = celonis.data_integration.get_data_pools().find("My Pool")
data_model = data_pool.get_data_models().find("My Model")
```

---

## Data Upload (Push)

Data is pushed as Pandas DataFrames into the Data Pool.

### Create new table
```python
data_pool.create_table(df=my_df, table_name="MY_TABLE", drop_if_exists=True)
```

### Append rows to existing table
```python
table = data_pool.get_tables().find("MY_TABLE")
table.append(my_df)
```

### Upsert (append + replace duplicates by key columns)
```python
table.upsert(my_df, keys=["_CASE_KEY", "ACTIVITY"])
```

> ⚠️ STRING columns default to VARCHAR(80). Use `column_config` for longer strings.

### Add table to Data Model and reload
```python
data_model.add_table(name="MY_TABLE", alias="MY_TABLE")
data_model.reload()

# Partial reload (specific tables only)
tables = data_model.get_tables()
ekko = tables.find("EKKO")
data_model.partial_reload(data_model_table_ids=[ekko.id])
```

---

## Data Export (Pull via PQL)

Data is queried **from the Data Model** (not the pool) using PQL.

### Import PQL objects
```python
from pycelonis.pql import PQL, PQLColumn, PQLFilter, OrderByColumn
```

### Build and execute a query
```python
query = PQL(distinct=False, limit=None, offset=None)

# Add columns (use triple-quotes to avoid escaping)
query += PQLColumn(name="_CASE_KEY",    query=""" "MY_TABLE"."_CASE_KEY" """)
query += PQLColumn(name="ACTIVITY",     query=""" "MY_TABLE"."ACTIVITY" """)
query += PQLColumn(name="EVENTTIME",    query=""" "MY_TABLE"."EVENTTIME" """)

# Add filter
query += PQLFilter(query=""" FILTER "MY_TABLE"."_CASE_KEY" = '1234'; """)

# Add sort order
query += OrderByColumn(query=""" "MY_TABLE"."EVENTTIME" """, ascending=True)

# Pagination
query.limit = 1000
query.offset = 0

# Execute and get DataFrame
df = data_model.export_data_frame(query)
```

### PQL supports advanced functions
- Aggregations: `COUNT`, `AVG`, `SUM`, `STDEV`
- Process functions: `PROCESS EQUALS`, `CALC_REWORK`, `VARIANT`
- Index functions: `INDEX_ACTIVITY_ORDER`, `INDEX_ACTIVITY_LOOP`
- PU-functions: `PU_COUNT`, `PU_FIRST`

Full PQL docs: https://docs.celonis.com/en/pql---process-query-language/pql-function-library.html

---

## Studio Interaction

```python
# List spaces
spaces = celonis.studio.get_spaces()
space = spaces.find("My Space")

# Get a package
package = space.get_packages().find("My Package")

# Get an analysis
analysis = package.get_analyses().find("My Analysis")
```

---

## Logging

```python
import logging
logging.getLogger("pycelonis").setLevel(logging.DEBUG)   # DEBUG | INFO | WARNING
```

---

## Common Patterns for Client Engagements

### Ingest client CSV → Data Pool
```python
import pandas as pd
from pycelonis import get_celonis

celonis = get_celonis()
data_pool = celonis.data_integration.get_data_pools().find("Client Pool")
df = pd.read_csv("client_data.csv")
data_pool.create_table(df=df, table_name="RAW_CLIENT_DATA", drop_if_exists=True)
```

### Pull OCPM data for Python enrichment
```python
from pycelonis.pql import PQL, PQLColumn

query = PQL()
query += PQLColumn(name="CASE_KEY",  query=""" "EKPO"."_CASE_KEY" """)
query += PQLColumn(name="ACTIVITY",  query=""" "ACTIVITIES"."ACTIVITY_EN" """)
query += PQLColumn(name="TIMESTAMP", query=""" "ACTIVITIES"."EVENTTIME" """)

df = data_model.export_data_frame(query)
# ... enrich with Python/ML ...
```

### Write enriched results back to Celonis
```python
data_pool.create_table(df=enriched_df, table_name="ENRICHED_OUTPUT", drop_if_exists=True)
data_model.add_table(name="ENRICHED_OUTPUT", alias="ENRICHED_OUTPUT")
data_model.reload()
```

---

## Notes
- PyCelonis 2.0 has breaking changes from 1.x — see migration guide at docs
- Data export must be enabled by Celonis support on new tenants
- Deleting a parent object (pool, space) cascades to all child objects
- ML Workbench auto-sets `CELONIS_URL`, `CELONIS_API_TOKEN`, `CELONIS_KEY_TYPE` env vars
