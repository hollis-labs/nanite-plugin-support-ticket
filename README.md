# Support Ticket Plugin

IT employee self-service support plugin for Conduit. Provides KB search, ticket creation with CRUD, HTML ticket download, and automatic agent profile seeding.

## What it does

- **KB Search**: Connects to a PostgreSQL database with KB articles and exposes `search_kb` / `get_kb_article` MCP tools. Uses tiered full-text search with fuzzy fallback.
- **Ticket CRUD**: In-app ticket store backed by Conduit's SQLite database. REST API at `/api/plugins/tickets`.
- **Ticket Download**: HTML-rendered ticket page for print/PDF at `/api/plugins/ui/support-ticket-download`.
- **Agent Profile**: Auto-seeds an "IT Support" agent with a system prompt tuned for KB search + ticket creation flow.
- **Envelope Components**: 4 React components for rich in-chat UI (KB results, ticket form, confirmation, resolution capture).

## Installation

This plugin is compiled into Conduit (no separate binary). Two options:

### Option A: Symlink (recommended for development)

```bash
cd ~/Projects-apps/fragments-engine/conduit/plugins/
ln -s ../../plugins/support-ticket support
```

### Option B: Import path

Add a replace directive to `conduit/go.mod`:

```
require github.com/hollis-labs/fragments-engine/plugins/support-ticket v0.0.0
replace github.com/hollis-labs/fragments-engine/plugins/support-ticket => ../plugins/support-ticket
```

Then update `conduit/internal/plugin/allplugins/allplugins.go`:

```go
import (
    _ "github.com/hollis-labs/fragments-engine/plugins/support-ticket"
)
```

## KB Database Setup

The KB search feature requires a PostgreSQL database with pg_trgm and the `kb_smart_search` function. Run the setup script against your target database:

```bash
psql --set ON_ERROR_STOP=1 -d kb_demo -f scripts/kb_demo_setup.sql
```

This installs the `pg_trgm` extension and creates the `kb_smart_search` function. The `ON_ERROR_STOP=1` flag ensures failures are not silently ignored (e.g. if pg_trgm is not available on the server).

If importing a full database dump instead, always use:

```bash
psql --set ON_ERROR_STOP=1 -f dump.sql
# or for pg_dump custom format:
pg_restore --exit-on-error -d kb_demo dump.pgdump
```

## Configuration

The plugin resolves config values in this order: environment variable, config file, default.

| Key | Env Var | Default | Description |
|-----|---------|---------|-------------|
| `database_url` | `SUPPORT_DATABASE_URL` | `host=localhost port=5432 dbname=kb_demo sslmode=disable` | PostgreSQL connection string for KB database |

Optional config file: create `config.yaml` alongside `plugin.yaml`:

```yaml
database_url: "host=myhost port=5432 dbname=kb_prod sslmode=require"
```

## Development

### Go backend

The Go package (`supportticket`) registers itself via `init()`. Key files:

- `plugin.go` — Plugin lifecycle, init() registration, Load/Unload
- `kb.go` — KB search MCP transport (Postgres queries)
- `tickets.go` — Ticket store (SQLite) + CRUD handler
- `download.go` — HTML ticket renderer
- `seed.go` — Agent profile auto-seeding

### React frontend

Envelope components in `ui/`:

- `KBResultCard.tsx` — Displays KB search results with expand/collapse
- `TicketFormCard.tsx` — Ticket creation form with validation
- `TicketConfirmationCard.tsx` — Post-creation confirmation card
- `ResolutionCaptureCard.tsx` — Resolution feedback capture
- `TicketInitFlow.tsx` — Two-step ticket creation flow (describe then review)

These are picked up by Conduit's build-time import map script via the `registers.envelopes` section in `plugin.yaml`.

### Testing

```bash
cd ~/Projects-apps/fragments-engine/conduit
go build ./...
go test ./...
```

### Agent profile

The agent YAML is in `agents/it-support.yaml`. The Go code seeds this into the database at plugin load time. Edit the YAML for reference; the Go constant in `seed.go` is the runtime source of truth.

## Directory structure

```
support-ticket/
  plugin.yaml              — Plugin manifest (config, envelopes, endpoints)
  plugin.go                — Entry point, init() registration, Load/Unload
  kb.go                    — KB search MCP transport
  tickets.go               — Ticket store + CRUD handler
  download.go              — HTML ticket download handler
  seed.go                  — Agent profile seeding
  agents/
    it-support.yaml        — Agent profile definition
  ui/
    KBResultCard.tsx        — KB search results envelope
    TicketFormCard.tsx       — Ticket creation form envelope
    TicketConfirmationCard.tsx — Ticket confirmation envelope
    ResolutionCaptureCard.tsx  — Resolution capture envelope
    TicketInitFlow.tsx      — Two-step ticket init flow
  README.md                — This file
```
