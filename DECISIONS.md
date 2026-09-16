# Architecture Decisions

This file records decisions agreed during the TabLab Clinic OS rebuild. Decisions should not be changed silently; material changes should be documented with the reason and date.

## D-001 — Pilot pricing

**Date:** 16 September 2026

For the initial validation phase, the clinic offer is **£150/month**, with **no setup fee and no per-lead fee**. This is a founding/pilot price and is not the permanent price. Pricing will be reconsidered after real-world testing and measurement.

## D-002 — Client-facing scope

**Date:** 16 September 2026

V1 should deliver five clearly understandable outcomes:

1. Instagram AI
2. WhatsApp
3. Automated follow-up
4. Booking
5. Dashboard/reporting

The technology should remain hidden behind a simple clinic onboarding experience.

## D-003 — Reporting as a core differentiator

**Date:** 16 September 2026

Reporting and measurable business outcomes are a core part of TabLab's positioning. The system should track the journey from enquiry through qualification and booking so the clinic can understand performance rather than simply receiving an AI chatbot.

## D-004 — Airtable direction

**Date:** 16 September 2026

Airtable is the preferred direction for the operational CRM/database and client-facing workspace, subject to final architecture and cost validation.

## D-005 — Make.com

**Date:** 16 September 2026

Make.com is treated as the existing prototype/legacy automation layer. Critical production workflows should not depend on Make unless a specific use case is evaluated and accepted. n8n is the preferred candidate for critical orchestration, subject to testing for reliability, recovery, monitoring and maintainability.

## D-006 — Manychat

**Date:** 16 September 2026

Manychat is the preferred conversational/channel layer for Instagram and potentially WhatsApp. We will use native Manychat capabilities where they are reliable and appropriate, rather than rebuilding functionality unnecessarily in external automation.

## D-007 — Test before production

**Date:** 16 September 2026

No production-critical workflow is considered ready because it works once. It must pass explicit synthetic-data tests covering normal flow, duplicates, failures, retries and human handover.
