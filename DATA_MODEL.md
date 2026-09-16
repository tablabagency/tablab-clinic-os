# TabLab Clinic OS — V1 Data Model

**Version:** 0.4 (revised — review corrections applied)
**Status:** Draft implementation contract — not yet built
**Depends on:** ARCHITECTURE.md, DECISIONS.md (D-001–D-010)
**Supersedes:** DATA_MODEL.md v0.3

## Revision changelog (v0.3 → v0.4)

Corrections from review, scoped exactly to the four items raised:

1. **Added ordering metadata** — `last_event_occurred_at`,
   `last_event_ingested_at`, `last_event_id` on every stateful table.
   `updated_at` alone couldn't tell a replayed event that it had
   previously lost a tie-break, since it only stores the winning
   `occurred_at`, not the full tuple that decided the tie. See §0, §8.
2. **Corrected `BOOKED` semantics** — now "has an active/confirmed
   appointment" rather than "a `Bookings` record exists," which was
   simultaneously true even after cancellation. See §6.
3. **Removed `RESCHEDULED`** from the live `Bookings.status` enum — a
   reschedule is now purely an event/history transition
   (`Booking_UID_History`), never a persistent status on the live
   booking, which stays `CONFIRMED` throughout. See §4.1.
4. **Fixed changelog wording** — v0.3's changelog said "seven items
   raised" when it was actually eight; corrected below.

Nothing else changed from v0.3. Full v0.2→v0.3 changelog retained below,
now correctly reading "eight items."

## Revision changelog (v0.2 → v0.3)

Corrections from review, scoped exactly to the eight items raised:

1. **Fixed a real contradiction:** `Booking_UID_History` (and `Bookings`)
   uniqueness changed from bare `cal_uid` to `(clinic_id, cal_uid)` —
   v0.2's global uniqueness rule contradicted the stated multi-clinic
   isolation requirement. See §4.1, §4.2.
2. Added explicit contact-field **normalisation** (email trim+lowercase,
   phone → E.164) ahead of matching — see §2.
3. Added an explicit **concurrency-safety requirement** for `Contacts`
   creation — search-then-create is not assumed atomic — see §2.
4. **Resolved** `NOT_BOOKED` vs `CANCELLED` lifecycle semantics — no
   longer an open item — see §6.
5. Reworded the AI-confidence open item from "needs a number" to "needs a
   defined, tested policy, numeric or rule-based" — see §11 item 6.
6. Clarified `provider_message_id` is provider-dependent and may be
   absent — its absence is expected, not a bug — see §5.
7. Renamed `match_confidence` value `CONFIRMED` → `MATCHED` — see §2.

(The "same transaction" wording correction lives in EVENT_CONTRACT.md,
where that language originated — not duplicated here.)

Nothing else changed from v0.2. Full v0.1→v0.2 history retained below for
continuity.

## Revision changelog (v0.1 → v0.2)

This revision addresses four specific gaps raised after v0.1 review, before any
implementation begins:

- **A.** Contact identity (the person) is now modelled separately from
  conversation identity (the channel thread) — see new `Contacts` table (§2)
  and revised `Leads` table (§3).
- **B.** Explicit event-ordering semantics for `occurred_at` / `ingested_at` /
  `updated_at` are now stated as data-model rules, not left to EVENT_CONTRACT.md
  alone — see §8.
- **C.** `Follow_ups` now has an explicit crash-safe state machine
  (`SCHEDULED → SENDING → SENT / SKIPPED / SEND_UNCERTAIN`) to close the
  "message sent but state update failed" window — see §5.
- **D.** `Bookings` is redesigned around a stable internal `booking_id` with a
  separate `Booking_UID_History` lineage table, rather than assuming Cal.com
  reuses one UID forever — see §4.

Nothing in v0.1 that wasn't flagged as a gap has been silently changed.

---

## 0. Cross-cutting rules (apply to every table)

- **DECISION (per ChatGPT resolution #4):** `lifecycle_stage` on `Leads` is
  the single canonical lead state. No other field may represent "where the
  lead is" — other fields are modifiers/flags, not parallel states.
- **REQUIREMENT:** every table carries `clinic_id` (directly or via a
  linked record) — see §9, multi-clinic isolation.
- **REQUIREMENT:** every table carries audit timestamps `created_at` and
  `updated_at`. `updated_at` semantics are precisely defined in §8.
- **REQUIREMENT (new in v0.4):** every *stateful* table (`Contacts`,
  `Leads`, `Bookings`, `Follow_ups` — not `Clinics` or the append-only
  `Booking_UID_History`/`System_Errors`) also carries
  `last_event_occurred_at`, `last_event_ingested_at`, and `last_event_id`
  — the ordering metadata described fully in §8. These are omitted from
  each table's individual field list below to avoid repeating the same
  three rows four times; they are implied by "stateful table" and defined
  once in §8.
- **REQUIREMENT:** no table stores secrets, tokens, or credentials
  (per CLAUDE.md, ARCHITECTURE.md §12).

---

## 1. Table list (V1)

`Clinics`, `Contacts` (new in v0.2), `Leads`, `Bookings`, `Booking_UID_History`
(new in v0.2), `Follow_ups`, `System_Errors`. `Conversations` remains excluded
— see §10.

---

## 2. Contacts (new in v0.2 — addresses item A)

Represents the **person**, independent of any single conversation thread or
channel. A contact may generate multiple `Leads` (enquiries) over time —
e.g. messaging on Instagram in March and again on WhatsApp in June should
resolve to one contact, two leads.

| Field | Required | Notes |
|---|---|---|
| `contact_id` (PK) | Required | UUID, generated by n8n |
| `clinic_id` (FK → Clinics) | Required | Contacts are scoped per clinic, not global across clinics — see §9 |
| `first_name` | Optional | |
| `last_name` | Optional | |
| `email` | Optional | Used for matching when present. Stored/matched in **normalised** form — see normalisation rule below |
| `phone` | Optional | Used for matching when present. Stored/matched in **normalised** form — see normalisation rule below |
| `match_confidence` | Required | `MATCHED` (v0.3: renamed from `CONFIRMED` — an exact email/phone match is a matching-rule result, not identity proof; `MATCHED` avoids implying more certainty than the rule actually gives) / `UNRESOLVED` (no matchable field yet — a placeholder contact created 1:1 with a lead until better identity data arrives) |
| `created_at` / `updated_at` | Required | Audit, per §8 |

**Primary key:** `contact_id`

**Normalisation rule (new in v0.3 — REQUIREMENT, part of the n8n
validation/normalisation contract, not a new architecture decision):**
before any matching or storage, inbound contact fields are normalised:
- `email` → trimmed and lower-cased
- `phone` → normalised to a canonical format, preferably E.164 (e.g.
  `07700 900123` and `+44 7700 900123` must normalise to the same value
  before matching, or they will incorrectly resolve to two different
  contacts)

Matching in the rule below always operates on the **normalised** values,
never the raw inbound strings.

**Matching rule (RECOMMENDATION for the overall approach — identity
resolution is inherently probabilistic — but the concurrency-safety clause
below is a REQUIREMENT):**
- If an inbound lead's payload includes `email` or `phone`, normalise it
  (above), then attempt to match an existing `Contacts` record within the
  same `clinic_id` on that normalised value.
- If matched → link the new `Leads` record to the existing `contact_id`,
  `match_confidence = MATCHED`.
- If no match (or no email/phone supplied) → create a new `Contacts`
  record with `match_confidence = UNRESOLVED`, linked 1:1 to the new lead
  for now.
- **V1 does NOT attempt automatic retroactive merging** of two `UNRESOLVED`
  contacts later found to be the same person (e.g. fuzzy name matching).
  That is a real feature gap but an explicit scope decision: automatic
  fuzzy merging risks incorrectly combining two different people's data,
  which is a worse failure mode than two separate contact rows for the
  same person. Manual merge (a human-reviewed action) is the V1 answer if
  this becomes a real operational problem — **not built pre-emptively**,
  per ARCHITECTURE.md §13.

**Concurrency-safety requirement (new in v0.3 — REQUIREMENT, closes a real
gap):** search-then-create against `Contacts` must not be assumed atomic.
Two `lead.created` events carrying the same normalised email/phone can
arrive close enough together that both independently search, both find
nothing, and both create a contact — producing exactly the duplicate the
matching rule was meant to prevent. Implementation **must** include either
(a) a second lookup/collision check immediately before the create write,
treating a collision found at that point as a match rather than proceeding
to create, or (b) a serialising mechanism (e.g. a per-`(clinic_id,
normalised-email-or-phone)` lock or an upsert-style conditional write) so
concurrent or replayed `lead.created` events for the same person cannot
both succeed in creating a contact. No design may rely on search-then-create
being atomic by default.

**Unique constraint:** none enforced at the database level on `(email)` or
`(phone)` alone — enforcing uniqueness there would make the matching rule
above impossible to represent (an `UNRESOLVED` contact with no email/phone
must be allowed to coexist with others). Deduplication is a *matching
rule* applied at write time by n8n, reinforced by the concurrency-safety
requirement above — not a schema constraint.

---

## 3. Leads

Represents one enquiry/journey instance — not the person (see §2) and not
the raw conversation thread (see below).

| Field | Required | Notes |
|---|---|---|
| `lead_id` (PK) | Required | UUID, generated by n8n at first `lead.created` event |
| `clinic_id` (FK → Clinics) | Required | Isolation key — see §9 |
| `contact_id` (FK → Contacts) | Required | **New in v0.2.** Always set — either matched or a fresh `UNRESOLVED` contact (§2). This is the person-level identity. |
| `external_conversation_id` | Required | **Conversation-level identity only** (Manychat conversation/subscriber ID). Used purely for webhook-replay idempotency (§6.1 in EVENT_CONTRACT.md) — it is *not* used to determine who the person is. |
| `channel` | Required | `instagram` / `whatsapp` / other approved entry point |
| `source` | Required | e.g. paid ad, organic, referral |
| `campaign` | Optional | |
| `entry_point` | Optional | |
| `preferred_channel` | Required | |
| `treatment_interest` | Optional | Category-level only — see §10 exclusions in the original scope (unchanged from v0.1) |
| `lifecycle_stage` | Required | Canonical state, enum in §7 (unchanged from v0.1) |
| `handover_required` | Required (default false) | Independent flag |
| `follow_up_active` | Required (default true) | Independent flag |
| `ai_confidence_flag` | Optional | Set true on AI low-confidence fallback |
| `response_time_seconds` | Optional | |
| `after_hours` | Optional | |
| `assigned_to` | Optional | |
| `last_contact_at` | Optional | |
| `next_follow_up_at` | Optional | |
| `booked_value` / `treatment_revenue` | Optional | |
| `notes` | Optional | Operational only |
| `created_at` / `updated_at` | Required | Audit, per §8 |

**Primary key:** `lead_id`

**Unique constraint:** `(clinic_id, external_conversation_id, channel)` —
this is a **conversation-thread** dedupe key (prevents the same webhook
replay from creating two lead rows for the same conversation). It is
deliberately *not* a person-identity constraint — that role now belongs to
the `Contacts` matching rule in §2. This is the concrete fix for item A:
before v0.2, this one field was doing both jobs (conversation dedupe *and*
implicit person identity), which broke down the moment a contact messaged
through a second channel.

**Note on first-name/last-name/email/phone:** these have moved to
`Contacts` and are no longer duplicated on `Leads`. A `Leads` record
reaches contact detail via `contact_id`.

---

## 4. Bookings (redesigned in v0.2 — addresses item D)

**Problem with v0.1:** the original design used Cal.com's `cal_booking_uid`
directly as both the primary lookup key and the assumed-permanent identity
of a booking. This assumed Cal.com either always reuses the same UID on
reschedule, or always issues a new one — without verifying which, and with
no way to represent the relationship between the old and new UID if it
does change.

**v0.2 design:** separate the **stable internal booking** from the
**Cal.com UID(s) that have ever represented it**, and record the relationship
explicitly. This works correctly regardless of which way Cal.com actually
behaves, so it removes the hard dependency on verifying that behaviour before
building — reschedule webhook handling is no longer blocked; it just needs a
resolution rule (see EVENT_CONTRACT.md §7).

### 4.1 `Bookings`

| Field | Required | Notes |
|---|---|---|
| `booking_id` (PK) | Required | **Internal, stable for the life of the appointment relationship — survives reschedules.** UUID, generated by n8n on first `booking.created`. |
| `clinic_id` (FK → Clinics) | Required | Denormalised for isolation, §9 |
| `lead_id` (FK → Leads) | Required | |
| `current_cal_uid` | Required | The Cal.com UID that currently represents this booking *right now*. Updated on reschedule if Cal.com issues a new UID; unchanged if it doesn't. |
| `status` | Required | `CONFIRMED` / `CANCELLED` (v0.4: `RESCHEDULED` removed — see rationale below) |
| `appointment_at` | Required | Current appointment time |
| `attendance_status` | Optional | `ATTENDED` / `NO_SHOW` / unset |
| `created_at` / `updated_at` | Required | Audit, per §8 |

**Primary key:** `booking_id`

**Unique constraint (corrected in v0.3 for the same reason as
`Booking_UID_History` below):** `(clinic_id, current_cal_uid)` unique
across active bookings — not `current_cal_uid` alone, for the same
clinic-isolation reason given in §4.2.

**`RESCHEDULED` removed from `status` in v0.4 (rationale):** v0.3 listed
`RESCHEDULED` as a live status value but then had to explain that it was
really a transient marker belonging to the *old* lineage entry, and that
the live booking's actual status after a completed reschedule was
`CONFIRMED` — a sign the state didn't belong on this field. A reschedule
is represented as an **event/history transition** (a new
`Booking_UID_History` row, §4.2), not a persistent status on the live
booking. The live `Bookings.status` now only ever answers "is this
appointment currently confirmed or cancelled" — `CONFIRMED` throughout a
reschedule (only `appointment_at` and `current_cal_uid` change),
`CANCELLED` if it's cancelled.

### 4.2 `Booking_UID_History` (new table)

Append-only lineage record — every Cal.com UID that has ever been
associated with an internal `booking_id`, in order.

| Field | Required | Notes |
|---|---|---|
| `history_id` (PK) | Required | |
| `booking_id` (FK → Bookings) | Required | The stable internal booking this UID belonged to |
| `cal_uid` | Required | The Cal.com UID at this point in the lineage |
| `event_type` | Required | `CREATED` / `RESCHEDULED_FROM` / `RESCHEDULED_TO` |
| `effective_from` | Required | When this UID became the active one (business time, i.e. `occurred_at` of the triggering event) |
| `recorded_at` | Required | When n8n wrote this row — audit, per §8 |

**Primary key:** `history_id`

**Unique constraint (corrected in v0.3):** `(clinic_id, cal_uid)` unique
across the whole history table — **not** `cal_uid` alone. v0.2 specified
a bare `cal_uid` uniqueness rule, which directly contradicted §9's
requirement that every idempotency/matching key include `clinic_id`; a
global-uniqueness rule on `cal_uid` would have meant a lookup could
theoretically resolve to a booking belonging to a *different* clinic. The
corrected rule: a given Cal.com UID must resolve to exactly one internal
`booking_id` **within a given clinic**, and every lookup against this
 table must filter on `(clinic_id, cal_uid)` together, never `cal_uid` alone
(see EVENT_CONTRACT.md §7 for the resolution algorithm, also corrected).

**Why a separate table rather than a version number on `Bookings`
(RECOMMENDATION rationale):** an append-only history table preserves a full
audit trail (useful for disputes — "the clinic says the client rescheduled
twice") without overloading the live `Bookings` row with historical
clutter, and keeps the "what is the UID *right now*" lookup on `Bookings`
fast and simple.

---

## 5. Follow_ups (revised in v0.2 — addresses item C)

**Problem with v0.1:** the status model was `SCHEDULED → SENT / SKIPPED /
CANCELLED`, written by n8n only after a message-send call returned. If the
n8n process crashed *after* the message was actually sent (e.g. Manychat
API call succeeded) but *before* the `SENT` write landed in Airtable, a
naive retry would see `SCHEDULED` and send the message again — a real
double-message risk to a lead, not just a data-consistency issue.

**v0.2 design:** an explicit claim/lock step and an "uncertain" terminal
state that requires reconciliation rather than a blind retry or a blind
assumption of success.

| Field | Required | Notes |
|---|---|---|
| `follow_up_id` (PK) | Required | |
| `clinic_id` (FK → Clinics) | Required | |
| `lead_id` (FK → Leads) | Required | |
| `type` | Required | e.g. `24H_NO_REPLY`, `48H_NO_REPLY` |
| `scheduled_at` | Required | |
| `claimed_at` | Optional | **New.** Set when the dispatcher transitions status to `SENDING` — the claim timestamp |
| `sent_at` | Optional | Null until confirmed sent |
| `provider_message_id` | Optional | **New.** The channel/Manychat-side message reference returned by the send call, **when the provider's API returns one at all** — provider message IDs are provider-dependent and are not guaranteed to be available. Absence of this field is an expected, normal case, not an implementation bug — see §5.1's reconciliation rule for how the `SEND_UNCERTAIN` path handles it. |
| `status` | Required | `SCHEDULED` → `SENDING` → `SENT` / `SKIPPED` / `SEND_UNCERTAIN` (see state machine below) → (manual resolution to `SENT` or `CANCELLED`) |
| `skip_reason` | Optional | Populated when status = `SKIPPED` |
| `uncertain_reason` | Optional | **New.** Populated when status = `SEND_UNCERTAIN` (e.g. "process crashed after send call, before state write") |
| `created_at` / `updated_at` | Required | Audit, per §8 |

**Primary key:** `follow_up_id`

**Unique constraint:** `(lead_id, type, scheduled_at)` — unchanged from
v0.1.

### 5.1 Crash-safe send state machine

```
SCHEDULED
   |  (dispatcher claims the job — conditional write:
   |   only succeeds if status is currently SCHEDULED)
   v
SENDING  (claimed_at set)
   |
   |-- send call succeeds, provider_message_id captured --> SENT (sent_at set)
   |
   |-- precondition re-check fails (lead replied/handed over/booked
   |   since scheduling) --> SKIPPED (no message sent, skip_reason set)
   |
   |-- CRASH between send call returning and the SENT write landing
       --> row is left in SENDING
```

**Reconciliation rule for rows stuck in `SENDING` beyond a timeout
(RECOMMENDATION for the exact timeout value, e.g. 5 minutes — the
*existence* of this rule is a REQUIREMENT):**

1. Do **not** resend automatically.
2. If `provider_message_id` was captured before the crash, use it to query
   the channel/Manychat API to confirm whether the message was actually
   delivered. If confirmed → transition to `SENT` retroactively.
3. If `provider_message_id` was never captured (crash happened before the
   send call even returned) or the provider query is inconclusive →
   transition to `SEND_UNCERTAIN`, write a `System_Errors` row, and alert.
   A `SEND_UNCERTAIN` follow-up requires a human to confirm one way or the
   other before any further automated follow-up fires for that lead —
   this is the safe failure mode: **risking one manual check is better
   than risking a duplicate message to a lead.**

This directly closes the crash window named in item C: the system now has
a state (`SENDING`) that makes the ambiguous window visible and
un-skippable, instead of a binary `SCHEDULED`/`SENT` model where the
crash window was invisible and defaulted to an unsafe retry.

---

## 6. `lifecycle_stage` — canonical enum (unchanged from v0.1)

```
NEW
CONTACTED
QUALIFYING
QUALIFIED
NOT_QUALIFIED
BOOKING_OFFERED
BOOKED
NOT_BOOKED
ATTENDED
CANCELLED
NO_SHOW
CONVERTED
NOT_CONVERTED
```

**REQUIREMENT (unchanged):** only n8n writes this field.

**`NOT_BOOKED` / `BOOKED` / `CANCELLED` semantics — corrected in v0.4
(v0.3's definition of `BOOKED` was self-contradictory: "a `Bookings`
record exists" remains true even after that booking is cancelled, which
made `BOOKED` and `CANCELLED` simultaneously true under the old wording):**
- `NOT_BOOKED`: the lead reached the booking opportunity (`BOOKING_OFFERED`)
  but never obtained an appointment at all.
- `BOOKED`: the lead has an **active/confirmed** appointment — i.e. the
  linked `Bookings.status = CONFIRMED` (§4.1). This is a live condition,
  not "a booking record exists at some point," precisely so it stops being
  true once that booking is cancelled.
- `CANCELLED`: the lead previously had an appointment (a `Bookings` record
  reached `CONFIRMED`) that was subsequently cancelled — **this stage
  applies regardless of whether the cancellation happened before or after
  the original appointment date.** A cancelled booking is never
  reclassified back to `NOT_BOOKED`; the funnel must preserve the
  distinction between "never booked" and "booked, then cancelled," since
  this is analytically meaningful (a cancellation represents lost
  conversion after initial success, not the same failure mode as never
  booking at all).

These three are now mutually exclusive at any point in time, which they
were not under v0.3's wording.

---

## 7. Fields that must NOT be collected in V1 (unchanged from v0.1)

No clinical/medical history beyond category-level `treatment_interest`, no
photos/biometric data, no NHS/government health ID, no payment card/bank
data, no free-text clinical notes, no DOB by default. See v0.1 for full
list — not restated in full here since nothing changed; flagged here only
so this revision doesn't read as having silently dropped it.

---

## 8. Event ordering and timestamp semantics (new in v0.2 — addresses item B)

These rules apply to every table above and are the data-model-side
counterpart to EVENT_CONTRACT.md's envelope fields.

- **`occurred_at`** (carried on every inbound event, not a table field
  itself): the authoritative business time the event happened, as reported
  by the source system (Manychat, Cal.com). This is what ordering
  decisions are based on.
- **`ingested_at`** (carried on every inbound event): the time n8n
  received the event. Required on every event envelope as of this
  revision — no longer optional. Used as (a) a tiebreaker when two events
  share the same `occurred_at`, and (b) the basis for detecting stuck
  states (e.g. the `SENDING` timeout in §5.1 is measured from
  `ingested_at` of the claim, not from `occurred_at`).
- **`created_at`** (table field): set once, at record creation, from the
  triggering event's `occurred_at` — not from `ingested_at` — so the
  record reflects when the business event happened, not when it was
  processed.
- **`updated_at`** (table field): **REQUIREMENT** — set to
  `max(current updated_at, incoming event's occurred_at)` every time a
  state-changing event is applied. Remains a simple, human-readable
  "when was this last meaningfully changed" field.
- **Ordering metadata (new in v0.4 — closes a gap found in review):**
  `updated_at` alone only records the *value* of the winning `occurred_at`
  — it cannot tell a later replay which event actually won a tie, because
  it doesn't retain that event's `ingested_at` or `event_id`. Without
  that, a replayed/duplicate delivery of an event that *lost* a prior tie
  cannot be correctly recognised as still losing. **REQUIREMENT:** every
  stateful record (`Leads`, `Contacts`, `Bookings`, `Follow_ups`) carries
  three additional fields, set together as one unit whenever a
  state-changing event is applied:
  - `last_event_occurred_at` — the `occurred_at` of the last event applied
  - `last_event_ingested_at` — that same event's `ingested_at`
  - `last_event_id` — that same event's `event_id`

  The ordering/out-of-order decision (below) is made against this
  **tuple**, not against `updated_at` alone. `updated_at` continues to
  exist for human/dashboard readability, but is no longer the source of
  truth for ordering decisions.
- **Out-of-order handling (revised in v0.4):** an incoming event is applied
  to state-changing fields only if its `(occurred_at, ingested_at,
  event_id)` tuple sorts **after** the record's current
  `(last_event_occurred_at, last_event_ingested_at, last_event_id)` tuple,
  using the tie-break order below. Otherwise it is **out-of-order** — it
  is logged, not silently dropped and not silently applied, and the
  record's ordering metadata is left unchanged. This is what prevents a
  delayed/retried webhook from overwriting a newer state with stale data,
  and — the specific gap this closes — what lets a *replay* of an event
  that previously lost a tie be correctly recognised as still losing,
  even though `updated_at` alone can't distinguish that case.
- **Tie-break rule:** if two events have identical `occurred_at`, apply in
  `ingested_at` order. If both are identical (rare, but possible with
  batched source-system timestamps), apply in `event_id` lexical order as
  a last-resort deterministic rule — **this is a recommendation for the
  rare edge case, not expected to matter in practice at V1 volume.**
- **Clock-skew tolerance (RECOMMENDATION):** reject events whose
  `occurred_at` is more than a small tolerance (e.g. 5 minutes) in the
  future relative to `ingested_at`, routing them to `System_Errors`
  rather than trusting a clearly-wrong source timestamp. Exact tolerance is
  an implementation detail, not specified further here.

---

## 9. Multi-clinic isolation strategy (unchanged from v0.1)

Restated briefly — see v0.1 for full detail; nothing in items A–D changes
this section's conclusions.

- **REQUIREMENT:** every idempotency/matching key includes `clinic_id` —
  this now explicitly extends to `Contacts` matching (§2 — matching is
  scoped `within a clinic_id`, never across clinics) and to
  `Booking_UID_History` lookups (§4.2 — resolution should also confirm the
  resolved `booking_id`'s `clinic_id` matches the incoming event's
  `clinic_id` as a sanity check, not just trust the UID lookup alone).
- **OPEN DECISION (unresolved, carried forward from v0.1):** Airtable
  single-base-with-permissions vs. one-base-per-clinic. Still blocked on
  verifying Airtable's Interface permission model against your actual plan
  tier before onboarding a second real clinic.

---

## 10. Conversations — still not included in V1 (unchanged justification)

See v0.1 rationale — Manychat already owns conversation-level history
reliably; no concrete V1 requirement justifies duplicating it. The new
`Contacts`/`Leads` split (§2, §3) does not change this: `Contacts` and
`Leads` are structured operational records, not a message-log
replacement.

---

## 11. Summary of items still requiring a decision before build

Carried forward and updated from v0.2:

1. ~~Cal.com reschedule UID behaviour~~ — **de-risked, not eliminated.**
   The `Booking_UID_History` design (§4) works correctly under either
   Cal.com behaviour, so this is no longer a hard blocker on starting
   booking-workflow implementation. It should still be empirically
   verified during TEST-005/TEST-006 execution so the resolution
   algorithm (EVENT_CONTRACT.md §7) can be tuned to what Cal.com's webhook
   payload actually provides.
2. Airtable multi-clinic isolation mechanism (§9) — still open, still a
   blocker before a second real clinic.
3. Enforcement that only n8n writes `lifecycle_stage` (§6) — needs to be a
   build-time rule (e.g. Airtable field permissions restricting direct
   edits), not just documented intent.
4. Exact `SENDING`-timeout duration (§5.1) — needs a number before build.
5. Contact-matching precedence when **both** email and phone are present
   but point to two different existing `Contacts` records (a genuine edge
   case not yet addressed) — **RECOMMENDATION:** treat as `UNRESOLVED` and
   flag for manual review rather than silently picking one match, but this
   should be a deliberate decision, not an assumption baked in silently.
6. **AI confidence policy (reworded in v0.3 — was "needs a number," now
   deliberately broader):** an AI confidence policy must be defined and
   tested before production. The threshold may be numeric or rule-based
   depending on what the chosen classifier actually provides — some
   classifiers don't return a trustworthy calibrated probability, in which
   case a rule-based policy is the right answer, not a forced numeric
   threshold. The fallback behaviour (human handover on low confidence)
   is already well-defined and does not depend on this decision.

~~`NOT_BOOKED` vs `CANCELLED`~~ — **resolved in v0.3**, see §6 above; no
longer an open item.
