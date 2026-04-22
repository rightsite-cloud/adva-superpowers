---
name: onboard-data
description: Use when the user wants to import, onboard, or migrate data from an external source (CSV, spreadsheet, API export, or another system like Airtable, Google Sheets, Jobber, Salesforce) into the Adva platform. Covers source inspection, entity mapping, validation, idempotent uploads via the adva-staging MCP, and troubleshooting common friction.
---

# Onboard Data to Adva

Guide an agent through importing data from any external source into Adva. This skill is **source-agnostic** — whatever the source system is (spreadsheet, API, database dump, hand-built CSV), the workflow is the same once the agent can read rows from it. Source-specific quirks live in separate reference files, loaded only when relevant.

## The core workflow

1. **Inspect the source.** Enumerate what's there (tables / sheets / files) and pull a small sample (3–5 rows) from each candidate table. Use whatever read path is connected — an MCP server for the source, the `Read` tool for local files, or a hand-written CSV. Avoid pulling the full dataset into context; samples are enough to draft mappings.

2. **List Adva's target entities.** Call `mcp__adva-staging__list_entity_types`. Entities are organized into tiers (1–6); **lower tiers must be imported before higher ones**. Typical order:
   - Tier 1: products, territories, brands
   - Tier 2: customers, team members
   - Tier 3: locations
   - Tier 4–5: proposals, jobs, invoices
   - Tier 6: transactions, commissions

3. **Fetch the target schema.** Call `mcp__adva-staging__get_import_schema` with the entity type. The returned schema is the contract — field names, types, required flags, enum constraints. Treat unknown fields as "drop from mapping" rather than "pass through".

4. **Draft a mapping.** Produce an explicit `{source_field → adva_field}` mapping by comparing the schema to the source columns and sample values. Show the mapping to the user and get confirmation before proceeding. The MCP exposes `mcp__adva-staging__suggest_column_mapping` for AI-assisted drafting, but it may return a Workers AI error code (5024); if it does, fall back to manual mapping from the schema + samples.

5. **Validate first.** Send a small batch (3–5 records) through `mcp__adva-staging__validate_records`. This is a dry run — it reports errors without writing. Surface every error to the user. Fix the mapping, drop bad rows, or ask about ambiguous values before moving on.

6. **Upload in chunks.** Once the batch validates cleanly, call `mcp__adva-staging__upload_records` with the full set. Chunk large imports (roughly 50–200 records per call). If the user has a well-formed CSV already, `mcp__adva-staging__upload_csv` is a one-shot alternative — the server parses it.

7. **Poll status.** Uploads are async — `upload_records` returns a `job_id`. Call `mcp__adva-staging__get_import_status(job_id)` until it reports completion. Expect 10–30s of queue latency before processing actually starts; that's normal, not a hang.

8. **Verify idempotency.** Call `mcp__adva-staging__get_external_ids` with the same `source` value to confirm records are linked. Re-running the import should upsert (match by `external_id`), not duplicate.

## Non-negotiable rules

These are Adva platform facts. Violating them breaks idempotency, produces bad data, or fails validation.

- **Always set `external_id` and `source` on every record.** `external_id` is the source system's primary key; `source` is a short string identifying the system (e.g. `"csv_upload"`, `"sheets"`, `"jobber"`). Without these, every re-run creates duplicates.
- **Cross-entity references use `*_external_id` fields, never Adva UUIDs.** E.g. on a proposal, set `customer_external_id` to the source's customer ID. The import API resolves these to Adva UUIDs via an external-id index.
- **Never send computed values.** Totals, margins, balances, tax amounts, line totals — Adva computes these natively from the constituent fields. If the source has them, drop them from the mapping rather than passing them through.
- **Respect entity tiers.** A tier-4 entity (e.g. proposal) that references a tier-2 entity (customer) requires the customer to already exist in Adva. If you try to import them in the wrong order, validation fails.
- **`custom_fields` is a JSON object, not a string.** The schema may describe it as a generic field, but the validator requires an object keyed by custom-field name, e.g. `{"HOA Name": "Oak Valley"}`.

## Building a CSV when the source doesn't expose one

If the source is rich (Airtable, a database) but the user prefers to review data before upload, an intermediate CSV can be a good pattern:

1. Inspect the source, extract only the columns in the current mapping.
2. Write a CSV with headers matching the mapped Adva field names (plus `external_id` and `source`).
3. Let the user review / edit the file.
4. Upload it via `mcp__adva-staging__upload_csv` or by reading it back and calling `upload_records`.

This is especially useful when the user wants to spot-check name cleanup, enum translation, or sensitive fields before anything lands in Adva.

## Source-specific guidance

Some source systems have quirks worth knowing. Load the relevant reference **only if** the user's source matches — otherwise keep it out of context.

- **Airtable** → [references/airtable.md](references/airtable.md)

If the user's source isn't in this list, proceed with the core workflow. If you hit friction that would have been useful to document for the next user, capture it — a new reference file for that source is the right follow-up.

## Common gotchas

- **Enum translation.** Source values rarely match Adva's enums out of the box (e.g. source has `"Business"` / `"Individual"`, Adva expects `"commercial"` / `"residential"`). Read the schema's `enum` constraints and build a per-field translation map. Validation errors will point at the offending field — use them.
- **Secondary contacts.** Many sources collapse primary + secondary customer onto one record. Adva's customer entity is per-contact; a second contact is its own customer record, linked via the shared deal or account.
- **Addresses are separate entities.** Billing / service addresses belong on a `location` record, not directly on a customer. Customer → location is 1:N.
- **Field names are exact.** Adva field names are snake_case and case-sensitive. Don't normalize spaces or change capitalization when building mappings.

## What this skill does NOT do

- **Authentication setup.** The adva-superpowers plugin wires the MCP; first tool call prompts SSO if the user isn't signed in.
- **Complex transformations.** If the source needs deduping, record splitting, or multi-row aggregation, handle those steps explicitly with the user before running the onboarding workflow — they're not automated here.
- **Source discovery when no MCP server or file access is available.** If the user's source has no connector yet and isn't a local file, the first step is getting read access, not improvising an import.
