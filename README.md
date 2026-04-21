*Last Edited: 2026-04-21*

# marketplace-admin

Palisades-Labs admin marketplace catalog. Lists the `client-admin` plugin for consultant / client-admin installs. **Not distributed to employees** — per-client marketplace catalogs (in each `<client-org>/claude-harness` repo) never reference this marketplace.

## Installation

Consultants (and, for self-service clients, admins) add in Claude Desktop:

```
/plugin marketplace add Palisades-Labs/marketplace-admin
/plugin install client-admin@palisades-admin
```

## Contents

A single-plugin catalog:

```json
{
  "name": "palisades-admin",
  "plugins": [
    { "name": "client-admin", "source": { "source": "github", "repo": "Palisades-Labs/plugin-client-admin" } }
  ]
}
```

The `client-admin` plugin's source is `Palisades-Labs/plugin-client-admin` — resolved at install time via cross-repo source reference.

## Related repos

- `Palisades-Labs/plugin-client-admin` — admin plugin source (manage-credentials, generate-installer)
- `Palisades-Labs/plugin-tools` — employee-facing plugin (separate catalog path via per-client marketplaces)
- `<client-org>/claude-harness` — per-client marketplace + workflow plugin + credentials
