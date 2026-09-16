# TabLab Clinic OS — V1 Architecture

**Version:** 0.1 (working architecture)
**Date:** 16 September 2026
**Target launch:** January 2027
**Status:** Approved direction; implementation details still to be validated

## 1. Product objective

TabLab Clinic OS is a vertical lead-conversion system for aesthetic clinics. The product is not positioned as a chatbot or an automation bundle. Its job is to reliably move an enquiry through:

**Enquiry → conversation → qualification → follow-up → booking → tracking → measurable outcome**

The system must show the clinic what happened to enquiries and where opportunities are being lost.

## 2. V1 client-facing scope

The pilot product must deliver:

- AI enquiry handling
- Instagram as the primary acquisition/conversation channel
- WhatsApp as an important channel/add-on, subject to final Manychat plan and cost validation
- Automated follow-up
- Booking
- Lead pipeline / CRM
- Clinic dashboard and performance reporting

The clinic should experience a simple onboarding process. Technology names should stay mostly behind the scenes.

## 3. Commercial model

Initial founding-client offer:

- £150/month
- £0 setup fee
- £0 per-lead fee
- Initial pilot/founding price only
- Review price only after testing real clinics and measuring outcomes

Potential WhatsApp pricing will be determined after real infrastructure costs and usage are validated. Do not promise a WhatsApp price until the cost model is confirmed.

## 4. Working system architecture

```text
Instagram / Facebook / WhatsApp / other approved entry points
                         |
                         v
                    MANYCHAT
          Conversation + channel automation
                         |
                         v
                 Critical events/API
                         |
                         v
                      n8n
       Orchestration + validation + state changes
                         |
              +----------+----------+
              |                     |
              v                     v
           Airtable              Cal.com
        CRM + dashboard           Booking
              |                     |
              +----------+----------+
                         v
                      Airtable
                 Booking/outcome data
                         |
                         v
                  Clinic Interface
             KPIs + pipeline + follow-ups

OpenAI is used where AI generation/classification adds value and where native Manychat capabilities are insufficient or unsuitable.
```

## 5. Responsibility by component

### Manychat

Primary responsibility:

- Channel conversations
- Instagram messaging
- WhatsApp messaging where enabled
- AI replies where native capability is reliable and appropriate
- Basic qualification/conversation state
- Channel-native follow-up where appropriate
- Human handover controls

Do not rebuild a capability externally if Manychat already handles it reliably.

### n8n

Primary responsibility:

- Critical orchestration
- Webhook/event ingestion
- Validation and normalisation
- Idempotency / duplicate protection
- Cross-system state updates
- Booking event processing
- Follow-up state that must survive independently of a channel tool
- Error handling/retry logic
- Monitoring and alerting hooks
- Integration glue not suitable for Manychat

n8n is the current preferred candidate for the critical path, but production readiness must be proven through testing.

### Airtable

Primary responsibility:

- Central operational lead record for V1
- Lead pipeline
- Clinic/client workspace
- Follow-up queue
- Booking/outcome tracking
- Dashboard/interface
- Reporting views

Airtable should be designed as a relational operational model rather than simply copying the old Google Sheet columns.

Airtable Interfaces are suitable for dashboards and drill-down views; published interfaces can be shared with collaborators according to Airtable plan/permission rules. See Airtable's current documentation before choosing the final client-sharing model.

### Cal.com

Primary responsibility:

- Appointment booking
- Availability
- Booking confirmation
- Booking event/webhook source for the system

### OpenAI

Use only where required for:

- Generating or classifying messages
- Intent/lead classification
- Summarisation or structured extraction

Do not call an external AI model for simple deterministic logic.

### Make.com

Make is retained as a legacy/prototype reference. It is not on the production critical path unless a specific use case is reviewed and accepted.

## 6. Lead record model — V1 working draft

The exact Airtable schema will be finalised before implementation, but the lead record should be able to represent at least:

- lead_id
- clinic_id
- created_at
- updated_at
- first_name
- last_name (optional)
- email (optional)
- phone (optional)
- preferred_channel
- source
- campaign
- entry_point
- treatment_interest
- conversation_status
- qualification_status
- follow_up_status
- booking_status
- appointment_id (optional)
- appointment_at (optional)
- attendance_status (optional)
- conversion_status
- booked_value (optional)
- treatment_revenue (optional)
- assigned_to (optional)
- last_contact_at
- next_follow_up_at (optional)
- response_time_seconds (optional)
- after_hours (optional)
- handover_required (optional)
- notes (minimal and relevant)

Do not store unnecessary clinical/health information in the V1 CRM.

## 7. Lead lifecycle

```text
NEW
 ↓
CONTACTED
 ↓
QUALIFYING
 ↓
QUALIFIED / NOT_QUALIFIED
 ↓
BOOKING_OFFERED
 ↓
BOOKED / NOT_BOOKED
 ↓
ATTENDED / CANCELLED / NO_SHOW
 ↓
CONVERTED / NOT_CONVERTED
```

A separate follow-up state must prevent repeated or conflicting messages.

## 8. Follow-up principles

Follow-up must be state-aware.

Rules should include:

- never send a scheduled follow-up after a lead has replied when that follow-up is no longer appropriate
- stop automated follow-up after booking when the sequence is no longer needed
- stop automation when human handover is required
- prevent duplicate sends when an event is retried
- record every automated follow-up attempt
- make timing configurable per clinic

## 9. Dashboard requirements

The dashboard should answer business questions, not just display technical events.

Minimum V1 metrics:

- New enquiries
- Qualified leads
- Bookings
- Booking conversion rate
- Response time
- After-hours enquiries handled
- Follow-ups due
- Appointment attendance where data is available
- Conversion to treatment where data is available
- Booked value / treatment revenue where supplied
- Lead source performance
- Treatment-interest performance

The dashboard should support drill-down from KPI to lead records.

## 10. Reliability requirements

A workflow is not production-ready because it succeeds once.

Critical workflows must have:

- deterministic validation where possible
- idempotency / duplicate protection
- timeouts and sensible retries
- explicit failure states
- logs or execution history
- alerting for failures affecting leads or bookings
- a recovery procedure
- synthetic-data test coverage

There must be no silent loss of a lead.

## 11. Environment strategy

### Development

Synthetic/test data only.

### Staging

A fake clinic or dedicated test workspace. Full end-to-end flows are exercised here.

### Production

Only after V1 test suite passes and production credentials are configured by the human owner.

## 12. Security and data handling

- No secrets in GitHub.
- Do not commit client credentials, access tokens or API keys.
- Use environment variables / secret storage in production.
- Minimise collected personal and health-related information.
- Do not use real patient/lead data for development testing.
- Define retention/deletion procedures before onboarding real clinics.
- Confirm GDPR roles and data-processing arrangements before production deployment.

## 13. Deployment principle

Prefer the smallest architecture that can meet the reliability requirements.

Do not introduce additional databases, custom services or platforms unless a concrete V1 requirement justifies them.

## 14. Open decisions before implementation

1. Final Manychat channel mix for the £150 core plan.
2. WhatsApp add-on pricing after real cost validation.
3. Exact n8n hosting choice and monitoring approach.
4. Exact Airtable base/table/relationship design.
5. Exact Manychat → n8n event payloads.
6. Booking event model and idempotency key.
7. Follow-up timing/state machine.
8. Human handover conditions.
9. Exact clinic dashboard layout and permissions.
10. GDPR/data-processing workflow and retention schedule.

No implementation should lock these decisions by assumption where a decision remains open.
