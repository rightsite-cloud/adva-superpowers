# Import error reference

Load this reference when `start_csv_import` or `get_import_status` surfaces import errors. Explain each error in plain terms to the user before asking them to fix anything. Classification drives the response: **data issues** the agent helps the user fix here; **platform bugs** the agent files automatically as Linear tickets.

> **Prefer the AI diagnosis.** Call `mcp__adva-staging__diagnose_import_failure({ job_id })` first — it returns a structured `root_cause.category` (`data` | `platform` | `config` | `transient`) with cited evidence and a `file_linear_ticket` recommendation, and is more reliable than matching error strings. Use the tables below as the fallback when the tool is unavailable, and to phrase the plain-English explanation once the category is known. When the diagnosis recommends filing, reuse its `title`/`body` (PII already stripped) rather than hand-writing the ticket.

## Error classification

### Data issues — guide the user to fix

| Error class | Plain-English meaning | Recommended fix |
|-------------|----------------------|-----------------|
| Enum mismatch (e.g. `"Business" is not a valid value for customer_type`) | The source value is not in Adva's allowed set for this field. | Use `mcp__adva-staging__get_field_schema` to get the allowed values. Build a translation map (e.g. `"Business"` → `"commercial"`). Apply to the full column before re-validating. |
| FK violation on `customer_id`, `territory_id`, or similar | A tier-4+ record references a tier-2 entity that doesn't exist yet in Adva for this business. | Import the referenced entity tier first, then retry this import. See [data-model.md](data-model.md) for tier order. |
| `custom_fields must be an object` | The `custom_fields` value was passed as a string instead of a JSON object. | Wrap the value in `{}`. Example: `{"HOA Name": "Oak Valley"}` not `"HOA Name: Oak Valley"`. |
| Required field missing | A non-nullable field is absent from the record. | Check the source columns. If the field is genuinely absent in the source, ask the user whether to skip these records or provide a default value (do not invent values). |
| `external_id` not unique within batch | Two or more records in the same upload share the same `external_id`. | Deduplicate the source before re-uploading. Show the user which rows collide. |
| Validation schema mismatch (unknown field name) | A field in the payload is not recognized by the import schema. | Drop the unrecognized field from the mapping. If the data matters, it may belong in `custom_fields` — check with `mcp__adva-staging__list_custom_fields`. |

### Platform bugs — auto-file as Linear ticket

These errors should not occur in normal operation. When encountered, call `mcp__adva-staging__report_import_problem({ job_id, title, body })` and surface the result to the user.

| Error class | Why it's a platform bug | What to include in the ticket |
|-------------|------------------------|-------------------------------|
| `duplicate key violates unique constraint idx_contacts_email` | Post-ADV-749, the pipeline uses ON CONFLICT upsert for email. This constraint should never surface to the caller. | Entity type, error verbatim, first offending record (strip PII — no email/name), expected: upsert succeeds; observed: constraint error |
| Unexpected 500 on small batch (< 10 records) | Server errors on tiny payloads indicate backend logic failure, not data size. | MCP tool name + parameters (with PII stripped), response body, batch size |
| Server timeout on small batch | Timeouts are expected on large batches; a small batch timing out suggests a query regression. | Entity type, batch size, time elapsed, any FK references in the payload |
| Any error that contradicts the stated data model | E.g. "email is required" when the import schema marks it optional. | The schema field spec, the sent payload, and the error verbatim |

## How to file a platform bug

Call the server-side MCP tool — the user needs no Linear account:

```
mcp__adva-staging__report_import_problem({
  job_id: <the import job id from step 6>,
  title: <short imperative title — use diagnose_import_failure's file_linear_ticket.title when present>,
  body:  <entity type, error verbatim, first offending record (PII already stripped server-side via diagnose_import_failure), expected vs. observed, MCP tool + parameters>
})
```

The server files under Adva's own Linear workspace (with tenant attribution) and returns `{ filed, identifier, url }`. Speak the result:

- `filed: true` — *"Logged as `<identifier>` — we're on it."*
- `filed: false` (server not yet configured to file) — *"Couldn't file automatically — Linear isn't wired up here yet. Here's what would have been filed: **<title>** — <one-line body excerpt>. Continuing with records that did succeed."*
- tool returns `isError: true` with `error_type: "permanent"` (HTTP 403, caller lacks business-owner or super-admin rights) — *"I don't have permission to file from this account — flagging it for someone who does."* Do not retry.

Do not block the rest of the import on a platform bug. Continue with the records that passed validation.

## Interpreting import errors

There is no separate dry-run validation step — validation runs server-side as part of `start_csv_import`, on the uploaded file. Errors surface in one of two shapes:

- **Whole-job rejection.** The `external_id` integrity gate runs before any write: if *any* row is missing or blank `external_id`, the entire import is rejected and the job ends `failed` with the offending rows listed in `job.errors[]`. You never get a half-written import — a file that fails this check writes nothing. The fix is always "correct the source file and re-upload"; the re-run is safe because the import is idempotent on `(source, external_id)`.
- **Row-level errors.** The job reaches `completed` with `error_rows > 0`. Bad rows (enum mismatch, FK violation, bad `custom_fields` type, etc.) are skipped; good rows land. `get_import_status` returns `next_action.action == "review_errors"` — read `job.errors[]` (hard failures) and `job.warnings[]` (automatic coercions) separately and present a grouped summary.

For each error, regardless of shape:

1. Identify the error class using the tables above.
2. State the plain-English meaning to the user (no raw database error messages).
3. Propose the fix. If the fix is a data transformation, show an example.
4. For platform bugs, file the ticket first, then continue.
5. After all errors are explained, ask: "Would you like me to apply these fixes to the CSV and re-upload, or would you prefer to review the data yourself first?"

Never silently drop erroring records without telling the user. Never invent values to make validation pass.
