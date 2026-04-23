# Industry packs

Load this reference when the source has industry-specific fields (e.g. turf application rates, crew capacity, warranty tiers, production metrics) that aren't in Adva's core schema. Packs are pre-defined bundles of schema extensions + custom field definitions that a business can enable; applying one before import means those fields are natively valid on the target entity, not custom-fields.

## When to use a pack vs. a custom field

- **Pack**: the source's extra fields are common to a whole vertical or industry (turf/landscaping, HVAC, roofing). Adva ships a named pack for that vertical; applying it is one call.
- **Custom fields**: the extra fields are specific to this one customer (internal job codes, HOA names, referral sources). Pack is overkill — use custom-field CRUD instead. See [custom-fields.md](custom-fields.md).
- If both apply, apply the pack first, then add customer-specific custom fields on top.

## Tool cheatsheet

- `mcp__adva-staging__list_industry_packs` — enumerate available packs. Returns `{name, description, entities_affected, field_count, ...}` per pack.
- `mcp__adva-staging__get_industry_pack(pack_name)` — full definition of one pack (every field it adds, every entity it touches).
- `mcp__adva-staging__preview_pack_apply(pack_name, business_id?)` — **always run this before `apply`**. Returns a diff: which fields will be added, which existing custom fields would conflict, which records might be affected.
- `mcp__adva-staging__apply_industry_pack(pack_name, business_id?)` — commits the pack to the business.
- `mcp__adva-staging__unapply_industry_pack(pack_name, business_id?)` — rolls back the pack application. Data in pack-defined fields on existing records may become orphaned; check the preview before unapplying on a business with real data.

## Safe application pattern

1. `list_industry_packs` → pick a candidate.
2. `get_industry_pack(name)` → read the full field list. Confirm it covers the gap you identified in the source.
3. `preview_pack_apply(name)` → read the diff. Look for conflicts with existing custom fields (same name, different type — this is the only way `apply` will refuse or overwrite).
4. Show the diff to the user and get explicit confirmation. Don't apply silently.
5. `apply_industry_pack(name)`.
6. Re-run `get_import_schema(entity_type)` — the new fields should now appear in the schema.
7. Proceed to the main workflow's mapping step.

## When to unapply

Rarely. Packs are additive; leaving an unused pack enabled has no cost beyond a slightly larger schema surface. Unapply only when:

- The user explicitly wants the pack removed (e.g. a trial decision reversed).
- A bad pack was applied in error and no records have yet been imported into its fields.

If records already reference pack-defined fields, unapplying orphans that data. Talk to the user first.

## Gotchas

- **Packs are per-business, not per-account.** If the account has multiple businesses and the onboarding spans them, apply per business. `list_businesses` tells you which.
- **Packs may add required fields.** After `apply`, existing records without those fields are still valid (they predate the pack), but new imports must supply them. Re-fetch `get_import_schema` to see what's now required.
- **No partial apply.** You can't enable a subset of a pack's fields. If only some fields are wanted, custom-fields is the right tool.
