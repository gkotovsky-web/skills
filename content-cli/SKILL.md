---
name: content-cli
description: Assists with developing and using the Celonis content-cli tool — a TypeScript CLI for managing content (packages, configs, data pools, action flows) across Celonis Platform environments. Always use this skill when the user mentions content-cli, wants to migrate or sync packages between Celonis teams, export configs to Git, set up CI/CD for Celonis content, or is adding/debugging commands in the content-cli repo — even if they just say something like "how do I move this package to production" or "sync my Celonis config."
---

# Content CLI

## Quick start

```bash
yarn install && yarn build
content-cli profile create
content-cli pull package -p <profile> --key <package-key>
content-cli push package -p <target-profile> --spaceKey <space-key> -f <package.zip>
```

## Key workflows

**Migrate content between teams:** pull from source profile → push to target profile

**Config sync with Git:**
```bash
content-cli config export -p <profile> --packageKeys key1 key2 --gitProfile <git-prof> --gitBranch main
content-cli config import -p <profile> --gitProfile <git-prof> --gitBranch main
```

**CI/CD (no profile):**
```bash
CELONIS_URL=https://team.celonis.cloud CELONIS_API_TOKEN=<token> content-cli <command>
```

## Command reference

### Profiles
| Command | Description |
|---|---|
| `profile create` | Create profile (API Token or OAuth) |
| `profile list` | List profiles |
| `profile default <name>` | Set default profile |
| `profile secure <name>` | Migrate secrets to system keychain |
| `git profile create/list/default` | Manage Git profiles for repo integration |

### Studio
| Command | Description |
|---|---|
| `list packages/spaces/assets` | List Studio content |
| `pull package` | Pull published package (`--draft`, `--store`) |
| `push package/packages` | Push package(s) to Studio |
| `pull/push asset/assets` | Transfer individual assets |
| `push widget` | Push custom widget |
| `pull/push bookmarks` | Transfer analysis bookmarks |
| `list assignments` | List variable assignments |

### Configuration management
| Command | Description |
|---|---|
| `config list` | List packages |
| `config export` | Batch export packages (supports `--gitProfile`, `--withDependencies`) |
| `config import` | Batch import packages |
| `config diff` | Diff configs between environments |
| `config metadata export` | Show unpublished changes |
| `config variables list` | List package variables |
| `config versions get/create` | Manage versions (`--bump-major/minor/patch`) |
| `config nodes find/list/diff` | Inspect and diff nodes |
| `config nodes dependencies list` | List node dependencies |

### Data pools & connections
| Command | Description |
|---|---|
| `export/import data-pool(s)` | Transfer data pools with dependencies |
| `list/pull/push/update data-pool` | CRUD on data pools |
| `list/get/set connection` | Manage connections within a data pool |

### Action flows & skills
| Command | Description |
|---|---|
| `analyze action-flows` | Analyze dependencies |
| `export/import action-flows` | Transfer action flows |
| `pull/push skill` | Transfer skills |

### Deployments _(beta)_
| Command | Description |
|---|---|
| `deployment create` | Create deployment |
| `deployment list history/active/deployables/targets` | Query deployment state |

### CPM4
| Command | Description |
|---|---|
| `push ctp` | Push legacy `.CTP` transport file |

### Global options
`-p <profile>` · `--gitProfile <name>` · `-q` · `--debug` · `--dev`

## Architecture (for development)

- **Entry**: `src/index.ts` → command modules in `src/commands/` auto-discovered
- **Adding a command**: add `*-command.service.ts` in the relevant module folder
- **Context**: carries profile, logger, and authenticated Axios client — passed to all commands
- **Profile storage**: `~/.celonis-content-cli-profiles`, secrets in system keychain
