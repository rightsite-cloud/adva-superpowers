---
name: onboard-data
description: Use when the user wants to import, onboard, or migrate data from any
  external source (CSV, spreadsheet, API export, Airtable, Google Sheets, Jobber,
  Salesforce) into the Adva platform. Also use when the user hits import errors,
  validation failures, or data model questions — this skill explains Adva's contact
  identity model, entity tiers, and common failure patterns. Covers business selection,
  entity mapping, industry packs, custom-field creation, validation, idempotent uploads
  via the adva-staging MCP, and automatic Linear bug filing when platform issues are found.
---

# Onboard Data to Adva

Guide an agent through importing data from any external source into Adva. This skill is **source-agnostic** — whatever the source system is (spreadsheet, API, database dump, hand-built CSV), the workflow is the same once the agent can read rows from it. Source-specific quirks and advanced target-side options live in separate reference files, loaded only when relevant.

## Data model foundations

Before importing, understand the key invariants that govern how Adva stores data. Load [references/data-model.md](references/data-model.md) for the full picture. Key points:

- **Multi-tenant hierarchy**: Account → Business → Brand. Every import targets exactly one business. `business_id` scopes all data.
- **Unified Contact Model**: `core.contacts` is a global identity store. A contact becomes a "customer" or "team member" via a `contact_role` row scoped to a specific business. The same person can hold roles in multiple businesses — they share one contact record.
- **Email is the global identity key**: `UNIQUE INDEX idx_contacts_email` enforces one contact per email address across the entire account. The import pipeline uses an upsert — if the email already exists, a new role is attached to the existing contact. Post-ADV-749, `idx_contacts_email` surfacing as a raw error is a platform bug, not a data error.
- **Phone is a soft match signal, not a unique key**: Normalized to E.164 via trigger. Multiple contacts can share a phone number. `find_duplicate_contacts()` uses phone for advisory matching only — no uniqueness enforced.
- **Entity tiers**: Tier-3+ records have FK constraints on tier-2. A proposal (`customer_id` FK) requires the customer to exist before the proposal can import. Wrong order = FK violation. Always import lower tiers first.
- **Platform-bug filing is built in**: The Adva MCP server files platform bugs on the user's behalf via `mcp__adva-staging__report_import_problem` — the user needs no Linear account. At the first sign of a platform bug you'll call this tool and tell them: *"Logged as `<id>` — we're on it."* If the server isn't configured to file yet, you'll surface what would have been filed instead; either way, the user keeps moving.

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
   - To get a pre-headered CSV scaffold, call `mcp__adva-staging__get_csv_template` — use this as the basis for the local CSV file you will write in step 7.
   - If you need to drill into a **specific field's enum or constraints**, `mcp__adva-staging__get_field_schema` returns detail for one field at a time (useful during enum translation).

   **Optional — prepare the target before mapping.** Before drafting mappings, check whether the source has data that doesn't fit the current schema. Two mechanisms close that gap:
   - **Industry packs** — bundles of schema extensions the business can enable. If the source has industry-specific fields (e.g., turf application rates, crew capacity, warranty tiers), an available pack may already define them natively. See [references/industry-packs.md](references/industry-packs.md) for the preview-then-apply workflow.
   - **Custom fields** — arbitrary business-defined fields. If the source has fields unique to this customer (HOA name, referral source, internal job codes), create custom-field definitions on the target entity first; the import validator then accepts those keys under `custom_fields`. See [references/custom-fields.md](references/custom-fields.md).

   Both are optional. Most imports don't need either.

5. **Draft a mapping.** Produce an explicit `{source_field → adva_field}` mapping by comparing the schema to the source columns and sample values. Show the mapping to the user and get confirmation before proceeding. The MCP exposes `mcp__adva-staging__suggest_column_mapping` for AI-assisted drafting, but it may return a Workers AI error code (5024); if it does, fall back to manual mapping from the schema + samples.

6. **Test with a small sample first.** There is no separate dry-run tool — validation runs server-side as part of the import itself. To catch mapping mistakes cheaply before committing the full dataset, write a 3–5 row CSV with the confirmed mapping and run it through the upload flow in step 7 (`get_upload_url` → PUT → `start_csv_import`). Poll `get_import_status`, then surface every error to the user — fix the mapping, drop bad rows, or ask about ambiguous values before importing the rest. The sample is safe to run for real: imports upsert on `(source, external_id)`, so the sample rows simply update in place during the full import — no duplicates, nothing to undo.

7. **Write a local CSV file, then upload it via the R2 pre-signed PUT flow.** Once the sample import looks clean, fetch **all** records from the source, apply the confirmed mapping, and write them to a CSV file in the user's current working directory. Name it `{entity_type}_{source}.csv` (e.g. `team_member_airtable.csv`). Then upload in three steps so the file bytes never travel through the LLM context:

   ```
   # 7a. Mint a pre-signed PUT URL — the upload_token bundles the import intent.
   { upload_url, headers, upload_token } =
     mcp__adva-staging__get_upload_url({
       entity_type: "team_member",
       source: "airtable",
       filename: "team_member_airtable.csv",
       content_type: "text/csv"
     })

   # 7b. Upload the file bytes directly to R2. Use the Bash tool — never load
   # the file into context. The `headers` object must be echoed verbatim.
   curl -X PUT --data-binary @team_member_airtable.csv \
     -H "Content-Type: text/csv" \
     "<upload_url>"

   # 7c. Kick off the import. The server reads the object from R2, runs the
   # same SHA-256 dedup + Workflow flow as the REST /upload endpoint.
   mcp__adva-staging__start_csv_import({ upload_token })
   ```

   **Rules for this step:**
   - Always write the CSV to disk first — this gives the user an artifact they can inspect or re-upload later, and keeps the file bytes out of the LLM context.
   - `get_upload_url` + `start_csv_import` is the **only** way to submit import data, regardless of file size. The bytes travel disk → R2 directly and never flow through the LLM context window, so no value — notably `external_id` — can be altered or invented along the way.
   - The pre-signed URL is valid for 15 minutes; the `upload_token` for 30 minutes. If either expires, mint a fresh pair.
   - Never write an intermediate JSON file. If records are already in context, write them directly to CSV.
   - `custom_fields` must be a JSON-encoded object in the CSV cell, e.g. `{"HOA Name": "Oak Valley"}` — not a plain string.
   - **ADV-1046 (merged + deployed 2026-05-20)** moved the import pipeline onto a durable Cloudflare Workflow — the historic 200-record stall on large imports is fixed. Imports up to 25 MiB now run end-to-end through one call.

8. **Poll for completion.** `start_csv_import` returns a `job_id`. The import processes asynchronously — call `mcp__adva-staging__get_import_status(job_id)` immediately after submitting, then again every 30 seconds until the status reaches a terminal state (`completed`, `failed`, or `awaiting_review`). Expect 10–30 seconds of queue latency before processing begins — that is normal.

   Terminal states and what to do next:
   - `completed` → proceed to step 9
   - `awaiting_review` → the tool returns a `next_action` directing you to `get_normalization_summary`; call `mcp__adva-staging__get_normalization_summary`. If `normalization_result.needs_manual_review` is non-empty, present each item to the user with a plain-language explanation and ask: "Accept the normalization as-is?" or "Revert these specific changes?" before calling `mcp__adva-staging__approve_import`.
   - `failed` → surface the error to the user; use `mcp__adva-staging__retry_normalization` if appropriate

9. **Review row-level errors (if any).** When polling reaches a terminal state, check the `next_action` field:

   a. If `next_action` is null (or `action` is not `review_errors`), skip to step 10.

   b. If `next_action.action == "review_errors"`, the import finished with row failures. Build a grouped error summary from `job.errors[]` — group by `field` + `message` pattern, show the count per group and one sample row. Show it to the user like:

      ```
      Import completed: 4,837 of 5,000 records imported. 163 records failed:

        Field 'status' — enum value not recognized (142 rows)
          Sample: row 12 — value "Active " (trailing space)

        Missing required field 'email' (21 rows)
          Sample: row 8 — external_id: "CUST-0042"
      ```

      Note: `job.warnings[]` contains coercion notices (type conversions applied automatically). Show these separately from `job.errors[]` (hard failures).

   b-bis. **Get an AI diagnosis before classifying.** Call `mcp__adva-staging__diagnose_import_failure({ job_id })`. It assembles this run's reconciliation diff, row errors, R2 artifacts, and Workers Logs (filtered by `importJobId`) and returns a structured diagnosis: `root_cause` (`category`: `data` | `platform` | `config` | `transient`), cited `evidence[]`, ordered `suggested_remediation[]`, `retry_safe`, and a `file_linear_ticket` recommendation. **Use `root_cause.category` as the primary signal** for the data-issue-vs-platform-bug decision in the Failure interpretation section — it is more reliable than matching error strings. When `file_linear_ticket.recommended` is true, call `mcp__adva-staging__report_import_problem({ job_id, title, body })` using the returned `title`/`body` (PII already stripped). The tool returns `{ filed, identifier, url }`; see the "Platform bugs" bullet in the Failure interpretation section below for how to speak each response. The tool is rate-limited to one fresh diagnosis per job per 60s, so the result is cached for follow-up reads. If the tool is unavailable, fall back to the static classification rules below.

   c. Ask the user what to do. Three paths:

      - **Accept partial result** — if `error_rows < 10%` of `total_rows`, default to this; ask to confirm. The partially-imported records stay; proceed to step 10.
      - **Export failing rows for correction** — go to step 9d.
      - **Leave it for now** — proceed to step 10; the failing rows can be fixed in a future upload.

   d. **Correction workflow** (if user chooses to fix):
      - Read the original CSV file you wrote in step 7 (still on disk as `{entity_type}_{source}.csv`). If the file is no longer on disk, tell the user to re-provide it.
      - Filter to only the rows whose `external_id` appears in `job.errors[].external_id`.
      - Check if any errors are FK-type (e.g. "customer not found") — if so, tell the user to confirm the dependency entity was imported first before fixing.
      - Write the failing rows to `{entity_type}_{source}_corrections.csv` in the same directory.
      - Tell the user: "Here are the N failing rows in `{filename}`. Fix the values, then tell me when you're ready and I'll re-upload."
      - When user confirms → upload the corrections file with the same `source` tag, through the same `get_upload_url` → PUT → `start_csv_import` flow as step 7. Previously successful rows are skipped (upsert on `external_id`); only the fixed rows are re-processed.
      - Re-enter step 8 to poll the correction job.

10. **Verify idempotency.** Call `mcp__adva-staging__get_external_ids` with the same `source` value to confirm records are linked. Re-running the import should upsert (match by `external_id`), not duplicate. `mcp__adva-staging__list_imports` gives an audit trail of prior runs — useful to confirm "did this already land?" before kicking off a re-run.

## Non-negotiable rules

These are Adva platform facts. Violating them breaks idempotency, produces bad data, or fails validation.

- **Never fabricate, invent, or fill in data values.** If a field is blank, null, or missing in the source data, pass it as blank/null. Do not generate placeholder emails, phone numbers, prices, or any other values. Your job is to map and load, not to clean or enrich.
- **Never modify source data without explicit user approval.** If you think a value looks wrong, incomplete, or needs formatting — ask the user. Do not silently fix, normalize, or transform values. Show them what you see and what you would propose, and wait for confirmation.
- **Always set `external_id` and `source` on every record.** `external_id` is the source system's primary key; `source` is a short string identifying the system (e.g. `"airtable"`, `"jobber"`, `"csv"`). Without these, every re-run creates duplicates.
- **Cross-entity references use `*_external_id` fields, never Adva UUIDs.** E.g. on a proposal, set `customer_external_id` to the source's customer ID. The import API resolves these to Adva UUIDs via an external-id index.
- **Never send computed values.** Totals, margins, balances, tax amounts, line totals — Adva computes these natively from the constituent fields. If the source has them, drop them from the mapping rather than passing them through.
- **Respect entity tiers.** A tier-4 entity (e.g. proposal) that references a tier-2 entity (customer) requires the customer to already exist in Adva. If you try to import them in the wrong order, validation fails.
- **`custom_fields` in CSV must be a JSON-encoded object string.** The CSV cell should contain the full JSON object, e.g. `{"HOA Name": "Oak Valley"}`. A plain text value will fail validation. If you want to *define* a new custom field, that's a separate step — see [references/custom-fields.md](references/custom-fields.md).
- **Never write intermediate JSON files.** Records fetched from a source belong in a CSV file on disk, not in a JSON file. Writing JSON to disk and reading it back adds latency with no benefit.

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
- **Secondary contacts.** Many sources collapse primary + secondary customer onto one record. Adva's customer entity is per-contact; a second contact is its own customer record, linked via the shared proposal or account.
- **Addresses are separate entities.** Billing / service addresses belong on a `location` record, not directly on a customer. Customer → location is 1:N.
- **Field names are exact.** Adva field names are snake_case and case-sensitive. Don't normalize spaces or change capitalization when building mappings.
- **Proposals and jobs with line items.** `line_items` and `job_items` cannot be expressed in a flat CSV. Import the parent entities first (as `proposal` / `job` entity types), then import child items as `proposal_item` / `job_item` with the parent's `external_id` as the FK reference.

### Failure interpretation

When `start_csv_import` or `get_import_status` surfaces errors, load [references/import-errors.md](references/import-errors.md) and explain each error in plain terms before asking the user to fix anything.

**Prefer the AI diagnosis for classification.** `mcp__adva-staging__diagnose_import_failure({ job_id })` (see step 9b-bis) returns `root_cause.category` — let that drive the data-vs-platform decision. The static rules below are the fallback when the tool is unavailable and the basis for the reference doc's per-error guidance.

Errors fall into two classes:
- **Data issues** (`root_cause.category == "data"` — or `"config"`) — enum mismatch, missing required field, wrong tier order, bad `custom_fields` type, duplicate `external_id` within batch. Guide the user to fix these.
- **Platform bugs** (`root_cause.category == "platform"`) — `idx_contacts_email` violation (post-ADV-749), unexpected 500 on small batch, server timeout on small batch, any error contradicting the stated model. Call `mcp__adva-staging__report_import_problem({ job_id, title, body })` using the diagnosis's `file_linear_ticket.title`/`body` (PII already stripped server-side). Continue with records that passed.

  The tool returns `{ filed, identifier, url }`. Speak each case:
    - `filed: true` (identifier + url populated) — *"Logged as `<identifier>` — we're on it."*
    - `filed: false` (identifier + url null) — the server isn't wired up to file yet. Say: *"Couldn't file automatically — Linear isn't wired up here yet. Here's what would have been filed: **<title>** — <one-line body excerpt>. Continuing with records that did succeed."*
    - tool returns `isError: true` with `error_type: "permanent"` (HTTP 403) — the caller doesn't have business-owner or super-admin rights. Say: *"I don't have permission to file from this account — flagging it for someone who does."* Do not retry.

Never show raw database error messages to the user. Translate them. Never drop failing records silently.

## What this skill does NOT do

- **Authentication setup.** The adva-superpowers plugin wires the MCP; first tool call prompts SSO if the user isn't signed in.
- **Complex transformations.** If the source needs deduping, record splitting, or multi-row aggregation, handle those steps explicitly with the user before running the onboarding workflow — they're not automated here.
- **Source discovery when no MCP server or file access is available.** If the user's source has no connector yet and isn't a local file, the first step is getting read access, not improvising an import.
