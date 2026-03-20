# Content CLI Skills

A reference of all commands and capabilities provided by the Celonis Content CLI.

---

## Profile Management

Profiles store connection details for Celonis Platform environments.

| Command | Description |
|---|---|
| `profile list` | List all stored profiles |
| `profile create` | Create a new profile (API Token or OAuth) |
| `profile default <name>` | Set the default profile |
| `profile secure <name>` | Migrate profile secrets to the system keychain |

**Authentication options:** API Token, OAuth Device Code, OAuth Client Credentials

---

## Git Profile Management

Git profiles store repository credentials used for config export/import workflows.

| Command | Description |
|---|---|
| `git profile create` | Create a new Git profile |
| `git profile list` | List all Git profiles |
| `git profile default <name>` | Set the default Git profile |

---

## Studio — Packages

| Command | Description |
|---|---|
| `list packages` | List all Studio packages (`--json` for machine-readable output) |
| `list spaces` | List all spaces |
| `pull package` | Pull a published package (`--draft` for draft, `--store` for app store) |
| `push package` | Push a package to Studio (`--overwrite` to replace existing) |
| `push packages` | Push multiple packages from a directory |

---

## Studio — Assets

| Command | Description |
|---|---|
| `list assets` | List assets by type within a package |
| `pull asset` | Pull a single asset from a package |
| `push asset` | Push a single asset to a package |
| `push assets` | Push multiple assets to a package |
| `push widget` | Push a widget to Studio |

---

## Studio — Analyses & Bookmarks

| Command | Description |
|---|---|
| `pull bookmarks` | Pull analysis bookmarks |
| `push bookmarks` | Push analysis bookmarks |

---

## Studio — Variables

| Command | Description |
|---|---|
| `list assignments` | List variable assignment values for a package |

---

## Configuration Management

Batch export/import of OCDM packages with Git integration.

| Command | Description |
|---|---|
| `config list` | List packages (supports filtering) |
| `config export` | Batch export packages (optionally to a Git repository) |
| `config import` | Batch import packages (optionally from a Git repository) |
| `config diff` | Diff package configurations between environments |
| `config metadata export` | Show unpublished changes for a package |

### Variables

| Command | Description |
|---|---|
| `config variables list` | List variables defined in a package |

### Versions

| Command | Description |
|---|---|
| `config versions get` | Get version metadata for a package |
| `config versions create` | Create a new package version (supports `--bump-major`, `--bump-minor`, `--bump-patch`) |

### Nodes

| Command | Description |
|---|---|
| `config nodes find` | Find a node configuration by key |
| `config nodes list` | List all nodes in a package |
| `config nodes diff` | Diff node configurations between versions |
| `config nodes dependencies list` | List node dependencies |

---

## Data Pools & Connections

| Command | Description |
|---|---|
| `list data-pools` | List all data pools |
| `pull data-pool` | Pull a single data pool |
| `push data-pool` | Push a single data pool |
| `push data-pools` | Push multiple data pools from a directory |
| `update data-pool` | Update an existing data pool |
| `export data-pool` | Export a data pool with all dependencies |
| `import data-pools` | Batch import multiple data pools |
| `list connection` | List connections within a data pool |
| `get connection` | Get properties of a specific connection |
| `set connection` | Update connection properties |

---

## Action Flows & Skills

| Command | Description |
|---|---|
| `analyze action-flows` | Analyze action flow dependencies |
| `export action-flows` | Export action flows with their dependencies |
| `import action-flows` | Import action flows |
| `pull skill` | Pull a skill from a project |
| `push skill` | Push a skill to a project |

---

## Deployments _(beta)_

| Command | Description |
|---|---|
| `deployment create` | Create a new deployment |
| `deployment list history` | List deployment history (supports filtering by date, status) |
| `deployment list active` | List currently active deployments |
| `deployment list deployables` | List available deployables |
| `deployment list targets` | List available deployment targets |

---

## CPM4 Transport Files

| Command | Description |
|---|---|
| `push ctp` | Push a Celonis 4 transport (`.CTP`) file, with optional analysis and data model extraction |

---

## Global Options

These options apply to every command:

| Flag | Description |
|---|---|
| `-p, --profile <name>` | Use a specific profile instead of the default |
| `--gitProfile <name>` | Use a specific Git profile |
| `-q, --quietmode` | Reduce log output |
| `--debug` | Print debug-level messages |
| `--dev` | Enable development mode |
| `-V, --version` | Print the CLI version |

---

## Further Reading

- [Getting Started](docs/getting-started.md) — installation, profiles, authentication
- [Studio Commands](docs/user-guide/studio-commands.md)
- [Config Commands](docs/user-guide/config-commands.md)
- [Data Pool Commands](docs/user-guide/data-pool-commands.md)
- [Action Flow Commands](docs/user-guide/action-flow-commands.md)
- [Deployment Commands](docs/user-guide/deployment-commands.md)
