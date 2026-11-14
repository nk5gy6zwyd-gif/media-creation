# Client Messaging → Claude: Master Overview

> Research date: 2026-11-14 · Synthesised from 9 subagent reports (01–09) · Status: v1

**Goal:** get every way clients contact the business (text, Facebook, Instagram, LiveChat, WhatsApp, email, phone, and informal channels) into one place Claude can read, summarise, track and eventually help reply to.

**Confidence warning:** most subagents worked from search summaries and vendor blogs. Vendor docs were reachable only some of the time, and YouTube and Reddit were largely blocked. Anything marked *verify* in a detail doc should be confirmed before you pay for or build anything. Vendor blogs often argue against a competitor, so treat cost and ban-risk claims from them with care.

---

## 1. The short answer

| Channel | Connectable to Claude today? | Best route | Can Claude reply? | Main blocker |
|---|---|---|---|---|
| **LiveChat** | **Yes, official MCP** (`https://mcp.text.com/`) | Add as custom connector in Claude | No (read-only); use Composio/Zapier or API `send_event` to reply | Plan: "Text App" required; *verify* on your account |
| **Text / SMS (business)** | **Yes, official MCP** via Quo (OpenPhone) | Quo connector (no code) | Yes (prepaid credits) | US A2P 10DLC registration if using Twilio/Telnyx |
| **Text / iMessage (personal phone)** | Partly | Anthropic's iMessage plugin (Mac only) | Yes, but risky | Whole-history access, privacy |
| **Phone calls & voicemail** | **Yes, official MCP** via Quo | Quo connector + recording on | n/a (creates tasks) | Transcripts need Quo Business/Scale; recording consent laws |
| **Facebook Messenger** | No good MCP | Helpdesk hub (respond.io / Chatwoot / Front) or own Meta app + webhook | Yes, within 24 h window | Meta App Review; API only returns last 20 msgs/conversation |
| **Instagram DMs** | Via third party | Inrō hosted connector, or a hub | Yes, within 24 h window | Business/Creator account required |
| **WhatsApp** | Via hub only (safely) | Coexistence via BSP (360dialog/Wati) → hub with MCP | Yes, 24 h window + paid templates | Meta verification; per-message pricing |
| **Email (Gmail)** | **Yes, already connected** | Gmail connector | Drafts / send | None; also the catch-all for transcripts |
| **Meetings (Zoom/Meet/Teams)** | Yes | Zoom connector, Fireflies official MCP | n/a | Notetaker plan |
| **Website forms, bookings** | Yes via glue | Zapier MCP / Make MCP | n/a | Zapier free tier ≈ 50 calls/month |
| **Google reviews / messages** | Community MCPs | Google Business Profile API | Yes | API access approval takes days–weeks |
| **Personal Telegram/Signal/LinkedIn/X** | Experimental | Beeper local MCP | Yes | Personal use only, ToS/privacy risk |
| **In-person / paper / informal** | Manual | Voice memo → transcription → log; photo → OCR | n/a | Discipline |

**Three standouts:**
1. **Quo (formerly OpenPhone)** handles both business texting *and* calls/voicemail in one official Claude connector. It's the strongest single first step.
2. **LiveChat has an official, read-only MCP** at `mcp.text.com`.
3. **Facebook, Instagram and WhatsApp are best handled through one helpdesk hub**, not three separate fragile connectors.

---

## 2. Recommended architecture

```
 Clients ──► Channels ─────────────────────────────► Where Claude reads it
 ─────────────────────────────────────────────────────────────────────────
 Phone calls / voicemail ─┐
 Business SMS ────────────┴─► Quo ─────────────────► Quo MCP connector
 Website chat ─────────────► LiveChat ─────────────► Text MCP (mcp.text.com)
                                 └─ transcript fwd ─► Gmail label "LiveChat"
 Messenger ─┐
 Instagram ─┼──────────────► Hub (respond.io │ Chatwoot │ Front) ► Hub MCP / n8n
 WhatsApp ──┘
 Email ────────────────────► Gmail ────────────────► Gmail connector
 Forms / bookings / reviews ► Zapier or Make ──────► Client Log ► Airtable/HubSpot MCP
 Meetings ─────────────────► Fireflies / Zoom ─────► their MCP
 Informal (in person, personal phone) ► voice memo / forward-to-email ► Gmail / Client Log
```

**The client communication log** (from 01): one table, one row per interaction, which Claude reads through Airtable's or HubSpot's official MCP:

| Field | Example |
|---|---|
| timestamp | 2026-11-14 10:32 |
| client | Jane Smith (link to client record) |
| channel | Instagram DM |
| direction | inbound / outbound |
| summary | Asked about Saturday availability |
| full_text / link | link to original |
| action_needed | Reply with quote |
| status | open / waiting / done |
| owner | who handles it |

Zapier or Make writes a row whenever a new message, transcript, form or booking arrives. Claude summarises and flags anything still open.

### Three budget tiers (from 06)

| Tier | Approx. monthly cost | Stack |
|---|---|---|
| **A. No-code** (start here) | ~$30–80 + Quo plan | Quo + LiveChat MCP + Gmail + Zapier/Make + Airtable log |
| **B. Self-hosted** | ~$10–40 + WhatsApp fees | A + Chatwoot (self-host) + n8n + Claude API |
| **C. Premium hub** | ~$150–500+ | respond.io or Front (official MCPs) for all social channels; GoHighLevel if you also want CRM/booking |

---

## 3. Step-by-step rollout

**Week 1: read-only wins, no code**
1. Connect the **LiveChat/Text MCP** (Claude → Customize → Connectors → add custom connector → `https://mcp.text.com/` → log in). Test: "Summarise yesterday's chats."
2. Turn on **LiveChat transcript forwarding** to a Gmail address, filtered into a `LiveChat` label, as a backup copy.
3. Sign up for **Quo** (or port your number to it). Turn call recording on (Business plan for transcripts) and add the **Quo connector** to Claude. Add a recording disclosure greeting.
4. Start any slow approvals now, since they run in the background:
   - A2P 10DLC registration, if you won't use Quo for SMS
   - Meta Business Verification
   - Google Business Profile API access

**Weeks 2–3: social channels**
5. Pick a hub. respond.io (official MCP, least work) or Chatwoot (cheap, self-host, connect via n8n).
6. Connect **Instagram** (Business/Creator account), **Messenger** (Facebook Page) and **WhatsApp** (via Coexistence through a BSP, so you keep the WhatsApp Business app) to the hub.
7. Connect the hub to Claude.

**Weeks 3–4: tracking**
8. Build the **Client Log** in Airtable (or HubSpot). Add Zapier/Make zaps: new form, new booking, new review, chat ended, call completed → new row.
9. Write custom Claude **skills** (07): `triage`, `reply-drafter` (in your tone), `follow-up-tracker`, `transcript-summariser`. Or install Anthropic's `customer-support` plugin from `anthropics/knowledge-work-plugins` as a starting point.

**Later: replies**
10. Let Claude **draft** replies. You approve every send at first, and never choose "Allow always" on send tools.

---

## 4. Safety rules (from 07, 08)

- **Inbound client messages are untrusted input.** A message can contain instructions aimed at Claude (prompt injection). Invariant Labs demonstrated a real data-exfiltration attack through a WhatsApp MCP.
- Keep **read** and **send** separate. Start with read-only and draft-only.
- Use scoped tokens and the official/OAuth connectors. Avoid MCP servers that ask for your password or save session cookies.
- **Don't** connect your business WhatsApp or Instagram through unofficial QR/password bridges (`lharries/whatsapp-mcp`, `trypeggy/instagram_dm_mcp`, etc.), because the account can be banned.
- Compliance:
  - **SMS:** consent, honour STOP, 8am–9pm quiet hours.
  - **Calls:** recording consent (about 11–13 US states require all parties).
  - **Meta:** 24-hour reply window.
  - **GDPR:** if you have EU/UK clients, you're sending their data to an AI provider.

---

## 5. Top 15 tools and resources (ranked for this business)

| # | Item | Type | Why |
|---|---|---|---|
| 1 | Quo connector (mcp.quo.com) | Official MCP | Calls, voicemail, SMS in one |
| 2 | Text/LiveChat MCP (mcp.text.com) | Official MCP | Chat transcripts, tickets |
| 3 | Gmail connector | Anthropic connector | Already connected; catch-all sink |
| 4 | Zapier MCP / Make MCP | Official MCP | Glue for forms, bookings, reviews, log |
| 5 | respond.io MCP | Official MCP | Instagram + Messenger + WhatsApp + SMS hub |
| 6 | Airtable or HubSpot MCP | Official MCP | Client communication log / CRM |
| 7 | `customer-support` plugin (anthropics/knowledge-work-plugins) | Anthropic plugin | Triage + reply drafting template |
| 8 | Chatwoot + n8n (`czlonkowski/n8n-mcp`, ~23k stars) | Open source | Cheap self-hosted hub |
| 9 | Inrō Instagram connector | Vendor MCP | Instagram DMs with write approval |
| 10 | Front MCP | Official MCP | Premium shared inbox |
| 11 | Fireflies MCP / Zoom connector | Official | Meeting transcripts |
| 12 | Composio LiveChat toolkit | Hosted MCP | Lets Claude *reply* in LiveChat |
| 13 | Twilio MCP (`twilio-labs/mcp`, alpha) / Telnyx MCP | Official MCP | Build-your-own SMS |
| 14 | Anthropic iMessage plugin | Official plugin | Personal iPhone texts (Mac) |
| 15 | Beeper MCP / Unipile MCP | Bridge | Telegram, Signal, LinkedIn, X (personal) |

---

## 6. Conflicts and open questions to verify

- **Twilio MCP:** 03 and 07 both found `twilio-labs/mcp` (official, alpha, ~114 stars). 09 found no official Twilio tooling for voice *transcripts*. Both can be true: the MCP wraps the general API, not Voice Intelligence specifically.
- Does the LiveChat/Text MCP work on legacy LiveChat-only plans, and does it cover human-agent chats or only ChatBot? Test on your account.
- WhatsApp pricing change on 2026-10-01 for in-window replies: check Meta's official pricing page.
- Meta's WhatsApp Business Tools MCP (announced Sept 2026) is reportedly admin-only. *Verify.*
- Quo voicemail transcription plan tier, number porting and webhook retries: confirm with Quo.
- Zendesk and Microsoft 365 official MCP status is disputed.
- Recording-consent rules outside the US (UK/EU/AU): get regulator or legal confirmation.

---

## 7. Document index

| File | Topic |
|---|---|
| [01-other-client-channels.md](01-other-client-channels.md) | Every other way clients reach you + informal channels + client log design |
| [02-livechat.md](02-livechat.md) | LiveChat: official MCP, APIs, transcripts |
| [03-sms-text.md](03-sms-text.md) | SMS providers, iMessage, 10DLC, compliance |
| [04-facebook-messenger-whatsapp.md](04-facebook-messenger-whatsapp.md) | Messenger Platform, WhatsApp Cloud API, Coexistence |
| [05-instagram.md](05-instagram.md) | Instagram Messaging API, Inrō, ManyChat |
| [06-unified-inbox-and-email.md](06-unified-inbox-and-email.md) | Hubs (respond.io, Chatwoot, Front…), automation glue, email |
| [07-claude-skills-and-mcp-ecosystem.md](07-claude-skills-and-mcp-ecosystem.md) | Skills vs MCP vs connectors vs plugins; catalogue; security |
| [08-youtube-and-community.md](08-youtube-and-community.md) | Videos and community lessons (thin, as YouTube was blocked) |
| [09-phone-and-voicemail.md](09-phone-and-voicemail.md) | Phone systems, voicemail, recording law |
