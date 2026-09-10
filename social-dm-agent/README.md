# Facebook Messenger DM Agent — Setup Guide

An AI agent that auto-replies to Adam Bernard Solicitors' Facebook Page DMs: answers
general questions in brand voice, qualifies leads, and hands off anything
resembling actual legal advice to a human.

This is built as a **Zap in Zapier** (not custom code), because Zapier already
has a Facebook Messenger connector and an AI step. This repo just documents
the exact build and holds the AI's brain (the system prompt).

## Why a human handoff, not full auto-reply

A solicitor's firm DM'ing back specific legal advice with no human review is a
real liability (unauthorized/negligent advice exposure). This agent only
handles: FAQs, scheduling/availability questions, and lead intake. Anything
that looks like "what are my rights in my specific case" gets a holding reply
+ an internal alert, not an AI-generated legal answer.

## 1. Connect Facebook Page to Zapier

1. In Zapier, go to **My Apps** → connect **Facebook Pages** and **Facebook
   Messenger**, authorizing the Adam Bernard Solicitors Page.
2. (This session already has a `FacebookMessengerCLIAPI` connection enabled
   via Zapier MCP for the `send_message` action — auth URL was provided
   separately. That's useful for testing a single reply from chat, but the
   live automation below still needs to be built as a Zap.)

## 2. Build the Zap

**Trigger**
- App: **Facebook Messenger**
- Event: **New Message** (fires on new inbound DM to the Page)

**Step 2 — Filter by Zapier** *(optional but recommended)*
- Continue only if `Message Text` exists (skip attachments-only pings, etc.)

**Step 3 — AI by Zapier** (or "ChatGPT"/"Claude" action if you prefer a named
model)
- System prompt: paste the contents of [`system-prompt.md`](./system-prompt.md)
- User input: map in the incoming `Message Text` and `Sender Name`
- **Ask it to return structured JSON**, not just prose — e.g.:
  ```json
  {
    "reply": "the text to send back to the customer",
    "needs_human": true/false,
    "reason": "why it needs a human, if applicable",
    "lead": {
      "name": "...", "contact": "...", "matter_type": "immigration|family|employment|property|unknown",
      "urgency": "low|medium|high"
    }
  }
  ```
  Zapier's AI step can parse this into separate output fields for the next
  steps — turn on "Structured output" / define the output fields to match.

**Step 4 — Paths by Zapier** (conditional branch on `needs_human`)

- **Path A: `needs_human` = false**
  - Action: **Facebook Messenger → Send Message**
  - Message: the AI's `reply` field
  - Recipient: the original sender (map from trigger)

- **Path B: `needs_human` = true**
  - Action: **Facebook Messenger → Send Message**
  - Message: a fixed holding reply, e.g. *"Thanks for reaching out — this
    sounds like something one of our solicitors should look at directly.
    Someone from the team will follow up with you shortly."*
  - Action: notify a human — pick one:
    - **Email by Zapier** or **Gmail → Send Email** to the intake inbox
    - **Slack → Send Channel Message** to a #new-leads channel
    - **Table by Zapier** / **Google Sheets → Create Row** as a lead log
  - Include: sender name, the original message, AI's `reason`, and any
    extracted `lead` fields.

**Step 5 — Log every conversation** *(recommended for compliance/audit)*
- Action: **Table by Zapier** or **Google Sheets → Create Row**, on *both*
  paths — log timestamp, sender, message, AI reply or "escalated", and reason.

## 3. Guardrails to configure

- Cap AI replies to FAQ/logistics only — this is enforced by the system
  prompt, but also sanity-check with a few adversarial test DMs (see below).
- Rate-limit: if the same sender DMs repeatedly in a short window, Zapier
  Filter step to avoid spamming them with repeated auto-replies.
- Office hours note: the system prompt should make clear replies are
  automated and outside-hours, so expectations are set correctly.

## 4. Test plan before going live

Send these as test DMs to the Page and verify behavior:
1. "What areas of law do you cover?" → should get a clean FAQ answer.
2. "How do I book a consultation?" → FAQ answer + booking info.
3. "My landlord is trying to evict me illegally, what should I do?" →
   should escalate (holding reply + internal alert), NOT give advice.
4. "Can I get a work visa if I have a criminal record?" → should escalate.
5. Gibberish / spam → should reply gracefully or skip, not crash the Zap.

## 5. Fill in before launch

These placeholders live in `system-prompt.md` and `faq-data.md` — replace
with the firm's real details (office hours, phone, address, booking link,
consultation fee/pricing language, practice areas) before turning the Zap on.
