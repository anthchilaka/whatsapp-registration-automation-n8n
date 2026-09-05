# Revision Log: Legion of Mary WhatsApp Registration Bot

How an existing n8n build (three draft versions, `ver1`/`ver2`/`ver3`, migrated from an earlier n8n Cloud subscription) was reviewed, debugged, and re-architected by Claude Code into a working WhatsApp registration bot for a real ministry use case.

**Context.** All three drafts share the same core design: a single AI agent (GPT-4.1-mini via LangChain) driving a 10-step conversation in four languages (English, Igbo, Yoruba, Hausa), collecting a Legion of Mary member's title, name, office, birth date, and photo, then writing the record to Airtable. `ver2` (14 nodes) was the most complete of the three and became the focus of this work. `ver1` and `ver3` (11 nodes each) are referenced throughout as a baseline showing what the original build looked like before revision, and in one case, as the source of a fix.

Each entry below follows the same structure: **Before** (what the workflow did), **Issue Found** (what was actually broken and how that was confirmed), **Fix Applied** (what changed), **Why It Matters** (the consequence of leaving it as-is).

---

## 1. Status-update webhooks would crash the bot on its own outgoing messages

**Before:** The workflow's first real node after the WhatsApp trigger read `$json.contacts[0].profile.name` and `$json.messages[0].text.body` directly, with no check on what kind of webhook call had actually arrived.

**Issue Found:** Checked against Meta's own WhatsApp Cloud API documentation. Every outbound message a WhatsApp bot sends triggers separate "status" webhook callbacks (sent, delivered, read) to the same endpoint the bot uses for real inbound messages, up to three per send. A status callback carries a `statuses` array, not `messages`/`contacts`; accessing those fields on it throws immediately. This is not a rare edge case; it is guaranteed, frequent traffic the moment the bot sends anything.

**Fix Applied:** A new guard node checks whether `$json.messages` exists before any further processing. A status callback now dead-ends harmlessly instead of crashing the workflow.

**Why It Matters:** As built, the bot could not have survived receiving delivery receipts for its own first sent message. This is the kind of bug that only shows up in production, not in manual testing where a developer never triggers a real delivery receipt.

---

## 2. The "registration complete" branch could never actually fire

**Before:** After the AI agent finishes the conversation, it outputs a JSON object (the collected registration data) as its final message. A downstream `If` node checks `$json.registrationComplete` to decide whether to proceed to the Airtable write.

**Issue Found:** n8n's AI Agent node returns `{ output: "<the agent's raw text>" }`. Without a Structured Output Parser attached, `registrationComplete` never exists as a real field on the item; it is buried inside a string. The `If` node's condition could never evaluate true. This same `If` node and the same unparsed-output design exist identically in `ver1` and `ver3`, so this was a latent bug shared across all three original drafts, not a `ver2`-specific mistake.

**Fix Applied:** A new parsing step attempts `JSON.parse` on the agent's output only when it looks like a JSON object, exposing the real fields at the top level; plain conversational replies pass through untouched. Attaching a Structured Output Parser node (the more obvious-looking fix) was considered and rejected: this agent handles all 10 conversation steps in one node, and n8n's own documentation states that parser is meant for an agent's final output only, not intermediate turns. Attaching it would have forced every ordinary reply into the same rigid JSON schema and broken the conversation itself.

**Why It Matters:** Without this fix, no registration could ever complete. The bot would ask all the right questions and then silently fail to save anything.

---

## 3. The photo-confirmation handoff referenced a workflow that does not exist

**Before:** `ver2` waited for the user to type "DONE," then tried to read `getWorkflowStaticData('global')`, expecting a separate workflow ("Workflow B") to have written photo-upload confirmation into it.

**Issue Found:** n8n's own documentation confirms `getWorkflowStaticData` is scoped to a single workflow, not shared across separate workflows. No such second workflow exists anywhere on the instance either way. `ver1` and `ver3` do not have this problem: they use a simpler, working pattern instead. Deterministically construct the expected Cloudinary URL from the phone number (a fixed naming convention), with no cross-workflow dependency at all.

**Fix Applied:** Adopted `ver1`/`ver3`'s working pattern in `ver2`, plus one improvement neither original draft had: a real HTTP existence check against the constructed URL before trusting it, rather than blindly assuming the naming convention held.

**Why It Matters:** This was the one place `ver2` had actually regressed behind its own sibling drafts. The photo step would have hung indefinitely; this branch could never advance.

*(Superseded by Revision 9 below: this fix itself was later replaced by native in-chat photo upload.)*

---

## 4. Every reply went to the developer's own phone number

**Before:** All three WhatsApp-sending nodes had a hardcoded `recipientPhoneNumber` (the developer's personal test number), left over from manual testing.

**Issue Found:** Found while reviewing the send nodes during credential setup. As built, the bot could only ever hold a conversation with one specific phone number, regardless of who actually messaged it.

**Fix Applied:** Recipient is now derived dynamically from the actual sender on every message.

**Why It Matters:** A registration bot that only replies to its own developer is not a bot that works for anyone else.

---

## 5. Airtable field mapping was writing corrupted, unevaluated text into a real production base

**Before:** All 9 field mappings on the Airtable write node contained literal text like `"Field: First Name\nValue: ={{ $json.firstName }}"` baked in as the field value.

**Issue Found:** In n8n, once a field's value starts with `=`, the entire string becomes the expression template. This was not a hypothetical bug: the live base already showed the evidence. The `Summary` column of every existing test row literally contained the raw string `{{ $json.summary }}` instead of an actual summary. The AI-generated summary was also mapped to the wrong Airtable column entirely (`Attachment Summary`, a formula/rollup field, likely read-only) instead of the real plain-text `Summary` column, found by cross-referencing the agent's own JSON schema against a screenshot of the live base.

**Fix Applied:** All 9 fields rewritten to clean expressions, and the summary redirected to the correct column.

**Why It Matters:** This bug was already live: real (test) data had already been written incorrectly to the production base before it was caught.

---

## 6. The entire conversation relied on the AI re-deriving its own memory from scratch every turn

**Before:** The agent's prompt included a "FIELD COUNTING RULE," asking it to re-read the full raw chat transcript on every single turn and reconstruct which of the 6 required fields had already been answered, purely through its own language reasoning.

**Issue Found:** This was reported directly by the project owner as an existing, observed problem: the registration summary would intermittently fail even with a conversation-memory node already in place. The real cause was architectural, not a memory-size setting: an LLM re-deriving state from unstructured text is inherently error-prone, especially after a typo, a correction, or a retry. A larger memory buffer would only delay the failure, not fix it.

**Fix Applied:** A full redesign, replacing LLM-based state tracking with a deterministic state machine: a Postgres table holds one column per field plus a `current_step` column, keyed by phone number. n8n itself decides which field an incoming message answers and what to ask next; the LLM is no longer responsible for tracking anything. This is the same pattern already proven in a separate, related project's own Telegram bot (`pending_action`-style state tracking), applied here for the first time to a WhatsApp flow.

**Why It Matters:** This is the single largest change in this revision history: it turns a fundamentally unreliable design (conversation state as a side effect of LLM memory) into a fully deterministic one, where the same input reliably produces the same result.

---

## 7. Removing the LLM from state-tracking also silently removed automatic language detection

**Before (mid-redesign):** The original agent auto-detected which of the four languages to reply in from the user's own words, as one of its many jobs.

**Issue Found:** Once the state machine took over conversation-driving, this quiet secondary job the LLM was also doing disappeared with it; nothing else was watching for it.

**Fix Applied:** A `language` column added to the state table, detected once via the same character/keyword markers the original prompt already used (Igbo `ọ/ụ/ị`, Yoruba `ẹ/ọ/ṣ`, Hausa `"don allah"/"na gode"`), then carried forward through every subsequent deterministic step, matching the original design's own "detect once, maintain throughout" behaviour.

**Why It Matters:** A worthwhile lesson on its own: replacing an AI-driven component with a deterministic one requires auditing everything that component was doing, not just its obvious primary job.

---

## 8. Messy, off-format answers had no fallback once the LLM was removed

**Before (mid-redesign):** The new deterministic validators (regex and enum matching) reject anything that does not cleanly match, such as "I'm a brother in the parish" instead of "Bro."

**Issue Found:** A fully rigid deterministic system trades away the one thing the original LLM-driven design was actually good at: tolerating loosely-worded human answers.

**Fix Applied:** One shared, reusable fallback, not duplicated per field: any failed validation routes to a single LLM-assist step (one OpenAI call, temperature 0, a one-line field-specific classification prompt), whose output is then re-validated using the exact same deterministic rules as the raw answer, never trusted blindly.

**Why It Matters:** Keeps the LLM's role narrow, bounded, and auditable, used only where it adds real value, rather than reintroducing the original reliability problem through the back door.

---

## 9. Photo capture depended on an external website and an unverified filename guess

**Before:** The bot sent users a link to an external, third-party-hosted web page to upload their photo, then asked them to type "DONE" back in WhatsApp; the bot then guessed the resulting file's name from the phone number and only checked that *something* existed at that URL.

**Issue Found:** Researched against Meta's own WhatsApp Cloud API documentation for a better-supported approach. Confirmed: users can simply send a photo as a normal WhatsApp attachment. The incoming webhook then carries a `media_id`, downloadable via a documented two-step API call using the same access token already in use. n8n's own WhatsApp node (the exact node type already used everywhere else in this workflow) has a built-in Media resource with Download/Upload/Delete operations, so this needs no custom workaround.

**Fix Applied:** Photo capture moved to native in-chat upload. The user is asked to send their photo directly in the chat; the incoming webhook's message type confirms it's really an image, and the message's media ID (not the file itself) is stored immediately; the actual download is deferred to the moment registration is confirmed, since a WhatsApp media ID stays valid for 7 days, comfortably longer than a normal confirmation delay. At confirmation, the photo is re-fetched via the same two-step WhatsApp Media API call, then attached to the Airtable record through Airtable's own separate upload endpoint, discovered via Airtable's own API documentation during this phase: an attachment field can't be set in the same call that creates a record, and doesn't accept a raw binary upload through the normal field-mapping form. It needs its own follow-up call, sent as base64, to a record that already exists. This is a genuine revision to the plan above, made only after checking Airtable's real documented behaviour rather than assuming it.

**Why It Matters:** No external website, no leaving WhatsApp mid-registration. It also serves this project's own stated next phase directly: an automated birthday shoutout using the member's photo, later a calendar reminder. A photo verified as belonging to this exact registration is a materially more trustworthy source for that feature than one merely guessed to exist at a predictable filename.

---

## 10. The Postgres table backing the entire deterministic state machine had never actually existed

**Before:** Revision 6's redesign, and every field-collection step built after it, depended on a table, `legio_registration_progress`, that prior notes described as already created via a direct database session.

**Issue Found:** Resuming this work, the table was checked directly against the live database rather than trusted from notes: it didn't exist, on that database or any other on the same host. Every node built against it had validated cleanly the whole time, because n8n's own workflow validator checks node configuration, not whether a table it queries is real; nothing had caught this because there had been no real end-to-end test yet.

**Fix Applied:** Table created for real, matching every column the existing nodes already expected, plus the additional columns this phase needed.

**Why It Matters:** Without this, none of Revision 6 through 9's design, however correct on paper, had ever actually run. A workflow validating clean is not the same as its dependencies being real.

---

## 11. Registration completion had no real destination once the agent was removed

**Before:** With the agent gone, nothing built the record the Airtable write node was expecting, or the summary text the success message displayed.

**Issue Found:** The Airtable node's field mappings and the success message's text both still pointed at the old agent-output-parsing step, which no longer runs anything relevant to this path.

**Fix Applied:** A new mapping step reads the Postgres row's underlying column names and produces the shape the Airtable node was already built to expect, plus a freshly built confirmation summary string, both fed from real, committed data rather than an LLM's own reconstruction.

**Why It Matters:** This is the actual finish line of the registration flow. Everything built in Revisions 6 through 10 only matters if a completed registration can still reach Airtable and tell the user it succeeded.

---

## 12. The "NO, let me change something" path had never been solved, in either version

**Before:** Flagged as an open gap in Revision 9's notes: the original agent-driven design only asked "what would you like to change" and hoped further conversation handled it. No version of this bot had ever actually routed a correction anywhere.

**Issue Found:** Solving this deterministically meant reusing the field-collection logic already built, not duplicating it: the same question that fires on a fresh registration also needed to fire mid-correction, then return to the confirmation summary afterward instead of continuing forward through the rest of the questions.

**Fix Applied:** A "return here when done" marker was added, checked at the moment any field's answer is saved: normally it's empty and the flow just advances to the next question as before; during a correction it's set, gets read first, and sends the person straight back to the confirmation summary once the one field they meant to fix is updated. A new step reads what the user names (office, name, photo, etc.) and jumps them to that question using this same mechanism.

**Why It Matters:** A registration bot people actually use will get typos and second thoughts. Without this, the only way to fix a mistake after confirmation was to abandon the conversation and start over.

---

## 13. The success message still had the same two problems this revision history already flagged elsewhere

**Before:** Reusing this node for the newly-deterministic completion path required first checking its actual configuration, not assuming Revision 4's fix still held.

**Issue Found:** It still had the developer's personal number hardcoded as the recipient (the same class of bug fixed everywhere else in Revision 4) and a malformed expression prefix that would have sent the literal, unrendered text rather than the intended message.

**Fix Applied:** Recipient made dynamic, matching every other send node; expression prefix corrected.

**Why It Matters:** A bug fixed once in a workflow doesn't stay fixed if a node carrying the same broken pattern is left behind or reused later. Worth re-checking a previously-fixed category of bug on any node being touched again, not just the node originally reported.

---

## 14. The agent, its memory, and the guess-and-check photo trio were fully removed from the flow

**Before:** The agent, its language model and memory, its output-parsing step, and the old confirmation branch (Revision 3's fix, itself superseded by Revision 9) remained in the workflow, reachable only as a fallback for a registration state nothing produces anymore.

**Issue Found:** None of the real registration states (title through the new correction step) ever route to this branch once the deterministic design covers all of them. It was dead weight, not a safety net.

**Fix Applied:** All ten nodes removed.

**Why It Matters:** This is the actual completion of Revision 6's stated goal: not just adding a deterministic path alongside the agent, but retiring the agent-driven design entirely.

---

## Summary

| # | Area | Kind of fix |
|---|---|---|
| 1 | WhatsApp trigger | Crash prevention (external API behaviour) |
| 2 | Agent output handling | Correctness (data never reaching its destination) |
| 3 | Photo confirmation | Broken cross-workflow design, fixed by adopting a sibling draft's working pattern |
| 4 | Message recipient | Configuration bug (hardcoded test value) |
| 5 | Airtable write | Data corruption, already live in production test data |
| 6 | Conversation state | Full architectural redesign (LLM-driven to deterministic) |
| 7 | Language detection | Consequence of Revision 6, caught by auditing what else the LLM was doing |
| 8 | Answer tolerance | Consequence of Revision 6, solved with a narrow, shared LLM fallback |
| 9 | Photo capture | Replaced an external-dependency workaround with a native platform feature |
| 10 | Postgres table | Infrastructure gap found via re-verification, fixed before building further |
| 11 | Registration completion | Correctness (Airtable write and success message reconnected to real data) |
| 12 | Correction handling | Full solve of a previously-open gap (per-field correction routing) |
| 13 | Success message | Configuration bug and malformed expression, caught while reusing the node |
| 14 | Agent retirement | Completion of Revision 6's architectural redesign |
