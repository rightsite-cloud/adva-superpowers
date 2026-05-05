# Data model foundations

Load this reference at Step 3 (listing entity types) or whenever the agent needs to explain why the platform behaves the way it does. Deep dives live in [contacts.md](contacts.md) (identity model) and [import-errors.md](import-errors.md) (failure patterns).

## Multi-tenant hierarchy

```
Account (subscription holder, e.g. "Canopy Home Services LLC")
  └── Business (operating division, e.g. "Canopy Turf", "Canopy Coatings")
      └── Brand (customer-facing identity, e.g. "Canopy", "Polyloom")
```

Every import lands in **exactly one business**. The business is the primary scope of all data — contacts, proposals, jobs, invoices, and commissions are all tied to a `business_id`. Brands are customer-facing display identities and don't affect where data lives.

## The Unified Contact Model

`core.contacts` is a **global identity store shared across the entire account**. A contact record holds the person's email, phone, and name — once, globally.

A contact becomes a *role* in a specific business by having a `contact_role` row:

```
core.contacts           (global — one row per person)
  → core.contact_roles  (scoped — one row per role per business)
      role_type: customer | team_member | vendor | ...
      business_id: the specific business this role belongs to
```

This means:
- The same person can be a `customer` in Business A and a `team_member` in Business B. They share one contact record; each business has its own `contact_role` row.
- When you import a customer list into Business A, you are creating contact records + `customer` contact_role rows, not just customer rows.
- When you import a team-member list into Business B, you are creating (or reusing) contact records + `team_member` contact_role rows.

## Email is the global identity key

`core.contacts` has a hard `UNIQUE INDEX idx_contacts_email` on `lower(email)`.

**What this means for imports:**
- If a contact with `email = "alice@example.com"` already exists globally (from any previous import or business), attempting to insert another contact with the same email will trigger `duplicate key violates unique constraint idx_contacts_email`.
- The import pipeline handles this correctly via an **upsert** (ON CONFLICT DO UPDATE) — if the email already exists, it attaches a new `contact_role` to the existing contact rather than creating a duplicate. Post-ADV-749, this is the expected path, not an error.
- If you see `idx_contacts_email` surface as an error in production, it is a platform bug. File a Linear ticket automatically. See [import-errors.md](import-errors.md).

## Phone is a soft match signal, not a unique key

Phone numbers are normalized to E.164 format via a database trigger (`phone_normalized`). There is **no unique constraint on phone** — multiple contacts can share the same number.

`find_duplicate_contacts()` uses normalized phone (plus name similarity) to flag potential duplicates. Its output is advisory — it returns `match_type = phone` rows for human review, but the platform does not enforce uniqueness on phone.

**Why this matters:**
- An import that has two customers with the same phone number is not automatically a data error.
- Future SMS/Twilio correlation must handle "multiple contacts, same phone" explicitly — inbound texts from a shared number can't be automatically attributed to one contact.

## Entity tiers and why order matters

Entities are organized into tiers based on their dependency graph. Tier-N entities hold FK references to tier-(N-1) entities.

| Tier | Examples | FK depends on |
|------|----------|---------------|
| 1 | products, territories, brands | (none) |
| 2 | customers, team members | (none) |
| 3 | locations | customers (tier 2) |
| 4 | proposals | customers (tier 2) |
| 5 | jobs, invoices | proposals (tier 4) |
| 6 | transactions, commissions | jobs/invoices (tier 5) |

**Why this matters for imports:** A proposal row has a `customer_id` FK. If the customer contact does not yet exist in Adva for this business, importing the proposal fails with a FK violation. The fix is always: import lower tiers first.

When the user's source has data spanning multiple tiers (e.g. a single Airtable base with both customer and proposal rows), always plan the import sequence: customers first, then proposals, then jobs, etc.

## Quick-reference invariants

1. One contact record per email address, globally across the account.
2. Contact roles are the mechanism that gives a contact a function within a business.
3. Import order must respect tier dependencies — lower tiers before higher tiers.
4. Phone is identity-signal, not identity-key — duplicates are allowed.
5. All data is business-scoped — every import run targets one business.
