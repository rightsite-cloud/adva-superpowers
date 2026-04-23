---
name: onboard-data
description: Use when the user wants to import, onboard, or migrate data from an external source (CSV, spreadsheet, API export, or another system like Airtable, Google Sheets, Jobber, Salesforce) into the Adva platform. Covers business + target selection, entity mapping, industry packs, custom-field creation, validation, idempotent uploads via the adva-staging MCP, and troubleshooting common friction.
---

# Onboard Data to Adva

Guide an agent through importing data from any external source into Adva. This skill is **source-agnostic** — whatever the source system is (spreadsheet, API, database dump, hand-built CSV), the workflow is the same once the agent can read rows from it. Source-specific quirks and advanced target-side options live in separate reference files, loaded only when relevant.

## The core workflow

1. **Pick the business.** Call `mcp__adva-staging__list_businesses` to see which businesses the signed-in account can write to. Data lands in exactly one business per run. If the account has only one business, skip this step — the tools infer it. If the user hasn't named one explicitly and there are multiple, ask before proceeding.

2. **Inspect the source.** Enumerate what's there (tables / sheets / files) and pull a small sample (3–5 rows) from each candidate table. Use whatever read path is connected — an MCP server for the source, the `Read` tool for local files, or a hand-written CSV. Avoid pulling the full dataset into context; samples are enough to draft mappings.

3. **List Adva's target entities.** Call `mcp__adva-staging__list_entity_types`. Entities are organized into tiers (1–6); **lower tiers must be imported before higher ones**. Typical order:
   - Tier 1: products, territories, brands
   - Tier 2: customers, team members
   - Tier 3: locations
   - Tier 4–5: proposals, jobs, invoices
   - Tier 6: transactions, commissions

4. **Fetch the target schema.** Call `mcp__adva-staging__get_import_schema` with the entity type. The returned schema is the contract — field names, types, required flags, enum constraints. Treat unknown fields as "drop from mapping" rather than "pass through".
   - If the user wants a **CSV scaffold to fill in or review**, `mcp__adva-staging__get_csv_template` returns a headered CSV for the same entity type — a good intermediate artifact for human review before upload.
   - If you need to drill into a **specific field's enum or constraints**, `mcp__adva-staging__get_field_schema` returns detail for one field at a time (useful during enum translation).

   **Optional — prepare the target before mapping.** Before drafting mappings, check whether the source has data that doesn't fit the current schema. Two mechanisms close that gap:
   - **Industry packs** — bundles of schema extensions the business can enable. If the source has industry-specific fields (e.g., turf application rates, crew capacity, warranty tiers), an available pack may already define them natively. See [references/industry-packs.md](references/industry-packs.md) for the preview-then-apply workflow.
   - **Custom fields** — arbitrary business-defined fields. If the source has fields unique to this customer (HOA name, referral source, internal job codes), create custom-field definitions on the target entity first; the import validator then accepts those keys under `custom_fields`. See [references/custom-fields.md](references/custom-fields.md).

   Both are optional. Most imports don't need either.

5. **Draft a mapping.** Produce an explicit `{source_field → adva_field}` mapping by comparing the schema to the source columns and sample values. Show the mapping to the user and get confirmation before proceeding. The MCP exposes `mcp__adva-staging__suggest_column_mapping` for AI-assisted drafting, but it may return a Workers AI error code (5024); if it does, fall back to manual mapping from the schema + samples.

6. **Validate first.** Send a small batch (3–5 records) through `mcp__adva-staging__validate_records`. This is a dry run — it reports errors without writing. Surface every error to the user. Fix the mapping, drop bad rows, or ask about ambiguous values before moving on.

7. **Upload in chunks.** Once the batch validates cleanly, call `mcp__adva-staging__upload_records` with the full set. Chunk large imports (roughly 50–200 records per call). If the user has a well-formed CSV already, `mcp__adva-staging__upload_csv` is a one-shot alternative — the server parses it.

8. **Poll status.** Uploads are async — `upload_records` returns a `job_id`. Call `mcp__adva-staging__get_import_status(job_id)` until it reports completion. Expect 10–30s of queue latency before processing actually starts; that's normal, not a hang.

9. **Verify idempotency.** Call `mcp__adva-staging__get_external_ids` with the same `source` value to confirm records are linked. Re-running the import should upsert (match by `external_id`), not duplicate. `mcp__adva-staging__list_import_jobs` gives an audit trail of prior runs — useful to confirm "did this already land?" before kicking off a re-run.

## Non-negotiable rules

These are Adva platform facts. Violating them breaks idempotency, produces bad data, or fails validation.

- **Always set `external_id` and `source` on every record.** `external_id` is the source system's primary key; `source` is a short string identifying the system (e.g. `"csv_upload"`, `"sheets"`, `"jobber"`). Without these, every re-run creates duplicates.
- **Cross-entity references use `*_external_id` fields, never Adva UUIDs.** E.g. on a proposal, set `customer_external_id` to the source's customer ID. The import API resolves these to Adva UUIDs via an external-id index.
- **Never send computed values.** Totals, margins, balances, tax amounts, line totals — Adva computes these natively from the constituent fields. If the source has them, drop them from the mapping rather than passing them through.
- **Respect entity tiers.** A tier-4 entity (e.g. proposal) that references a tier-2 entity (customer) requires the customer to already exist in Adva. If you try to import them in the wrong order, validation fails.
- **`custom_fields` is a JSON object, not a string.** The value on a record is an object keyed by custom-field definition name, e.g. `{"HOA Name": "Oak Valley"}`. A string will fail validation. If you want to *define* a new custom field, that's a separate step — see [references/custom-fields.md](references/custom-fields.md).

## Building a CSV when the source doesn't expose one

If the source is rich (Airtable, a database) but the user prefers to review data before upload, an intermediate CSV can be a good pattern:

1. Fetch a CSV scaffold with `mcp__adva-staging__get_csv_template` (headers already match the schema).
2. Inspect the source, extract only the columns in the current mapping, and fill the scaffold.
3. Include `external_id` and `source` columns.
4. Let the user review / edit the file.
5. Upload via `mcp__adva-staging__upload_csv`, or read it back and call `upload_records`.

This is especially useful when the user wants to spot-check name cleanup, enum translation, or sensitive fields before anything lands in Adva.

## Source-specific guidance

Some source systems have quirks worth knowing. Load the relevant reference **only if** the user's source matches — otherwise keep it out of context.

- **Airtable** → [references/airtable.md](references/airtable.md)

If the user's source isn't in this list, proceed with the core workflow. If you hit friction that would have been useful to document for the next user, capture it — a new reference file for that source is the right follow-up.

## Advanced target-side options

Load only when the onboarding actually needs schema extension or customer-specific fields:

- **Industry packs** → [references/industry-packs.md](references/industry-packs.md) — preview, apply, and unapply schema bundles on a business.
- **Custom fields** → [references/custom-fields.md](references/custom-fields.md) — create, update, archive, and bulk-upsert business-defined fields; validate a value against a definition.

## Common gotchas

- **Enum translation.** Source values rarely match Adva's enums out of the box (e.g. source has `"Business"` / `"Individual"`, Adva expects `"commercial"` / `"residential"`). Read the schema's `enum` constraints and build a per-field translation map. `mcp__adva-staging__get_field_schema` is the fastest way to drill into one field's allowed values. Validation errors will also point at the offending field — use them.
- **Secondary contacts.** Many sources collapse primary + secondary customer onto one record. Adva's customer entity is per-contact; a second contact is its own customer record, linked via the shared deal or account.
- **Addresses are separate entities.** Billing / service addresses belong on a `location` record, not directly on a customer. Customer → location is 1:N.
- **Field names are exact.** Adva field names are snake_case and case-sensitive. Don't normalize spaces or change capitalization when building mappings.

## What this skill does NOT do

- **Authentication setup.** The adva-superpowers plugin wires the MCP; first tool call prompts SSO if the user isn't signed in.
- **Complex transformations.** If the source needs deduping, record splitting, or multi-row aggregation, handle those steps explicitly with the user before running the onboarding workflow — they're not automated here.
- **Source discovery when no MCP server or file access is available.** If the user's source has no connector yet and isn't a local file, the first step is getting read access, not improvising an import.
