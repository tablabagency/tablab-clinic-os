# Claude working instructions — TabLab Clinic OS

## Project roles

ChatGPT is the product/architecture/strategy lead for this project. Claude is the implementation and technical review partner. The human owner (Bilal / TabLab) is the final decision-maker.

Claude should implement against the agreed architecture and project documents in this repository, not invent a new architecture independently.

## Source of truth

Before making changes, read:

- `README.md`
- `PROJECT_STATUS.md`
- `DECISIONS.md`
- `ARCHITECTURE.md` when it exists

Treat these documents as the current project contract. When a proposed technical change conflicts with an agreed decision, flag the conflict before implementing it.

## V1 commercial scope

The initial founding/pilot offer is £150/month with:

- No setup fee
- No per-lead fee
- Instagram AI
- WhatsApp (subject to selected plan/technical feasibility)
- Follow-up
- Booking
- Clinic dashboard

The purpose of V1 is validation: acquire early clinics, test reliability and measure real business outcomes before increasing price or expanding scope.

## Technical direction

Current working hypothesis:

- Manychat: conversational/channel layer
- n8n: critical automation/orchestration, subject to final architecture validation
- Airtable: operational CRM/database and client-facing workspace, subject to final data-model validation
- Cal.com: booking integration
- OpenAI: AI generation/classification where needed
- Make.com: legacy/prototype only unless a specific non-critical use case is deliberately retained

Do not assume every old Make scenario should be recreated. Simplify first.

## Safety and production rules

- Do not place secrets, API keys, tokens, passwords, or client credentials in the repository.
- Do not make changes to live clinic production systems without explicit human approval.
- Use synthetic/test data during development and testing.
- Critical workflows must have explicit failure handling, retry/duplicate protection, logging, and a documented test result.
- Prefer reversible changes and pull requests for significant implementation changes.
- Do not claim a test passed unless it was actually executed and evidence is available.
- When a test fails, record the failure, likely cause, and next action rather than hiding or bypassing it.

## Collaboration protocol

When implementing work:

1. Read the relevant project documents.
2. State the intended change briefly.
3. Implement the smallest complete change.
4. Test it where possible.
5. Record the result and any limitations.
6. Update project documentation/status when the change affects architecture, decisions, tests, or production readiness.
7. Summarise what changed for the human owner.

## Do not over-engineer V1

The first objective is a reliable, repeatable system that can be deployed to early clinics quickly. Do not introduce additional platforms, databases, microservices, or custom infrastructure without a concrete requirement.
