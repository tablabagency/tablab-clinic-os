# TabLab Clinic OS

TabLab Clinic OS is the working project for TabLab's AI-powered lead handling, booking and performance system for aesthetic clinics.

## V1 goal

Build a simple, reliable and repeatable system that moves an enquiry through:

**Enquiry → AI conversation → qualification → follow-up → booking → tracking → performance reporting**

## Founding-client offer

- £150/month
- No setup fee
- No per-lead fee
- Initial offer intended for early pilot/founding clinics
- Pricing to be reviewed only after real-world testing and evidence of value

## Intended client-facing scope

1. Instagram AI
2. WhatsApp
3. Automated follow-up
4. Booking
5. Clinic dashboard

## Initial technology direction

- Manychat: conversational layer and channel automation
- n8n: critical workflow orchestration (to be validated)
- Airtable: operational CRM/database and client-facing workspace
- Cal.com: booking integration (to be validated)
- OpenAI: AI classification/generation where required
- Make.com: legacy/prototype workflows only unless a specific non-critical use case remains

## Project principles

- Reliability before feature count
- Simple clinic onboarding
- No silent failures
- Test with synthetic data before production
- Minimise personal/health data
- Keep production credentials and secrets out of GitHub
- Every important architectural decision is documented
- Every production-critical workflow has an explicit test result

## Status

See `PROJECT_STATUS.md` for current state and `DECISIONS.md` for agreed architecture decisions.
