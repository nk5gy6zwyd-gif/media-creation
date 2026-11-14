# Client Communications System: Plan (v0, pre-interview)

> Status: **DRAFT, under interview.** Built from `research/messaging-integrations/` (00–09).
> Decisions marked **[Q#]** are open and will be settled in the interview round, then this doc is revised to v1.
> Scope: plan only. The web interface and secure login are **planned here, not built**.

---

## 1. Goal

One system where every client message, call, chat, email, form, booking and informal conversation lands in a **single client timeline**. You can use it from **iPhone, iPad and iMac**, with Claude to summarise, flag what needs action, track follow-ups and draft replies that you approve.

### Success criteria
1. Nothing a client sends goes untracked for more than **[Q: target, e.g. 15 min]**.
2. A daily brief tells you who's waiting, what's overdue and what's new.
3. Any client's full history can be pulled up in seconds on any device.
4. Claude drafts replies; **you approve every send** (at least in Phase 1–3).
5. Setup and running cost stays within **[Q: monthly budget]**.

---

## 2. Device roles (Apple ecosystem)

| Device | Role | What runs there |
|---|---|---|
| **iPhone** | Capture + respond on the go | Claude iOS app with connectors; Quo app (business line); Shortcuts for "log this to client" (share sheet, screenshots, voice memo); push notifications; web dashboard as a home-screen app (PWA) |
| **iPad** | Review + triage desk | Claude iPadOS app; web dashboard in Split View next to Claude; the same Shortcuts |
| **iMac** | **Always-on hub** for things only a Mac can do | Claude Desktop with local MCPs (iMessage plugin reads `chat.db`; optional Beeper bridge); Claude Code for maintenance; a folder for voice-memo transcription; scheduled jobs (daily brief) |
| **Cloud** | Source of truth | Client Hub database + webhook receiver + own remote MCP server; vendor connectors (LiveChat/Text, Quo, Gmail, hub, Airtable/Zapier) |

**Key fact:** *remote* connectors (MCP servers reached over the internet with OAuth) are attached to your Claude account, so they work the same on iPhone, iPad and iMac. *Local* MCPs (iMessage, Beeper) only run on the iMac, which is why the iMac is the hub. iPhone and iPad have **no API** for reading SMS, iMessage or call history, so anything personal-phone must go through the iMac or a business number (Quo).

---

## 3. Architecture

```
            ┌──────────────────────────── CHANNELS ────────────────────────────┐
 Calls/VM ─► Quo ──────────┐     LiveChat ─► Text MCP (read) + chat_deactivated webhook
 Business SMS ─► Quo ──────┤     Messenger/Instagram/WhatsApp ─► Hub (respond.io | Chatwoot)
 Email ─► Gmail ───────────┤     Forms/Bookings/Reviews ─► Zapier/Make
 Meetings ─► Fireflies/Zoom┤     Personal iMessage ─► iMac (iMessage plugin)
 Informal ─► iPhone Shortcut (voice memo / screenshot / note) ─┘
            └──────────────────────────────┬──────────────────────────────────┘
                                           ▼
                     ┌──────────── CLIENT HUB (cloud) ────────────┐
                     │ Ingestion: webhooks + n8n/Zapier            │
                     │ DB: clients, interactions, follow-ups,      │
                     │     drafts, audit log                       │
                     │ Claude worker: summarise, classify, flag,   │
                     │     draft (Claude API, approval queue)      │
                     │ Own remote MCP server (OAuth)               │
                     └───────┬───────────────────────┬────────────┘
                             ▼                       ▼
            Claude apps (iPhone/iPad/iMac)   Web dashboard (PWA, secure login)
            via connectors                    [planned, not built]
```

### 3.1 Data model (Client Hub)
- **clients**: id, name, phones[], emails[], social handles[], tags, notes, status
- **interactions**: id, client_id, channel, direction, timestamp, raw_text / transcript, attachments, source_ref (link back), summary, sentiment, intent, action_needed
- **follow_ups**: id, client_id, interaction_id, due_at, owner, status
- **drafts**: id, interaction_id, channel, body, status (pending/approved/sent/rejected), approved_by, sent_at
- **audit_log**: who/what read or sent what, when
- **identity matching**: phone/email/handle → client (with a "who is this?" queue for unknowns)

### 3.2 Build options for the hub ([Q] which)
| Option | What it is | Pros | Cons |
|---|---|---|---|
| A. **No-code** | Airtable (official MCP) + Zapier/Make | Fastest, no hosting | Per-task cost, weaker security model, limited UI |
| B. **Low-code** | Supabase (Postgres + auth) + n8n + small MCP server | Cheap, owns data, real auth | Some build + maintenance |
| C. **Existing CRM** | HubSpot (official MCP) as the hub | CRM features included | Lock-in, pricing tiers |
| D. **Off-the-shelf inbox** | respond.io / Front as the hub | Least build | Monthly per-seat cost, weaker cross-channel log |

Working recommendation: **start with A to prove value, move to B** once the workflow is right.

---

## 4. Channel-by-channel plan

| Channel | Phase | Capture | Claude access | Reply path |
|---|---|---|---|---|
| LiveChat | 1 | Text MCP + transcript forwarding to Gmail + `chat_deactivated` webhook → Hub | Text MCP (read) | Composio/API `send_event`, with approval (Phase 4) |
| Email (Gmail) | 1 | Gmail labels | Gmail connector | Drafts in Gmail |
| Calls & voicemail | 1 | Quo (Business plan, recording + disclosure) → webhooks → Hub | Quo connector | Tasks / callback reminders |
| Business SMS | 1 | Quo | Quo connector | Quo send (approval) |
| Personal iMessage/SMS | 2 | iMac iMessage plugin, allowlist of client contacts only | Local MCP on iMac | Manual or approved send from iMac |
| Instagram / Messenger / WhatsApp | 2 | Hub (respond.io or Chatwoot); WhatsApp via Coexistence | Hub MCP / n8n | Hub send, 24 h window, approval |
| Forms / bookings | 2 | Zapier/Make → Hub | Hub MCP | n/a |
| Google reviews | 3 | GBP API (apply early) | Hub | Approved review replies |
| Meetings | 3 | Fireflies/Zoom → Hub | Their MCP | n/a |
| Informal (in-person, paper, other apps) | 1 | iPhone Shortcut: dictate / screenshot / share → Hub | Hub MCP | n/a |

---

## 5. iPhone/iPad/iMac workflows

1. **Morning brief** (scheduled, 7:30 **[Q]**): who's waiting, overdue follow-ups, new leads, yesterday's summaries. Delivered as push + email + dashboard.
2. **"Log it" Shortcut** (iPhone/iPad share sheet + Action Button/Back Tap): pick client → dictate or attach → saved as an interaction.
3. **"Who is this?"** Shortcut: paste a number/handle → client card + last 5 interactions.
4. **Approval queue:** Claude-drafted replies wait in the dashboard/Claude; tap approve/edit/reject.
5. **Urgent ping:** messages Claude classifies as urgent → push notification within **[Q]** minutes.
6. **Ask Claude anything:** in the Claude app on any device: "What did the Smiths ask about last month?"
7. **End-of-day sweep:** anything unanswered gets a follow-up created.

---

## 6. Web interface (PLANNED ONLY, not being built now)

### 6.1 Form factor
- Responsive **PWA**: add to Home Screen on iPhone/iPad, web push notifications (iOS 16.4+), full-screen on iMac.
- Views: **Inbox** (all channels, filters), **Client timeline**, **Approval queue**, **Follow-ups**, **Search**, **Daily brief**, **Settings/connections**.

### 6.2 Secure login (plan)
- **Passkeys (WebAuthn) first.** Face ID / Touch ID on all three devices, synced by iCloud Keychain. No passwords to phish.
- Fallback: Sign in with Apple, or email magic link plus TOTP.
- Sessions: short-lived, device list with "sign out everywhere", re-auth (Face ID) before **sending** anything or exporting data.
- Optional extra wall: Cloudflare Access / Tailscale in front of the app so it's not publicly reachable.
- Roles if staff join: owner / staff / read-only **[Q]**.

### 6.3 Security & privacy baseline
- All channel tokens in a secrets store, never in the browser. OAuth scopes minimal.
- Inbound messages treated as **untrusted** (prompt-injection defence): Claude worker has no send tool; sending happens only from the approval queue.
- Encryption at rest and in transit; audit log of every read, draft and send.
- Data retention policy **[Q]**; GDPR/consent handling; recording disclosures.
- Backups (daily) and an export button.

### 6.4 Suggested stack (for when it's built)
Next.js (or SvelteKit) PWA · Supabase (Postgres, Auth with passkeys, Row-Level Security, Edge Functions for webhooks) · n8n for ingestion · Claude API worker · custom remote MCP server so the Claude apps see the same data.

---

## 7. Phased roadmap

| Phase | Weeks | Deliverables | Exit test |
|---|---|---|---|
| **0. Approvals & accounts** | 0–1 | Quo account + number decision; start A2P/Meta verification/GBP API; LiveChat plan check | Approvals submitted |
| **1. Read-only Claude** | 1–2 | Connect Text MCP, Quo, Gmail in Claude (all devices); transcript forwarding; "Log it" Shortcut → Airtable | Ask Claude on iPhone "what's waiting?" and get a correct answer |
| **2. Single timeline** | 2–4 | Hub (Airtable or Supabase) + Zapier/n8n ingestion for all Phase-1/2 channels; identity matching; social hub connected | Every test message from 6 channels appears under the right client |
| **3. Brief + follow-ups** | 4–5 | Daily brief, urgent push, follow-up tracker, custom Claude skills (triage, summariser, follow-up, reply-drafter in your voice) | One week with zero missed follow-ups |
| **4. Approved replies** | 5–7 | Draft → approve → send for SMS, LiveChat, social, email | 20 approved sends, zero wrong-recipient |
| **5. Web dashboard** | later | PWA + passkey login per §6 | Security review passed |

---

## 8. Costs (rough, verify)
| Item | Est. monthly |
|---|---|
| Quo (Business, per user) | verify (~$20–40) |
| Zapier/Make | $20–70 |
| Airtable (or Supabase free → $25) | $0–25 |
| Social hub (Chatwoot self-host ~$10 / respond.io paid) | $10–200 |
| WhatsApp per-message fees | usage |
| Claude plan / API usage | existing + usage |
| **Total (Tier A start)** | **≈ $60–150** |

---

## 9. Risks
- **Personal phone gap:** iPhone texts/calls can't be read directly; mitigated by Quo + iMac iMessage plugin (Mac must stay on and signed in).
- Meta approvals and 24 h window; WhatsApp pricing changes.
- Prompt injection via client messages; mitigated by no-send worker + approval queue.
- Vendor claims unverified (see research 00 §6).
- Recording-consent law by state/country.

---

## 10. Open questions → interview
Tracked as **[Q]** above; answers will be recorded in §11 and the plan revised to v1.

## 11. Decisions log
_(to be filled after the interview)_
