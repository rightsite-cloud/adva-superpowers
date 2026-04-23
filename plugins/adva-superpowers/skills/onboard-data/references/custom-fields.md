# Custom fields

Load this reference when the source has fields specific to one customer (HOA name, referral source, internal job codes, inspection notes) that aren't part of Adva's core schema and aren't covered by an industry pack. The decision tree:

- Covered by core schema? → map directly.
- Covered by an industry pack? → apply the pack first. See [industry-packs.md](industry-packs.md).
- Customer-specific? → create a custom field definition here, then reference it under `custom_fields` on each record.
- None of the above and the data isn't actually needed? → drop from the mapping.

## Shape on a record (recap from main skill)

On a record during import, `custom_fields` is a **JSON object** keyed by the custom-field *definition name*:

```json
{
  "external_id": "abc123",
  "source": "sheets",
  "name": "Jane Smith",
  "custom_fields": {
    "HOA Name": "Oak Valley",
    "Referral Source": "Neighbor"
  }
}
```

A string fails validation. The definition for each key must exist on the target entity type before the record is uploaded.

## Tool cheatsheet

- `mcp__adva-staging__list_custom_fields(entity_type?, business_id?)` — enumerate existing custom-field definitions. Filter by entity type.
- `mcp__adva-staging__create_custom_field({entity_type, name, type, required?, ...})` — create one definition.
- `mcp__adva-staging__update_custom_field(field_id, {...})` — update type/required/display attributes. Renames are generally not safe mid-import because record `custom_fields` keys reference the old name.
- `mcp__adva-staging__archive_custom_field(field_id)` — soft-delete. Existing records keep the value; new imports won't accept it.
- `mcp__adva-staging__bulk_upsert_custom_fields([{...}, ...])` — batch create or update. **This is the right tool for onboarding prep** — one call to declare every custom field the mapping needs.
- `mcp__adva-staging__validate_custom_field_value(field_id, value)` — check a single value against the definition. Useful for ambiguous enum-style fields or when the source has dirty data.

## Onboarding prep pattern

1. After step 4 (fetching the import schema), compare source fields to mapped Adva fields. Collect the leftover source fields the user wants to preserve.
2. Decide: for each leftover field, is it customer-specific (custom field), vertical-common (industry pack), or drop?
3. For the custom-field set, build a single `bulk_upsert_custom_fields` call. Use consistent naming (match source header casing if the user will recognize it — `"HOA Name"` beats `"hoa_name"` in a review).
4. Choose types deliberately:
   - `text` — free text, no constraints.
   - `number` — numeric values; specify `precision` if decimals matter.
   - `select` — bounded choice list; provide the allowed values up front.
   - `boolean` — yes/no.
   - `date` — ISO date.
   - Avoid starting everything as `text` "to be safe" — once records land with text values, migrating to a stricter type is painful.
5. Re-fetch `get_import_schema(entity_type)` to confirm the definitions appear, then proceed to mapping.

## Gotchas

- **Custom fields are per-business.** Like packs, definitions live on a business, not the account. Multi-business imports need parallel definitions per business.
- **Definition names are the join key** on a record's `custom_fields` object. Renaming a definition after records exist orphans those values.
- **`required: true` bites backfills.** If the source doesn't have a value for the field on every record, don't mark it required — imports will reject rows without the value. Start optional; tighten later once the data is clean.
- **Duplicate names across entity types are allowed** (same `"HOA Name"` on customer and location). But be intentional — a single definition is not shared across entities; each entity type has its own.
- **Validation before upload.** `validate_custom_field_value` is cheaper than a full `validate_records` when debugging one problem field. Use it in tight loops while sorting out dirty source data.
