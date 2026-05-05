# Import error reference

Load this reference when `validate_records` or `upload_records` returns errors. Explain each error in plain terms to the user before asking them to fix anything. Classification drives the response: **data issues** the agent helps the user fix here; **platform bugs** the agent files automatically as Linear tickets.

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

These errors should not occur in normal operation. When encountered, file a ticket automatically (if `LINEAR_FILING_ENABLED`) and tell the user.

| Error class | Why it's a platform bug | What to include in the ticket |
|-------------|------------------------|-------------------------------|
| `duplicate key violates unique constraint idx_contacts_email` | Post-ADV-749, the pipeline uses ON CONFLICT upsert for email. This constraint should never surface to the caller. | Entity type, error verbatim, first offending record (strip PII — no email/name), expected: upsert succeeds; observed: constraint error |
| Unexpected 500 on small batch (< 10 records) | Server errors on tiny payloads indicate backend logic failure, not data size. | MCP tool name + parameters (with PII stripped), response body, batch size |
| Server timeout on small batch | Timeouts are expected on large batches; a small batch timing out suggests a query regression. | Entity type, batch size, time elapsed, any FK references in the payload |
| Any error that contradicts the stated data model | E.g. "email is required" when the import schema marks it optional. | The schema field spec, the sent payload, and the error verbatim |

## How to file a platform bug

When `LINEAR_FILING_ENABLED` is true in the session:

```
mcp__linear__createIssue({
  teamId: "<ADV team ID>",
  title: "<Short imperative title>",
  description: "<Entity type, error verbatim, first offending record with PII stripped, expected vs. observed, MCP tool + parameters>",
  labelIds: ["<onboarding label ID>"],
  priority: 2   // P2
})
```

Tell the user: *"Filed ADV-XXX — '[title]'. Ryan will see this. Let's continue with the records that did succeed."*

Do not block the rest of the import on a platform bug. Continue with the records that passed validation.

## Interpreting validate_records output

`validate_records` returns errors at the record level. For each error:

1. Identify the error class using the table above.
2. State the plain-English meaning to the user (no raw database error messages).
3. Propose the fix. If the fix is a data transformation, show an example.
4. For platform bugs, file the ticket first, then continue.
5. After all errors are explained, ask: "Would you like me to apply these fixes and re-validate, or would you prefer to review the data yourself first?"

Never silently drop erroring records without telling the user. Never invent values to make validation pass.
