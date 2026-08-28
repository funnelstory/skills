# Linking conversations to accounts

Read this **before** any query that pulls support conversations for a named account.
Consumed by `account-support-history/`, and applicable to `account-brief/`,
`meeting-prep/`, `churn-risk-report/`, `qbr-internal/`, and `feature-requests/`.

## Why this exists

There is no workspace-independent way to get "the tickets for account X". Across
observed deployments, **four** things vary independently:

| Varies | Observed values |
|---|---|
| Which table holds support | `tickets`, `chat_threads` (with `tickets` empty) |
| How rows link to an account | `conversation_accounts`, tag convention, channel-name convention, contact email domain |
| Account granularity | one per company, or one per *site* (8+ accounts sharing a domain) |
| Noise profile | negligible, or ~98% machine-generated alerts, or one debug channel = 31% of volume |

A hardcoded query has never survived contact with a second workspace. Do not write one.
Run the routine below instead. It costs ~8–12 queries on first use in a workspace.

**Skip the routine** if `file://workspace/profile.json` → `instructions` already records
the answers (see "Recording findings" at the end). Read the profile first, always.

---

## Step 1 — Find where support actually lives

Never assume `tickets` is populated. Confirm which tables exist, then which have rows.

```sql
SELECT name FROM sqlite_master WHERE type='table' ORDER BY name;
```

```sql
SELECT
  (SELECT COUNT(*) FROM tickets)         AS tickets,
  (SELECT COUNT(*) FROM ticket_comments) AS ticket_comments,
  (SELECT COUNT(*) FROM chat_threads)    AS chat_threads,
  (SELECT COUNT(*) FROM chat_messages)   AS chat_messages,
  (SELECT COUNT(*) FROM notes)           AS notes,
  (SELECT COUNT(*) FROM meetings)        AS meetings;
```

If `tickets` returns 0, support lives elsewhere — `chat_threads` is the common
alternative for Slack-Connect and DevRev-style deployments. Reporting
"no tickets found" off an empty `tickets` table is a **confidently wrong answer**
and the single worst failure mode of this skill.

Get real column names before writing anything; they differ between the two tables
(notably, `chat_threads` has no `status`, `priority`, `assignee_email`, or `resolved_at`):

```sql
SELECT name FROM pragma_table_info('tickets');
SELECT name FROM pragma_table_info('chat_threads');
```

## Step 2 — Split by source

Linkage is a property of the **connector**, not the workspace. A workspace that has
migrated ticketing systems will have one source that links cleanly and one that does
not. Always group by `source`.

```sql
SELECT source, COUNT(*) AS n, MIN(timestamp) AS oldest, MAX(timestamp) AS newest
FROM <table> GROUP BY source ORDER BY n DESC;
```

If two sources have **overlapping date ranges**, a migration copied history and the
same issue may appear twice. Check for duplicates before any query spanning the
overlap, and say so in the output if you cannot rule it out.

## Step 3 — Measure each linkage mechanism, then pick

Test in this order and adopt the **highest-coverage** mechanism *per source*. This is a
measurement, not a judgement call — do not ask the user which to use.

### M1 — `conversation_accounts` join (preferred)

```sql
SELECT t.source,
       COUNT(*) AS total,
       SUM(CASE WHEN ca.account_id IS NOT NULL THEN 1 ELSE 0 END) AS linked,
       ROUND(100.0 * SUM(CASE WHEN ca.account_id IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct
FROM <table> t
LEFT JOIN conversation_accounts ca ON ca.conversation_id = t.id
GROUP BY t.source;
```

Observed: 99.9–100% for Salesforce Service Cloud and DevRev; **0% for every Zendesk
and Slack source seen so far**. If a source returns 0%, that connector's model has no
account association mapped — fall through, and consider flagging it as a data-model
gap worth fixing at source (see "Escalating a mapping gap").

### M2 — Tag convention

Helpdesks often carry the customer as an organization tag.

```sql
SELECT j.value AS tag, COUNT(*) AS n
FROM <table> t, json_each(t.tags) j
WHERE t.source = '<source>' AND j.value LIKE 'org\_%' ESCAPE '\'
GROUP BY j.value ORDER BY n DESC LIMIT 40;
```

Inspect the result before trusting it. Process tags and the vendor's own org will
sit in the same namespace and must be excluded by name — e.g. a `org_notes_added`
workflow tag, and an `org_<vendor-name>` internal org, together accounted for 68% of
tagged rows in one workspace. Coverage is what remains after exclusions.

Map tag → account by normalising: strip the prefix, convert `_` and `-` to spaces,
collapse repeats, then fuzzy-match against `accounts.name`.

### M3 — Channel-name convention

For Slack-based support, the channel encodes the customer.

```sql
SELECT channel_name, COUNT(*) AS n, MAX(timestamp) AS latest
FROM chat_threads WHERE source = 'slack'
GROUP BY channel_name ORDER BY n DESC LIMIT 40;
```

Look for prefix families (`ext-customer-<slug>`, `internal-customer-<slug>`,
`internal-<slug>`). **Preserve the prefix as a `visibility` attribute** — external
channels contain the customer's own voice, internal channels contain colleagues
talking *about* them. The same customer routinely appears in both. Never merge them,
and never let internal channels reach a customer-facing deliverable.

### M4 — Contact email domain (last resort)

Only when M1–M3 all fail, and only with **suffix anchoring**:

```sql
WHERE t.contact_email LIKE '%@' || :domain
   OR t.contact_email LIKE '%.' || :domain
```

This matches `@acme.com`, `@external.acme.com`, `@contractors.acme.com` and correctly
excludes `dana.acme@othercorp.com`. A bare `LIKE '%acme%'` will not — it is how this
routine originally produced false positives.

Two conditions **disqualify** M4 entirely; check both before using it:

- **Vendor-dominated contacts.** If the top contact domain is the workspace's own
  domain, the field holds the internal filer, not the customer. Observed at 75% in one
  workspace, which would have mis-attributed three quarters of history to the vendor.
  ```sql
  SELECT SUBSTR(contact_email, INSTR(contact_email,'@')+1) AS domain, COUNT(*) AS n
  FROM <table> WHERE source = '<source>' GROUP BY domain ORDER BY n DESC LIMIT 15;
  ```
- **Shared domains across accounts.** If one domain maps to several accounts, M4
  collapses distinct sites into one bucket and destroys the distinction the user cares
  about. Observed: a single 3PL domain spanning nine site-level accounts.
  ```sql
  SELECT domain, COUNT(*) AS accounts FROM accounts
  GROUP BY domain HAVING accounts > 1 ORDER BY accounts DESC LIMIT 20;
  ```

### M5 — Title prefix (rarely viable)

Some teams prefix titles (`Customer | URGENT | ...`). Measure coverage before relying
on it; observed at 5.7%, with prefixes mixing customer names, site names, alert types
and internal categories, plus heavy aliasing. Treat as a hint for manual review, not a
mechanism.

## Step 4 — Separate signal from machine noise

Support tables frequently contain far more monitoring output than human conversation.
Detect and default to excluding it, then report how much you excluded and offer it.

Signals of machine-generated rows:

- `contact_email` null across the whole source
- Priority uses a severity ladder (`Severity 1..4`) rather than low/normal/high
- Titles carry hostnames, metrics or thresholds (`... CPU limit utilisation`, `... on host=...`)
- Near-identical titles recurring on a fixed cadence, differing only by a number
- One channel or filer dominating volume (a debug or alerting channel)

```sql
SELECT priority, status, COUNT(*) AS n,
       SUM(CASE WHEN contact_email IS NULL OR contact_email = '' THEN 1 ELSE 0 END) AS no_contact
FROM <table> WHERE source = '<source>'
GROUP BY priority, status ORDER BY n DESC LIMIT 20;
```

Machine alerts are not worthless — **alert mix is a real health signal**, and more
discriminating than volume. In one workspace an account with a seventh of the alert
volume of the noisiest account had 71% at Severity 1 against that account's 3%.
Surface this as an aggregate, never as a list of individual alerts.

## Step 5 — Report provenance, always

Every response built on this routine states, per source: the table used, the mechanism,
the coverage percentage, and what was excluded and why. A 33%-coverage answer presented
as complete is misleading even when every row in it is correct.

## Performance notes

- **Never `SELECT *`** on `tickets` or `conversations`. `custom_fields` can carry dozens
  of per-row fields (one workspace stores warehouse checklist booleans there). Name columns.
- Correlated subqueries and `COUNT(DISTINCT ...)` joins over 100k+ rows time out. Split
  into one query per source.
- `json_each` needs a real JSON array column. For text-array fields use `INSTR(col, value) > 0`.

## Extending this reference

When a workspace reveals a mechanism not listed here, add it as `M6`, `M7`… with its
coverage query and its disqualifying conditions. Keep entries **vendor-generic**: describe
the *shape* of the convention, not the customer that uses it. No customer names,
helpdesk subdomains, employee emails, or internal channel names belong in this file.

## Escalating a mapping gap

A source at 0% on M1 usually means the connector's model is missing its account
association, not that the data is unlinkable. Worth surfacing to the workspace owner —
fixing it at source is better than every skill compensating with a fuzzier mechanism.

## Recording findings

Discovery is repeatable but not free. Once run, offer the user a block they can paste
into their workspace profile `instructions` so later runs skip Steps 1–4. Present it as
text for them to apply; **this skill must never write to workspace-level configuration**,
which is an admin action with side effects beyond the current user.

Re-run discovery and compare when the recorded coverage numbers no longer match what a
spot-check returns — that is the signal a workspace has migrated systems.
