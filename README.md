# Adva Superpowers

A [Claude Code](https://claude.com/claude-code) plugin marketplace that wires up MCP access and skills for working with the [Adva](https://getadva.ai) platform.

## What you get

Installing the `adva-superpowers` plugin auto-configures:

- **`adva-staging` MCP server** — 11 tools for listing entity types, fetching import schemas, validating records, uploading data, checking import status, and looking up external IDs against the Adva staging environment.
- **Skills** (coming soon) — source-agnostic onboarding workflows for mapping external data (Airtable, Google Sheets, CSV, etc.) into Adva entities.

## Install

```bash
# From inside Claude Code
/plugin marketplace add rightsite-cloud/adva-superpowers
/plugin install adva-superpowers@adva-superpowers
```

Restart Claude Code after install. The first time you use an `adva-staging` tool, you'll be prompted to complete an SSO sign-in to Adva — this is a one-time OAuth dance that Claude Code caches for future sessions.

## Prerequisites

- An Adva staging account. If you don't have one, ask your Adva contact.
- Claude Code ≥ the version that supports `/plugin` (check with `/plugin --help`).

## Updating

Claude Code refreshes the marketplace automatically at session start. To force a refresh:

```bash
/plugin marketplace update
```

## What's here

```text
.claude-plugin/
  marketplace.json            # Registry — lists plugins available in this marketplace
plugins/
  adva-superpowers/
    .claude-plugin/
      plugin.json             # Plugin manifest — declares MCP servers + skills
    skills/                   # Skill files live here
```

## License

MIT — see [LICENSE](LICENSE).
