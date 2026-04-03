# dao-ai-apps-example

A minimal example showing how to deploy a [dao-ai](https://github.com/natefleming/dao-ai) agent directly as a [Databricks App](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/) using Databricks Asset Bundles.

The entire project is just three config files and a `pyproject.toml` -- no application code required. The dao-ai framework handles the agent server, routing, and MLflow integration out of the box.

## Project Structure

```
dao-ai-apps-example/
├── databricks.yaml      # Databricks Asset Bundle configuration
├── dao_ai.yaml           # dao-ai agent configuration (model, tools, prompts)
├── app.yaml              # Databricks App manifest (command, env, resources)
├── pyproject.toml        # Python dependencies (dao-ai)
├── uv.lock               # Lockfile for reproducible installs
└── .gitignore
```

| File | Purpose |
|------|---------|
| `databricks.yaml` | Defines the bundle, app resource, compute config, environment variables, and serving endpoint permissions. Uses `${workspace.file_path}` for portable source code paths. |
| `dao_ai.yaml` | Declares the LLM, agents, tools, and app metadata. This is the primary config you edit to change agent behavior. |
| `app.yaml` | Standalone app manifest used by the Databricks Apps runtime. Mirrors the `config` section of `databricks.yaml`. |
| `pyproject.toml` | Declares `dao-ai` as the sole dependency. The Databricks Apps runtime uses `uv` to install from this file automatically. |
| `uv.lock` | Generated lockfile ensuring reproducible dependency installs across deployments. |

## Prerequisites

- [Databricks CLI](https://docs.databricks.com/dev-tools/cli/index.html) (v0.290+)
- [uv](https://docs.astral.sh/uv/) package manager
- A Databricks workspace with access to a model serving endpoint (e.g. `databricks-claude-sonnet-4`)
- A configured Databricks CLI profile (`databricks configure --profile <name>`)

## Deployment

### 1. Clone the repository

```bash
git clone <repo-url>
cd dao-ai-apps-example
```

### 2. Install dependencies locally

```bash
uv sync
```

### 3. Configure your agent

Edit `dao_ai.yaml` to set your model, agent prompt, and tools:

```yaml
resources:
  llms:
    default_llm:
      name: databricks-claude-sonnet-4

agents:
  simple_agent:
    name: simple_agent
    model: *default_llm
    tools: []
    prompt: |
      You are a helpful AI assistant.
```

### 4. Generate schema files (optional, for IDE validation)

```bash
databricks bundle schema > bundle_config_schema.json
dao-ai schema > dao_ai_schema.json
```

Both `databricks.yaml` and `dao_ai.yaml` include `yaml-language-server` schema comments for autocompletion and validation in VSCode.

### 5. Deploy the bundle

```bash
databricks bundle deploy --profile <your-profile>
```

This uploads the source files to your workspace and registers the app.

### 6. Run the app

```bash
databricks bundle run dao-ai-apps-example --profile <your-profile>
```

The app will start and output its URL:

```
✓ App started successfully
You can access the app at https://dao-ai-apps-example-<workspace-id>.aws.databricksapps.com
```

### Redeploying after changes

After editing any config file, redeploy and restart:

```bash
databricks bundle deploy --profile <your-profile>
databricks bundle run dao-ai-apps-example --profile <your-profile>
```

### Tearing down

```bash
databricks bundle destroy --profile <your-profile>
```

## How Configs Evolve

This project starts simple, but as your agent grows in complexity, the three config files evolve together.

### Starting point (this project)

A single agent with one LLM and no tools:

- **`dao_ai.yaml`**: one LLM, one agent, a prompt
- **`databricks.yaml`**: one serving endpoint resource
- **`app.yaml`**: one serving endpoint resource

### Adding tools (e.g. Unity Catalog functions)

When you add UC function tools to your agent, the app's service principal needs access:

**`dao_ai.yaml`** gains a tools section:
```yaml
resources:
  llms:
    default_llm:
      name: databricks-claude-sonnet-4
  warehouses:
    default_warehouse:
      warehouse_id: abc123

agents:
  my_agent:
    model: *default_llm
    tools:
      - catalog.schema.my_function
```

**`databricks.yaml`** gains a warehouse resource:
```yaml
resources:
  - name: default_llm
    serving_endpoint:
      name: databricks-claude-sonnet-4
      permission: CAN_QUERY
  - name: default_warehouse
    sql_warehouse:
      id: abc123
      permission: CAN_USE
```

### Adding vector search (RAG)

For retrieval-augmented generation, you add vector store config:

**`dao_ai.yaml`** adds vector stores:
```yaml
resources:
  vector_stores:
    knowledge_base:
      index:
        full_name: catalog.schema.my_index
```

**`databricks.yaml`** adds the vector search index resource.

### Adding Genie rooms, Lakebase, secrets

Each new resource type in `dao_ai.yaml` needs a corresponding entry in the `resources` section of `databricks.yaml` (and `app.yaml`) so the app's service principal gets the right permissions:

| dao_ai.yaml resource | databricks.yaml resource type |
|----------------------|-------------------------------|
| `llms` | `serving_endpoint` (CAN_QUERY) |
| `warehouses` | `sql_warehouse` (CAN_USE) |
| `vector_stores` | UC securable (READ) |
| `genie_rooms` | `genie_space` (CAN_RUN) |
| `databases` (Lakebase) | `database` (CAN_CONNECT_AND_CREATE) |
| Secrets | `secret` (READ) |

### Multi-agent orchestration

For supervisor/multi-agent patterns, `dao_ai.yaml` grows with multiple agents and orchestration config, but `databricks.yaml` only needs additional resources if the new agents use new endpoints or tools.

## How Dependency Management Works

The Databricks Apps runtime supports `uv` natively. During deployment:

1. The runtime detects `pyproject.toml` + `uv.lock` (and no `requirements.txt`)
2. It runs `uv sync` to install all dependencies
3. Then executes the command from `app.yaml` / `databricks.yaml`

This means you never need to `pip install` anything manually in the app command. Just declare dependencies in `pyproject.toml` and keep `uv.lock` up to date with `uv lock`.

## Links

- [dao-ai](https://github.com/natefleming/dao-ai) -- Multi-agent orchestration framework
- [dao-ai Documentation](https://natefleming.github.io/dao-ai)
- [dao-ai-builder](https://github.com/natefleming/dao-ai-builder) -- Full-stack agent builder UI
- [Databricks Apps Documentation](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/)
- [Databricks Asset Bundles](https://docs.databricks.com/dev-tools/bundles/index.html)
