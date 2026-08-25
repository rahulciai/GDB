# YAML → Docling Graph → PostgreSQL + Apache AGE

This project converts YAML into a graph using **Docling Graph + Gemini**, then stores the graph in **PostgreSQL using Apache AGE**.

## 1. Sample YAML

File:

```text
sample.yaml
```

Contains entities such as:

```text
Company
Department
Person
Project
```

and relationships such as:

```text
HAS_DEPARTMENT
MANAGED_BY
HAS_EMPLOYEE
HAS_PROJECT
OWNED_BY
HAS_MEMBER
```

## 2. Install Python dependencies

```bash
pip install docling-graph python-dotenv psycopg[binary]
```

## 3. Configure Gemini

Create `.env`:

```env
GEMINI_API_KEY=your_key_here

PG_HOST=localhost
PG_PORT=5432
PG_DATABASE=graphdb
PG_USER=your_user
PG_PASSWORD=your_password
```

Add `.env` to `.gitignore`.

## 4. Read YAML

```python
from pathlib import Path

yaml_text = Path("sample.yaml").read_text(
    encoding="utf-8"
)
```

## 5. Automatically detect graph schema

```python
from docling_graph.templategen import (
    DocumentContent,
    build_llm_call_fn,
    induce_spec_from_documents,
)

llm_call = build_llm_call_fn(
    "gemini",
    "gemini-2.5-flash",
    structured_output=False,
)

spec, report = induce_spec_from_documents(
    [
        DocumentContent(
            name="sample_yaml",
            text=yaml_text
        )
    ],
    llm_call,
    strategy="one-shot",
)

print(spec.model_dump_json(indent=2))
```

Docling Graph automatically detected the entities, properties and relationships.

## 6. Generate Pydantic template automatically

```python
from pathlib import Path

from docling_graph.templategen import (
    render_template,
    verify_template_source,
)

template_source = render_template(spec)

verification = verify_template_source(
    template_source,
    root_class=spec.root,
    spec=spec,
)

Path("generated_yaml_template.py").write_text(
    template_source,
    encoding="utf-8"
)

print("Template valid:", verification.passed)
```

Generated file:

```text
generated_yaml_template.py
```

No Pydantic classes were written manually.

## 7. Generate actual graph

```python
from docling_graph import PipelineConfig, run_pipeline
from generated_yaml_template import Company

config = PipelineConfig(
    source=yaml_text,
    template=Company,
    backend="llm",
    inference="remote",
    provider_override="gemini",
    model_override="gemini-2.5-flash",
    processing_mode="many-to-one",
)

context = run_pipeline(
    config,
    mode="api"
)

graph = context.knowledge_graph
```

Inspect:

```python
for node_id, data in graph.nodes(data=True):
    print(node_id, data)

for source, target, data in graph.edges(data=True):
    print(source, target, data)
```

## 8. PostgreSQL Docker

Existing container:

```text
github-pr-builder-postgres
```

Image:

```text
postgres:17
```

PostgreSQL version:

```text
17.11
```

OS:

```text
Debian 13 (trixie)
```

## 9. Create separate database

Created:

```text
graphdb
```

This keeps graph work separate from the existing application database.

## 10. Install Apache AGE

Installed build dependencies inside the PostgreSQL container:

```bash
apt-get install -y \
git \
build-essential \
bison \
flex \
libreadline-dev \
zlib1g-dev \
postgresql-server-dev-17 \
ca-certificates
```

Cloned Apache AGE:

```bash
git clone https://github.com/apache/age.git
```

Checked out PostgreSQL 17 compatible release:

```bash
git checkout 'PG17/v1.7.0-rc0'
```

Compiled and installed:

```bash
make PG_CONFIG=/usr/lib/postgresql/17/bin/pg_config install
```

## 11. Enable AGE in `graphdb`

```sql
CREATE EXTENSION age;

LOAD 'age';

SET search_path = ag_catalog, "$user", public;
```

## 12. Create AGE graph

```sql
SELECT create_graph('yaml_graph');
```

Graph name:

```text
yaml_graph
```

## 13. Verify AGE with a test node

```sql
SELECT *
FROM cypher('yaml_graph', $$
    CREATE (n:TestNode {
        name: 'AGE test'
    })
    RETURN n
$$) AS (n agtype);
```

Verify:

```sql
SELECT *
FROM cypher('yaml_graph', $$
    MATCH (n:TestNode)
    RETURN n
$$) AS (n agtype);
```

## 14. Connect Python to AGE

```python
import os
import psycopg
from dotenv import load_dotenv

load_dotenv()

conn = psycopg.connect(
    host=os.getenv("PG_HOST"),
    port=os.getenv("PG_PORT"),
    dbname=os.getenv("PG_DATABASE"),
    user=os.getenv("PG_USER"),
    password=os.getenv("PG_PASSWORD"),
)

with conn.cursor() as cursor:
    cursor.execute("LOAD 'age';")
    cursor.execute(
        'SET search_path = ag_catalog, "$user", public;'
    )
```

To reopen the connection after:

```python
conn.close()
```

just rerun the connection cell.

## 15. Current status

Current flow:

```text
sample.yaml
    ↓
Docling Graph
    ↓
Gemini
    ↓
Automatically generated schema
    ↓
generated_yaml_template.py
    ↓
Actual nodes + edges
    ↓
Python
    ↓
PostgreSQL
    ↓
Apache AGE
    ↓
yaml_graph
```

The AGE database is working and graph nodes are ready to be inserted.

### Next

Insert Docling-generated **nodes first**, then insert the corresponding **edges** into `yaml_graph`.