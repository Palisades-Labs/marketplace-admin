*Last Edited: 2026-04-21*

# plugin-client-admin

Palisades-Labs admin-only plugin. Consultant/admin-side skills for managing client Claude Code harness engagements.

## Skills

- `/client-admin:manage-credentials` — encrypt and push a client's `credentials.env.age` to their per-client harness repo. Admin provides the passphrase from their own password manager.
- `/client-admin:generate-installer` — emit the decrypt-only bootstrap one-liner for admin to distribute to employees.

## Not distributed to employees

This plugin is installed only on consultant / client-admin machines via the `Palisades-Labs/marketplace-admin` marketplace. It is NOT included in per-client marketplace catalogs, so client employees never see these skills in their Claude Code session.

## Installation

Consultants (and, for self-service clients, admins) add this plugin via the admin marketplace in Claude Desktop:

```
/plugin marketplace add Palisades-Labs/marketplace-admin
/plugin install client-admin@palisades-admin
```

## Repo structure

```
.claude-plugin/plugin.json        # plugin manifest
skills/
├── manage-credentials/
│   └── SKILL.md
└── generate-installer/
    └── SKILL.md
```

## Dependencies

- `age` — install via `brew install age` (Mac) or `winget install FiloSottile.age` (Windows)
- `git`, `gh` (for consultant/admin use)
- `GITHUB_TOKEN` with write access to target client harness repo

## Related repos

- `Palisades-Labs/marketplace-admin` — marketplace catalog that lists this plugin
- `Palisades-Labs/plugin-tools` — separate plugin for employee-facing tools
- `<client-org>/claude-harness` — per-client marketplace + workflow plugin + credentials (one per client)
