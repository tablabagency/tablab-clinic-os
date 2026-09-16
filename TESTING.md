# TabLab Clinic OS — V1 Test Plan

Testing uses synthetic data only until production sign-off.

## Pass criteria

A test is PASS only when the expected outcome is observed and evidence is recorded. A workflow that succeeds once is not sufficient for production readiness.

## Core tests

### TEST-001 — New lead capture

Input: synthetic enquiry enters through the configured channel.

Expected:
- unique lead_id created
- lead stored in Airtable
- source/channel captured
- timestamp captured
- no duplicate record

### TEST-002 — Instant AI response

Input: synthetic Instagram enquiry.

Expected:
- appropriate response generated
- response within agreed target
- conversation state stored
- no unsupported clinical advice

### TEST-003 — Qualification

Input: prospect provides treatment interest and answers qualification questions.

Expected:
- required fields extracted correctly
- qualification status updated
- next action selected correctly

### TEST-004 — Follow-up

Input: qualified lead does not reply.

Expected:
- follow-up occurs according to configured timing
- lead remains linked to the correct record
- no duplicate follow-up after retries
- follow-up stops when lead responds or status changes

### TEST-005 — Booking

Input: qualified lead books through Cal.com.

Expected:
- booking event received
- correct lead matched
- booking status and appointment time updated
- duplicate booking events do not create duplicate records

### TEST-006 — Post-booking state

Input: booking confirmed, cancelled, rescheduled or otherwise updated.

Expected:
- lead status updates correctly
- inappropriate follow-up is stopped or changed
- audit trail retained

### TEST-007 — Human handover

Input: lead asks for human support or triggers a configured escalation condition.

Expected:
- AI automation stops where appropriate
- clinic team can see the lead and context
- handover state is recorded

### TEST-008 — Duplicate prevention

Input: same webhook/event delivered more than once.

Expected:
- one business event is recorded
- no duplicate message, booking or lead record

### TEST-009 — Failure/retry

Input: deliberately fail a downstream dependency in staging.

Expected:
- failure is recorded
- retry occurs where appropriate
- alert/recovery path works
- lead is not silently lost

### TEST-010 — End-to-end synthetic lead

Input: complete synthetic journey from enquiry to booking and final outcome.

Expected:
- all systems remain consistent
- Airtable contains complete lifecycle data
- dashboard metrics reflect the test event correctly
- no orphan records

## Test record format

For each test record:

- Test ID
- Date/time
- Environment
- Input
- Expected result
- Actual result
- PASS / FAIL / BLOCKED
- Evidence
- Failure cause (if applicable)
- Fix/next action
- Retest result
