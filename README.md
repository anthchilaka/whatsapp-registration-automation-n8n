# Member Registration Automation | n8n, WhatsApp Cloud API & Postgres | Nonprofit / Community Operations

## Executive Summary

Legion of Mary's CIC Comitium registers members by hand: paper forms and ad hoc messages, no structured record, no consistent process across the languages members actually speak (English, Igbo, Yoruba, Hausa). This project replaces that with a WhatsApp-native bot: members register entirely inside a chat they already use, in their own language, ending with a verified record (including a photo) written straight to a shared database. **Status: built and internally tested; not yet piloted with real members.** See Next Steps.

## Business Problem

A volunteer-run ministry organization needs member records (name, office held, date of birth, a photo) kept consistently enough to support real operations (birthday recognition, contact lists, office rosters) without asking anyone to learn a new tool or fill out a form on a computer they may not have. WhatsApp is the one channel every member already has.

## Methodology

Built a WhatsApp-native conversation as an n8n workflow, backed by a Postgres state machine, not an LLM's own memory, so the same input reliably produces the same result. Each answer is validated by explicit rules first; only a genuinely ambiguous reply escalates to a single shared LLM classification step, kept narrow and auditable. Registration ends with an in-chat photo upload, verified against the exact inbound message, written to Airtable.

## Skills Demonstrated

- **n8n / Workflow Automation:** multi-branch Switch/IF routing across 9 conversation states, a Postgres-backed deterministic state machine (replacing an earlier LLM-memory design), a shared "route after update" pattern that lets a mid-conversation correction return to the right place without duplicating logic
- **API Integration:** Meta WhatsApp Cloud API (native trigger, in-chat media download via its two-step media endpoint), Airtable REST API (record creation plus its separate base64 attachment-upload endpoint), OpenAI Chat Completions (a single bounded classification call, not open-ended chat)
- **Postgres:** schema design for conversation-state tracking, parameterized queries, `COALESCE`-based conditional routing for the correction flow
- **Infrastructure:** isolated Docker Compose deployment, credential-scoped n8n instance, self-hosted on a VPS

## Results & Business Recommendations

Replacing an LLM-driven "recount the whole conversation every turn" design with a deterministic one eliminated a specific, previously-reported failure mode: the registration summary intermittently failing to reflect what was actually said. Recommendations before wider rollout:
1. Run a small real pilot with a handful of CIC Comitium members before opening it up fully.
2. Decide deliberately on Meta Business Verification (raises the daily-conversation cap; not required to function at small scale).
3. Treat the verified in-chat photo as the seed for the two features it was specifically designed to support next (see below). Don't let it sit unused once registrations start.

## Next Steps

- A real end-to-end WhatsApp test with a live member, not just internal review.
- An automated birthday shoutout using the member's verified photo.
- A calendar reminder 2 days ahead of each member's birthday.
- Migrate off the shared demo tenant into a dedicated tenant once CIC Comitium has seen and approved it, per this project's own demo-to-production migration pattern.

---

See `ARCHITECTURE.md` for how the system is put together, `SECURITY.md` for how credentials and data are handled, `PRIVACY.md` for what member data is collected and why, and `revision.md` for the full record of what was found and fixed while building this.
