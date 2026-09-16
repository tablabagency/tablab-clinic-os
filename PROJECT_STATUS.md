# Project Status

**Project:** TabLab Clinic OS
**Version:** V1 Pilot
**Target launch:** January 2027
**Last updated:** 16 September 2026
**Overall status:** 🟡 Architecture / rebuild planning

## Commercial validation

- [x] Founding-client price set at £150/month
- [x] No setup fee for pilot
- [x] No per-lead fee
- [ ] Define exact third-party software cost policy
- [ ] Recruit/test first clinic(s)
- [ ] Measure business results before increasing price

## Existing prototype audit

- [x] Make Scenario 1 reviewed: new lead instant response
- [x] Make Scenario 2 reviewed: follow-up sequence
- [x] Make Scenario 3 reviewed: email reply classifier/router
- [x] Make Scenario 4 reviewed: Cal.com webhook → Google Sheets post-booking update
- [x] Make Scenario 5 reviewed: 48-hour follow-up
- [x] Make Scenario 6 reviewed: Tally → Google Sheets lead capture
- [x] Make Scenario 7 reviewed: Twilio WhatsApp workflow
- [x] Existing Google Sheet schema reviewed
- [x] Existing Tally Glow Clinic form reviewed
- [x] Existing performance dashboard reviewed
- [x] Manychat account reconnected; current workspace/channel availability needs verification

## V1 architecture

- [ ] Finalise channel architecture
- [ ] Finalise Airtable data model
- [ ] Finalise n8n architecture
- [ ] Decide exactly what Manychat handles natively
- [ ] Define booking flow
- [ ] Define follow-up state machine
- [ ] Define dashboard/reporting metrics
- [ ] Define failure/retry/alerting strategy
- [ ] Define privacy and data-minimisation approach

## Testing

- [ ] TEST-001 New lead capture
- [ ] TEST-002 Instant AI response
- [ ] TEST-003 Qualification
- [ ] TEST-004 Follow-up
- [ ] TEST-005 Booking
- [ ] TEST-006 Post-booking update
- [ ] TEST-007 Human handover
- [ ] TEST-008 Duplicate prevention
- [ ] TEST-009 Failure/retry handling
- [ ] TEST-010 End-to-end synthetic lead

## Production readiness

- [ ] Staging environment
- [ ] Synthetic test dataset
- [ ] Monitoring/alerting
- [ ] Credential/secrets policy
- [ ] Clinic onboarding process
- [ ] Client handover documentation
- [ ] Production sign-off
