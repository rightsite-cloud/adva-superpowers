# Airtable source notes

Load this reference when the user's source is an Airtable base. The [`airtable-mcp-server`](https://www.npmjs.com/package/airtable-mcp-server) MCP server must be installed and configured with a Personal Access Token (PAT). If `/mcp` doesn't list an `airtable` server, ask the user to add one before proceeding.

## Tool cheatsheet

- `mcp__airtable__list_bases` — enumerate bases the PAT can reach.
- `mcp__airtable__list_tables(baseId, { tableIdentifiersOnly: true })` — get table IDs. **Always pass `tableIdentifiersOnly: true`** on the first call; without it, a rich table can return 60KB+ of field schema and blow the context budget.
- `mcp__airtable__describe_table(baseId, tableId)` — full schema for one table. Expect large output; after reading, extract just `fields[].name` and `fields[].type` for the mapping draft.
- `mcp__airtable__list_records(baseId, tableId, { maxRecords: 2 })` — **always cap `maxRecords`**. Records with attachments, buttons, or long rich-text fields are huge.
- `mcp__airtable__search_records` — useful for pulling a targeted sample (e.g., "records with a non-empty email") without scanning the whole table.

## Quirks

- **Table name vs ID.** The MCP requires the `tblXXX` table ID, not the display name. If a tool errors with "Table &lt;name&gt; not found", call `list_tables` first to resolve.
- **Field names are case-sensitive and may have trailing spaces.** A field that displays as `Active` may actually be stored as `"Active "`. Always grab the exact string from `describe_table`.
- **Formulas, rollups, and lookups are computed fields.** These must **not** be mapped to Adva fields — Adva computes its equivalents natively. Common offenders in home-services Airtable bases: anything that looks like a grand total, final price, tax amount, line total, or balance. When `describe_table` reports a field of type `formula`, `rollup`, or `multipleLookupValues`, skip it.
- **"DEV |" cross-base join keys are Airtable automation artifacts.** They link records across bases inside Airtable and have no meaning outside it. Map each source record through its own primary key (`external_id = recordId`); don't try to preserve these join keys.
- **`multipleAttachments` and `button` fields return large URLs.** Skip them on the initial import unless the user specifically wants attachment metadata. If they do, pull attachment URLs in a second pass once the parent records exist.
- **Single-select and multi-select values are strings, not IDs.** They map cleanly to Adva enum fields once the values are translated (e.g. Airtable `"Commercial"` → Adva `"commercial"`). See the main skill's "enum translation" note.

## Idempotency

Use the Airtable `recordId` (e.g. `rec001ET0HHTfEFZx`) as `external_id` and `"airtable"` as `source` on every uploaded record. Re-running the import then upserts against the existing Adva record rather than duplicating.

## Typical flow

1. `mcp__airtable__list_bases` → pick the target base.
2. `mcp__airtable__list_tables` with `tableIdentifiersOnly: true` → pick the target table.
3. `mcp__airtable__describe_table` once → extract field names + types.
4. `mcp__airtable__list_records` with `maxRecords: 2` → get representative sample values.
5. Drop computed fields, attachments, and cross-base join keys from the mapping.
6. Return to the main onboarding workflow (starting at step 2, "List Adva's target entities").
