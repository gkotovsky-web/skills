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

## Creating Studio Views and Components

Views are created by calling the Celonis blueprint API directly via `pkg.client.request()`. **Do NOT use `pkg.create_view()`** — it has a bug where it serializes config as YAML with a malformed key (`allowAdvancedFilters:`) causing 500 errors.

### Setup

```python
import json, uuid
from pycelonis import get_celonis

celonis = get_celonis(base_url="https://team.celonis.cloud", api_token="...")
space = celonis.studio.get_spaces().find("My Space")
pkg = space.get_packages().find("My Package")
client = pkg.client
km_key = "my-knowledge-model-key"  # from pkg.get_knowledge_models()
```

### Get column types from data model

```python
TYPE_MAP = {"STRING": "string", "DATE": "date", "INT": "int", "FLOAT": "float", "BOOLEAN": "boolean", "DATETIME": "date"}

pool = celonis.data_integration.get_data_pool("<pool-id>")
model = pool.get_data_model("<model-id>")

cols = []
for t in model.get_tables():
    if t.name == "t_o_custom_MyObject":
        for c in t.get_columns():
            cols.append({"name": c.name, "type": TYPE_MAP.get(c.type_, "string")})
        break
```

---

### Building a `table` component

One dataSource with all columns as attributes. Each column in `data.columns` references an attribute by `field` ID.

```python
alias = "o_custom_MyObject"                  # table alias (no t_ prefix)
object_prefix = "O_CUSTOM_MYOBJECT"          # uppercase, used in referencedEntity

ds_id = str(uuid.uuid4())
attributes, columns = [], []

for i, col in enumerate(cols):
    attr_id = str(uuid.uuid4())
    attributes.append({
        "id": attr_id,
        "pql": f'"{alias}"."{col["name"]}"\n',
        "columnType": col["type"],
        "aggregation": False,
        "description": "",
        "referencedEntity": {
            "id": f"{object_prefix}.{col['name'].upper()}",
            "type": "NATIVE_ATTRIBUTE",
            "columnType": col["type"]
        },
        "shortDisplayName": "",
        "hasKnowledgeSyncDisplayChanges": True
    })
    columns.append({"id": str(uuid.uuid4()), "field": attr_id, "order": (i + 1) * 100})

table_component = {
    "id": f"table-{str(uuid.uuid4())}",
    "type": "table",
    "scope": None,
    "knowledgeModelKey": None,
    "excludedTools": None,
    "settings": {
        "data": {"columns": columns},
        "dataSources": [{
            "id": ds_id,
            "attributes": attributes,
            "description": "",
            "displayName": ds_id,
            "shortDisplayName": ""
        }]
    },
    "tools": None,
    "filters": None,
    "description": None,
    "semanticLayerId": None
}
```

---

### Building a `kpi-list` component

One dataSource **per KPI** (unlike table). The `kpi` field references `"{dataSourceId}.{attributeId}"`.

```python
# kpis: list of dicts with keys: displayName, pql, columnType, format
kpis = [
    {"displayName": "Total Net Order Value", "pql": 'SUM("o_custom_PurchaseOrderItem"."NetOrderValue")', "columnType": "float", "format": ",.3~s"},
    {"displayName": "Total Order Quantity",   "pql": 'SUM("o_custom_PurchaseOrderItem"."OrderQuantity")', "columnType": "float", "format": ",.3~s"},
]

data_sources, kpi_refs = [], []

for i, kpi in enumerate(kpis):
    ds_id   = str(uuid.uuid4())
    attr_id = str(uuid.uuid4())
    data_sources.append({
        "id": ds_id,
        "attributes": [{
            "id": attr_id,
            "pql": kpi["pql"],
            "columnType": kpi["columnType"],
            "format": kpi["format"],
            "aggregation": True,
            "description": "",
            "displayName": kpi["displayName"],
            "shortDisplayName": ""
        }],
        "description": "",
        "displayName": ds_id,
        "shortDisplayName": ""
    })
    kpi_refs.append({"id": str(uuid.uuid4()), "kpi": f"{ds_id}.{attr_id}", "show": True, "order": (i + 1) * 100})

kpi_list_component = {
    "id": f"kpi-list-{str(uuid.uuid4())}",
    "type": "kpi-list",
    "filters": None,
    "settings": {
        "data": {"kpis": kpi_refs},
        "options": {"itemWidthMode": "evenly"},
        "dataSources": data_sources
    },
    "tools": None,
    "description": None,
    "semanticLayerId": None
}
```

To pull KPI definitions from the knowledge model:

```python
km = pkg.get_knowledge_models().find("My KM")
km_content = km.serialized_content
if isinstance(km_content, str):
    km_content = json.loads(km_content)

kpis = [
    {"displayName": k["displayName"], "pql": k["pql"], "columnType": "float", "format": k.get("format", "")}
    for k in km_content.get("kpis", [])
]
```

---

### Creating the view

```python
def create_view(client, pkg, name, key, km_key, components):
    """
    components: list of dicts with:
        - "component": component object (table, kpi-list, etc.)
        - "position": {"positionX", "positionY", "width", "height"}

    Grid notes:
    - Uses a 27-column system (not 12)
    - Use mode "full-height" to stretch vertically to fill the screen
    - positionY is 1-based; stack components by incrementing Y
    """
    config = {
        "base": None,
        "metadata": {
            "key": key,
            "name": name,
            "template": False,
            "showCaseCount": True,
            "knowledgeModelKey": km_key,
            "allowAICapabilities": True,
            "allowInsightsCapabilities": None,
            "allowAdvancedFilters": True,
            "allowScheduledReports": True,
            "advancedFiltersSettings": {
                "processFilter": {"active": True},
                "attributeFilter": {"active": True},
                "eventLogs": []
            },
            "objectCountSettings": None
        },
        "variables": None,
        "layout": None,
        "grid": {
            "elements": [
                {
                    "id": str(uuid.uuid4()),
                    "componentId": c["component"]["id"],
                    "position": c["position"]
                }
                for c in components
            ],
            "mode": "full-height",
            "height": None,
            "width": None
        },
        "components": [c["component"] for c in components],
        "filters": None,
        "selections": None,
        "asides": None
    }

    payload = {
        "boardAssetType": "BOARD_V2",
        "configuration": json.dumps(config),
        "id": None,
        "inputVariableDefinitions": None,
        "parentNodeId": pkg.id,
        "parentNodeKey": pkg.key,
        "rootNodeKey": pkg.key
    }

    resp = client.request("POST", "/blueprint/api/boards", json=payload, type_=None)
    return json.loads(resp.text)
```

#### Single full-width table

```python
create_view(client, pkg, "My View", "my-view-key", km_key, [
    {
        "component": table_component,
        "position": {"positionX": 1, "positionY": 1, "width": 27, "height": 36}
    }
])
```

#### KPI list on top + table below

```python
create_view(client, pkg, "My View", "my-view-key", km_key, [
    {
        "component": kpi_list_component,
        "position": {"positionX": 1, "positionY": 1, "width": 27, "height": 7}
    },
    {
        "component": table_component,
        "position": {"positionX": 1, "positionY": 8, "width": 27, "height": 36}
    }
])
```

#### Update an existing view (PUT)

```python
view_id = "<view-id>"
resp = client.request("GET", f"/blueprint/api/boards/{view_id}", type_=None)
config = json.loads(resp.text)["configuration"]

# Modify config (add component, update grid, etc.)
config["components"].append(new_component)
config["grid"]["elements"].append({
    "id": str(uuid.uuid4()),
    "componentId": new_component["id"],
    "position": {"positionX": 1, "positionY": 1, "width": 27, "height": 36}
})

payload = {
    "boardAssetType": "BOARD_V2",
    "configuration": json.dumps(config),
    "id": view_id,
    "inputVariableDefinitions": None,
    "parentNodeId": pkg.id,
    "parentNodeKey": pkg.key,
    "rootNodeKey": pkg.key
}
client.request("PUT", f"/blueprint/api/boards/{view_id}", json=payload, type_=None)
```

---

## Migration from 1.x → 2.x

- `get_celonis()` replaces `Celonis()` constructor
- `celonis.data_integration` replaces direct pool access
- `celonis.studio` / `celonis.apps` replaces `celonis.workspaces`
- Collection `.find()` works by name or ID
- Full migration guide: https://celonis.github.io/pycelonis/2.0.1/tutorials/migration/
