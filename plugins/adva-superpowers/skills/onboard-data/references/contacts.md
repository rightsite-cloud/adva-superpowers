# Contacts deep dive

Load this reference when the agent hits a contact-related error, the user asks about deduplication, or the behavior of the email uniqueness constraint needs explanation. Overview is in [data-model.md](data-model.md).

## The contact → role → business chain

```
core.contacts (global)
  id, email, phone, first_name, last_name, ...
  UNIQUE INDEX idx_contacts_email ON lower(email)

core.contact_roles (scoped)
  contact_id → core.contacts.id
  business_id (the business this role belongs to)
  role_type: 'customer' | 'team_member' | 'trade_partner' | ...
  + role-specific attribute table (see below)
```

Role-type specific data lives in attribute tables that reference `contact_roles`:

- `sales.customers` — customer-role attributes (billing address, customer tier, etc.)
- `core.team_members` — team-member-role attributes (hire date, role, pay type, etc.)

An import that creates a "customer" actually writes to three places: `core.contacts`, `core.contact_roles` (role_type = customer), and `sales.customers`.

## Email uniqueness in detail

**Hard constraint**: `UNIQUE INDEX idx_contacts_email ON lower(email)` on `core.contacts`.

**What the pipeline does when email already exists:**

1. The import pipeline calls an upsert with ON CONFLICT on `lower(email)`.
2. ON CONFLICT: the existing `core.contacts` row is returned as-is (or updated if the incoming record has a newer name/phone).
3. A new `core.contact_roles` row is inserted for the target business + role_type.
4. The role-specific attribute row is inserted.

This is the correct behavior — "already exists globally" means the contact is a real person the platform has already seen (perhaps from another business in the same account). The platform attaches a new role rather than creating a duplicate person.

**If `idx_contacts_email` surfaces as a raw error in validate or upload results:** this should not happen after ADV-749. If it does, it is a platform bug. Do not ask the user to rename the email. Auto-file a Linear ticket via `mcp__linear__createIssue` with:
- team: ADV
- label: onboarding
- priority: P2
- entity type, error verbatim, first offending record (strip PII — no email/name in ticket body), expected vs. observed

## Phone matching

**No unique constraint on phone.** Multiple contacts can share the same normalized phone number.

`find_duplicate_contacts()` is an advisory function that returns potential duplicates based on:
- Exact match on `phone_normalized` (E.164)
- Name similarity (fuzzy)

Use `mcp__adva-staging__get_dedup_candidates` to surface these candidates before or after import. Use `mcp__adva-staging__resolve_dedup_candidates` to merge them.

**When NOT to automatically merge:** if two contacts share a phone but have different names and different emails, they may legitimately be different people. Show the candidates to the user before merging.

## Customer vs. team_member at the data level

Same `core.contacts` row. Same `core.contact_roles` table. Different `role_type`.

A team member who is also a customer in the same business has **two** `contact_roles` rows — one with `role_type = team_member` and one with `role_type = customer`.

**Common confusion: "why does my team member show up under a different business?"**

The contact record is global. If "Alice Smith" was a team member at Business A and later imports as a customer at Business B, she has two contact_roles rows — one per business. The contact appears in both businesses' data. This is expected behavior, not a bug.

## When to use deduplication tools

| Situation | Tool |
|-----------|------|
| Before import — check if source contacts already exist | `get_dedup_candidates` with email list |
| After import — surface phone/name matches | `get_dedup_candidates` on the just-imported batch |
| User wants to consolidate two records | `resolve_dedup_candidates` |
| Debugging `idx_contacts_email` error | File as platform bug (post-ADV-749 this should not occur) |

## Gotchas

- **Do not deduplicate by phone alone.** The platform doesn't — don't you either. Phone is a signal, not a key.
- **Importing the same customer into two businesses is correct.** They'll share a contact record; each business gets its own contact_role.
- **"Secondary contact" on a source record.** If the source collapses primary + secondary contacts onto one row, split them into two separate contact records before uploading. Each contact gets its own email. A "secondary contact" is not a sub-field — it is its own customer record linked via the shared proposal (not "deal" — the `/core/deals` redirect was removed in ADV-752).
- **Name-only dedup is fragile.** "John Smith" is not a unique key. Always prefer email-based matching when available.
