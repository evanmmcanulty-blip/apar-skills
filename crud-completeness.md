---
name: crud-completeness
description: >
  Audits data management interfaces for completeness gaps — missing operations,
  inaccessible fields, broken feedback loops, and orphaned records. Distinct from
  empathize (friction), wayfinding (navigation), and fortify (security). Trigger on:
  "crud-completeness", "#crud", "what are we missing in the admin", "what operations
  are missing", "can users see their own submissions", "is the admin complete",
  "what can't we do yet", any signal the user wants to audit whether a data management
  interface covers all necessary operations for all roles that touch each entity.
  Do NOT run on static content pages, read-only dashboards with no user-submitted
  data, or marketing/informational pages — this skill is for interfaces where records
  are created, managed, and resolved.
---

# crud-completeness — Data Management Completeness Audit

The experience skills audit what's there. This skill audits what's *missing*.

For every entity a system manages, every role that touches it must be able to do every
operation their job requires. The absence of a button causes no friction — it causes
silent failure. A support team member who discovers they can't delete a test record,
an employee who submits something and never learns its outcome, an admin who has no
way to remove a departed user: none of these produce error messages. They produce
workarounds, support tickets, and eroded trust in the tool.

This audit finds those gaps before they become complaints.

---

## WHEN TO RUN — AND WHEN NOT TO

**Run when:**
- Any admin queue, dashboard, management panel, or CRUD interface has just been built or modified
- A new entity (product, request, order, user, record) has been added to the system
- A new role has been introduced that touches existing data
- Explicitly triggered: `#crud`, `crud-completeness`, "what operations are missing", "is the admin complete"

**Do not run on:**
- Static content pages, landing pages, or marketing pages — no records to manage
- Read-only dashboards that display but don't manage data
- Configuration pages with a single fixed form — scope is too narrow for this framework

When a passive trigger fires (new entity, new role, new admin feature), run the audit
before declaring the feature done. Completeness is invisible until it breaks.

---

## THE CORE MODEL

Every entity has a lifecycle. Every role that touches it needs a specific subset of
operations at each stage. A complete interface is one where no role is blocked from
an operation their job requires — and where every submitter can see what happened to
what they submitted.

### Operations

| Operation | What it means | Common gap |
|---|---|---|
| **Create** | Authorized roles can add new records with all required fields | Form exists but required fields absent from UI; UI-required fields don't match API-required fields |
| **Read (list)** | Authorized roles can find and browse records | Pagination hides records beyond page 1; sticky filters silently exclude records; empty state indistinguishable from "no records exist" |
| **Read (detail)** | Authorized roles can see all fields they need on a specific record | Fields exist in the data model but aren't surfaced in the UI; list loads, detail view 404s |
| **Update** | Authorized roles can edit what their job requires | Edit path exists but some fields are read-only or absent; description editable in add form but not in the table |
| **Delete / close / resolve** | Authorized roles can remove or resolve records that shouldn't persist | No delete, no archive, no cancel — records accumulate forever with no resolution path |
| **Status override** | Admins can force-advance a record stuck in any state | Normal status flows assume all actors complete their step; stuck records have no escape |

### Additional completeness dimensions (not CRUD but consistently missing)

| Dimension | What it means | Common gap |
|---|---|---|
| **Bulk action** | High-volume operations don't require one-by-one processing | Delete ✓ per row; deleting 200 records takes 200 clicks |
| **Export / download** | Admins can extract records for payroll, compliance, vendor handoff, reporting | Not in the CRUD model, so never checked; surfaces in week 1 when ops asks for "a report" |
| **Audit trail** | For high-trust operations (role changes, payroll, inventory), who did what and when | Feels like nice-to-have until the first "I didn't do that" dispute |
| **Offboarding / deactivation** | Departed users can be removed or disabled | Treated as HR process, never shows up in CRUD audit; former employees retain access indefinitely |

### Feedback loop (submitter → reviewer entities)

| Checkpoint | What it means |
|---|---|
| Submission confirmation | Submitter knows their record was received |
| Past submissions visible | Submitter can see all their prior records without asking a human |
| Current status visible | Submitter can see current status without polling a colleague |
| Status notification | Submitter is notified when status changes — not just "the page exists if they check" |
| Rejection reason visible | If rejected, submitter sees why |
| Resolution confirmation | Submitter learns when their request is fully resolved |

A feedback loop is broken at its weakest link. A status page with no notification is
80% of a feedback loop — submitters wait for an email that never comes. Shipping the
status page and calling the loop complete is the single most common feedback loop mistake.

---

## PHASE 1 — SCOPE AND ENTITY DISCOVERY

Define scope before auditing. Specify:
- **System / feature area** — what are we auditing? The whole portal, or a specific section?
- **Roles in scope** — which roles touch this area?
- **Out of scope** — what are we explicitly not auditing this pass?

Then enumerate every entity. **An entity is any distinct thing that gets created,
stored, and managed — not a UI component or a page.**

**How to discover entities — read these sources, in order:**

1. **API routes** — every route that handles POST (create), GET (read), PATCH/PUT (update), DELETE is an entity endpoint. List them.
2. **Data model / type definitions** — TypeScript interfaces, Firestore collections, database schemas. Every top-level collection or table is an entity candidate.
3. **Navigation** — every admin section, tab, or queue corresponds to an entity.
4. **Forms** — every form that submits data creates or updates an entity.

Do not rely on UI observation alone. The UI may not expose all fields the data model
contains. Always cross-reference the schema.

For each entity, record:
- **Name** and **storage location** (Firestore collection, table, etc.)
- **All fields** — from the schema, not the UI
- **Roles that touch it** — who creates, reads, updates, or deletes it
- **Lifecycle** — the states it moves through
- **Submitter→reviewer relationship** — does a lower-privilege role submit something a higher-privilege role acts on?

---

## PHASE 2 — CRUD MATRIX

For each entity, build the matrix. **Read the schema before filling Read cells —
"can they see all fields?" requires knowing what all the fields are.**

```
Entity: [name] — stored in: [collection/table]
Schema fields: [complete list from data model, not from UI]
Lifecycle: [state1 → state2 → ...]

| Role      | Create | Read: list | Read: detail | Update | Delete/close | Status override | Notes |
|-----------|--------|------------|--------------|--------|--------------|-----------------|-------|
| [role A]  |        |            |              |        |              |                 |       |
| [role B]  |        |            |              |        |              |                 |       |
```

**Cell values:**
- `✓` — fully supported
- `partial` — supported but incomplete (specify what's missing in Notes)
- `✗` — missing; role needs this and has no path to do it
- `—` — intentionally absent; role is not expected to have this operation (note the reason)
- `?` — unclear; needs verification

**Critical distinction: `✗` vs `—`**
A `✗` is a gap. A `—` is a deliberate design decision. If you're unsure which it is,
mark `?` and resolve it with the developer before reporting it as a gap. Failing to
make this distinction causes alarm fatigue and makes the audit less trusted.

**Per-cell checks:**

*Create:* Does the UI form include all fields marked required in the schema? Are any
fields available through the API but absent from the add form?

*Read (list):* Does the list load beyond the first page? Is any filter sticky in a
way that could hide records? Does an empty list clearly indicate "no records" vs
"filtered to zero"?

*Read (detail):* Does clicking a record in the list open a complete detail view? Do
all schema fields appear in the detail view or expandable row? Test with at least one
actual record — don't assume from the list that the detail works.

*Update:* Are all fields that appear in the add form also editable on existing records?
Are any fields in the schema absent from the edit UI entirely (the Colton pattern: description
visible in add form, invisible in the table)?

*Delete/close:* Is there a path to remove or resolve records? For records that should
persist for audit purposes, is there an archive or close path that removes them from
active queues without hard-deleting them? If hard-delete exists, does it require
confirmation? Does it cascade correctly to related records?

*Status override:* Can an admin force a record to any status, regardless of normal
flow? What happens to a record that gets stuck — reviewer leaves, approver is
unavailable, state machine has no exit?

**Additional dimensions — check for each entity:**

| Dimension | Check |
|---|---|
| Bulk action | If an admin realistically needs to act on >10 records at once, is there a bulk select + bulk action path? |
| Export | Is there a download/export path for this entity's records? If ops teams need this data outside the tool, is there a way to get it? |
| Audit trail | For payroll, role changes, inventory adjustments, or any operation with financial or trust implications: is there a change log? |
| Offboarding | For the User entity specifically: can a user be deactivated or removed? What happens to their records when they leave? |

---

## PHASE 3 — FEEDBACK LOOP AUDIT

For every entity where a lower-privilege role submits something that a higher-privilege
role acts on, audit the full loop:

```
Entity: [name]
Submitter role: [role] — Reviewer role: [role]

| Checkpoint                                    | Present? | How it works | Gap |
|-----------------------------------------------|----------|--------------|-----|
| Submission confirmation (immediate)           |          |              |     |
| Past submissions listed for submitter         |          |              |     |
| Current status visible to submitter           |          |              |     |
| Submitter notified when status changes        |          |              |     |
| Rejection reason visible to submitter         |          |              |     |
| Resolution confirmation                       |          |              |     |
```

**Notification vs polling:** A status page is not a notification. If the submitter
must remember to go back and check, the loop is incomplete. Note the distinction:
- ✓ notification: system tells the submitter something changed (email, in-app banner)
- ~ polling: status is visible if the submitter goes to look
- ✗ absent: no status visibility at all

Mark the "How it works" column with: `notification` / `polling` / `absent`.
A `polling` checkmark is partial credit — flag it as a gap if the submitter's
context makes polling unreliable (mobile workers, high volume, infrequent portal use).

---

## PHASE 4 — GAP REGISTER

Aggregate every `✗`, `partial`, and broken feedback loop checkpoint into a prioritized
gap register. Include the `?` cells as "needs verification" entries.

```
GAP-01
Entity: [name]
Role: [who is blocked, or "all submitters"]
Operation: [Create / Read-list / Read-detail / Update / Delete / Status override /
            Bulk / Export / Audit trail / Offboarding / Feedback loop: [checkpoint]]
Specific gap: [exactly what is missing — one sentence, specific enough to fix]
Impact: [what goes wrong — workaround required, data accumulates, submitter has no visibility]
Severity: Critical / High / Medium / Low
Fix: [what to build — one sentence, implementation-ready: name the endpoint, the UI element, the behavior]
Effort: [hours / day / days]
```

**Severity definitions:**

- **Critical** — a role cannot perform an operation their job requires with no workaround; data is accumulating with no resolution path; a departed user retains system access
- **High** — a role must go outside the tool to complete their job; submitters have no status visibility; records can get permanently stuck; high-trust operations have no audit trail
- **Medium** — a workaround exists but is slow or error-prone; one-by-one where bulk is needed; no export when ops regularly needs one; polling where notification is expected
- **Low** — affects edge cases or rarely-needed operations; polish-level completeness

---

## PHASE 5 — PRIORITIZED FIX LIST

Order: Critical first, then High, then Medium and Low. Within the same severity,
cheaper fixes first.

**Every fix must be implementation-ready.** The test: a developer who wasn't in the
room reads the fix and builds the right thing without a follow-up conversation.

Weak: "Add a delete button."
Strong: "Add a DELETE /api/portal/admin/requests endpoint (check that the Firestore
document exists before deleting; return 404 if not) and a 'Delete' button in the
Actions column of the admin requests table, gated behind a native `confirm()` dialog.
Removes the document from `portal_custom_requests`. Effort: ~2 hours."

---

## ADVERSARIAL PRESSURE

After the gap register is complete, apply three pressure tests before finalizing:

**The ops team member on day 30:** "I've been using this for a month. What did I
discover was missing that the audit said was fine?" — probe for gaps the matrix
marked ✓ that real use would expose (pagination, sticky filters, empty states
that hide filtered-out records).

**The developer who built it:** "Every ✗ I flagged, they have a reason it's fine.
What would I say back?" — force every gap through a steelman. If the steelman
is convincing, downgrade severity or reclassify as `—` (intentional).

**The person who leaves the company:** "What access do I still have? What records
am I still the owner of? What happens to my open requests?" — specifically targets
offboarding and ownership transfer gaps, which are almost never caught otherwise.

If any pressure test reveals a genuine gap not in the register, add it as a GAP entry.

---

## OUTPUT FORMAT

```
CRUD COMPLETENESS AUDIT — [system / feature area]
Scope: [what was audited — roles, entities, date]
Out of scope: [what was explicitly excluded]

ENTITY INVENTORY
[list: name, storage location, roles, lifecycle, submitter→reviewer flag]

CRUD MATRICES
[one matrix per entity, with schema fields listed above each matrix]

ADDITIONAL DIMENSIONS
[bulk / export / audit trail / offboarding — one row per entity]

FEEDBACK LOOP AUDIT
[one table per submitter→reviewer entity, with notification vs polling noted]

GAP REGISTER
[GAP-01 through GAP-N, ordered by severity]

FIX LIST
[ordered by severity × effort, each fix implementation-ready]

ADVERSARIAL PRESSURE
[three voices — what changed, if anything]

VERDICT
COMPLETE — all operations verified for all roles and all entities in scope
MINOR GAPS — N low/medium gaps; safe to ship, fix in next cycle
SIGNIFICANT GAPS — N high gaps; fix before telling users the feature is done
BLOCKED — N critical gaps; the interface cannot be considered functional for [role]
```

---

## RULES

**Read the schema before filling any Read cells.** "Can they see all fields?" requires
knowing what all the fields are. UI observation alone is not sufficient. Always
cross-reference the data model, TypeScript types, or API response.

**Distinguish `✗` from `—` explicitly.** Every `✗` needs to be confirmed as a gap,
not a design decision, before being reported as one. Unverified gaps undermine the
audit's credibility.

**Test the detail view, not just the list.** A list that loads 50 records and a
detail view that 404s on all of them is a Read gap. Always test clicking through.

**Empty states are findings.** An empty list that looks identical whether no records
exist or a filter is hiding 200 records is a Read gap. Always probe what an empty
state actually means.

**The feedback loop is not complete when the status page exists.** It is complete
when the submitter receives a notification without having to remember to go look.
Mark the distinction explicitly.

**Soft deletes require read-side filters.** If delete is implemented as a flag
(`deleted: true`, `active: false`), verify that every list view and admin queue
filters out deleted records. This is where soft delete consistently breaks.

**Offboarding is a completeness gap, not an HR process.** The User entity audit
must include: can a user be deactivated, can their access be revoked, and what
happens to records they own.

**Don't let the seven conditions become a checkbox.** The conditions below are
a diagnostic frame, not a list to audit back. For each condition, probe the system
actively — don't accept "yes" without verifying specifically how.

**A complete interface satisfies:**
1. Every role that needs to create records has a form with all fields the API requires
2. Every role that needs to read records can see all schema fields their job requires — in both list and detail views
3. Every role that needs to update records can edit all fields their job requires, on existing records (not just in the add form)
4. Every role that needs to remove or resolve records has a path to do so, without records accumulating in active queues indefinitely
5. High-volume operations have bulk paths; high-trust operations have audit trails
6. Every submitter can see their own past submissions and their current status
7. Every submitter is notified — not just able to check — when their submission is acted on
