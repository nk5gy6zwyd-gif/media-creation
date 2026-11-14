# Customer Communications System: Plan v1

> **Who it's for:** one internet sales manager at a Toyota dealership in **BC/Alberta, Canada**.
> **Devices:** iPhone, iPad, iMac (on 24/7).
> **Status:** v1, rewritten after the interview (see §13 Decisions log). Plan only, with nothing built yet.
> The **web dashboard and secure login are planned (§9) but not built.**
> **Inputs:** `research/messaging-integrations/` 00–09.
> **Price disclaimer:** prices are approximate list prices (mostly USD), as of late 2026, and must be verified before purchase.

---

## 1. Goal

Every customer touchpoint lands in **one customer timeline** that you can reach from iPhone, iPad and iMac. That includes website/Toyota.ca leads, AutoTrader/CarGurus, Facebook Marketplace, Instagram, Messenger, LiveChat, texts, calls, email and showroom conversations. The timeline is kept in step with **DealerSocket**, and Claude:
- summarises every conversation,
- flags urgent items to your phone,
- tracks sales stage and follow-up cadence,
- **drafts replies that you approve before anything is sent**,
- emails you a **morning brief at 7:30 am**.

### Success criteria
1. Every new lead gets a first response within your target time (default **15 min** during shift), and the time is measured.
2. Zero forgotten follow-ups: each lead has a day 1/3/7/14/30 cadence with drafted messages.
3. Any customer's full history (all channels plus showroom notes) comes up in under 10 seconds on iPhone.
4. DealerSocket stays the official record. Nothing important lives *only* in the side system.
5. Customer data at rest stays in **Canada**, and the setup stays within dealership approval.

---

## 2. Constraints that shape everything

| Constraint | Consequence |
|---|---|
| **Volume ~75–200 contacts/day** | Per-task automation pricing (Zapier) gets expensive, so self-hosted n8n is preferred. Auto-sorting and urgency alerts are essential, and the brief is summarised, not a full list. |
| **Personal iPhone number used by customers today** | iPhone has no API for texts or calls. **Decision: a second Quo business number** for new customers and gradual migration. The iMac reads personal iMessages for customers still on the old number. |
| **iMac on 24/7** | The iMac is the local hub. It runs the iMessage reader, n8n (free self-host), local voice-note transcription and scheduled jobs. |
| **Employer data, AI use approved, CRM API access uncertain** | DealerSocket integration is designed with **fallbacks**: API, then email (ADF/XML), then scheduled report export, then copy-assist. See §5. |
| **Canada-only storage** | The hub database lives in a Canadian region. Third-party tools (Quo, Gmail, Meta, Anthropic) still process data in the US, so this is disclosed to and signed off by dealership IT (§10). |
| **Draft-and-approve only** | Claude never sends on its own. Every outbound message goes through an approval tap. |
| **Mix of dealership and personal accounts** | Dealership pages and LiveChat need an admin to connect them. Your own profiles you can connect yourself. |

---

## 3. Device roles

| Device | Role | What runs there |
|---|---|---|
| **iPhone** | Capture and respond on the lot | Quo app (business line); Claude app with connectors; **Shortcuts**: Voice note, Screenshot/Share, Card/Photo, Quick note, Who-is-this; urgent push alerts; dashboard home-screen app (later) |
| **iPad** | Desk triage | Claude app plus dashboard in Split View; the same Shortcuts |
| **iMac (24/7)** | **Local hub** | Claude Desktop with the **iMessage plugin** (reads only allowlisted customer threads); **n8n** (self-hosted, free) for ingestion; **whisper.cpp** for local voice-note transcription; Cloudflare Tunnel (free) so webhooks reach n8n securely; nightly backups |
| **Cloud (Canada)** | Source of truth for the side system | **Supabase, Canada (Central) region**: Postgres DB, auth, file storage, webhook functions. Custom **remote MCP server** so Claude on all devices can query the hub |

> Remote connectors (Quo, Text/LiveChat, Gmail, the hub's own MCP) follow your Claude account, so they work on all three devices. Local MCPs (iMessage) work only on the iMac.

---

## 4. Architecture

```
 SOURCES                                  INGEST (iMac n8n + Supabase functions)        HUB (Supabase, Canada)
 ───────────────────────────────────────  ────────────────────────────────────────────  ─────────────────────────
 Toyota.ca / dealer site leads ──email──►  Gmail label "Leads" → parse ADF/XML ───────►  customers
 AutoTrader.ca / CarGurus ───────email──►  Gmail label "Leads" → parse ─────────────────►  leads (stage, vehicle,
 LiveChat (dealer) ── transcript email ──►  Gmail label "LiveChat" → parse ─────────────►        source, trade…)
                   └─ Text MCP (read, if admin connects) ─────────────────────────────►  interactions
 Quo calls/VM/SMS ───── webhooks ───────►  Supabase function ───────────────────────────►  follow_ups
 Personal iMessage ──── iMac plugin ────►  n8n poll (allowlist) ────────────────────────►  drafts (approval queue)
 Instagram / Messenger (own + dealer) ──►  Chatwoot (self-host) or respond.io → webhook ►  vehicles / deals
 Facebook Marketplace ── screenshot ────►  iPhone Share Shortcut → OCR → Claude parse ──►  audit_log
 Showroom / walk-ins ─── voice note ────►  Shortcut → whisper.cpp (iMac) → Claude parse ►
 Business cards / worksheets ── photo ──►  Shortcut → Claude vision → parse ────────────►
 DealerSocket ◄──────────── sync (see §5) ─────────────────────────────────────────────►  dealersocket_sync

 OUT: Claude worker (Claude API: summarise, classify, urgency, stage, draft)
      → urgent push (iPhone)  → 7:30 am brief email  → approval queue → send via Quo / Chatwoot / Gmail / LiveChat
 READ: Claude apps on iPhone/iPad/iMac via the hub's MCP server + vendor connectors
```

---

## 5. DealerSocket integration (start here), with fallbacks

> DealerSocket (Solera) API access was **not** researched in the earlier sweep. A follow-up research task is listed in §12. Pick the highest level that IT allows.

| Level | Method | Direction | Needs | Notes |
|---|---|---|---|---|
| **L1: API / MCP** | DealerSocket partner/open API (via a certified vendor or dealer-authorised credentials); wrap it in our MCP server | Two-way: read customers/leads/appointments, write activities and notes | Dealer authorisation + Solera partner access | Best, but likely gated to certified vendors. Ask IT. |
| **L2: Lead email in (ADF/XML)** | The hub sends new leads it captures (Marketplace, walk-ins, Instagram) to DealerSocket's **lead-intake email address** in ADF/XML | Hub → DealerSocket | The intake address from the CRM admin | ADF/XML is the industry standard. Every dealer CRM accepts it. |
| **L3: Scheduled report export** | DealerSocket scheduled reports (leads, appointments, sold, owner/maturity lists) emailed daily as CSV/Excel → Gmail label → n8n imports | DealerSocket → Hub | Report access for your user | Gives the hub stages, appointments, sold status and lease maturities. |
| **L4: Copy-assist** | Claude formats an activity note per conversation; a **"Copy for DealerSocket"** button/Shortcut puts it on the clipboard to paste | Manual | Nothing | Always available, and keeps DealerSocket the official record. |
| **Avoid** | Screen-scraping or browser automation of DealerSocket | n/a | n/a | Likely breaches the CRM's terms and dealership policy. |

**Adapter design:** the hub talks to CRMs through one `crm_adapter` interface with `push_lead`, `push_activity`, `import_report` and `lookup_customer`. DealerSocket is implemented first. Adapters planned for: **VinSolutions** (Cox), **Elead / CDK Drive** (CDK), **PBS**, **Activix**, **Momentum**, **Reynolds**, plus a generic **CSV + ADF/XML** adapter that works with any CRM.

---

## 6. Channel-by-channel plan

| Channel | Owner | Capture | Claude reads via | Reply (draft, then you approve) | Phase |
|---|---|---|---|---|---|
| Toyota.ca / dealer website leads | Dealer | ADF/XML lead emails to Gmail label | Hub MCP, Gmail | Quo SMS / email | 1 |
| AutoTrader.ca / CarGurus | Dealer | Lead and message notification emails | Hub MCP | Their portals / email (copy-assist) | 1 |
| LiveChat (website) | Dealer | Transcript forwarding to your email; Text MCP if an admin connects it | Text MCP / Hub | LiveChat (manual or API later) | 1 |
| Calls / voicemail / SMS (new number) | You | **Quo** (Business plan, recording + disclosure) webhooks | Quo connector, Hub | Quo send | 1 |
| Personal iMessage/SMS | You | iMac iMessage plugin, **allowlist of customers only** | Local MCP, Hub | Send from iMac after approval | 2 |
| Showroom / walk-ins / lot | You | **Voice-note Shortcut** → local Whisper → Claude parse | Hub | n/a | 1 |
| Facebook Marketplace | You | **Screenshot/Share Shortcut** → Claude vision parse | Hub | Manual in Messenger, with Claude-drafted text copied to the clipboard | 1 |
| Instagram / Messenger (own profiles) | You | Chatwoot or respond.io (Business/Creator account required for Instagram) | Hub | Via inbox tool, 24 h window | 2 |
| Instagram / Messenger (dealer pages) | Dealer | Same, connected by a page admin | Hub | Same | 3 |
| Business cards / worksheets | You | Photo Shortcut → Claude vision | Hub | n/a | 2 |
| Email | You | Gmail (connected) | Gmail connector | Gmail drafts | 1 |

**Facebook Marketplace note:** Meta offers no API for personal Marketplace chats. Bridges like Beeper break Meta's terms and risk the account, so they're **not used**. The screenshot Shortcut is the supported path.

---

## 7. What gets tracked (data model)

- **customers**: name, phones, emails, social handles, preferred channel, language, consent flags (CASL express/implied + date), VIP flag, DealerSocket ID
- **leads/deals**:
  - source (Toyota.ca, AutoTrader, CarGurus, Marketplace, walk-in…), received_at, **first_response_at** (response time)
  - **stage**: new, contacted, appointment set, shown, test drive, negotiating, sold, delivered, or lost (with reason)
  - **vehicle of interest**: model, trim, new/used, stock #, colour
  - **trade-in**: vehicle, KMs, condition notes
  - **delivery & after-sale**: delivery date, 3-day/30-day check-ins, review request, referral ask
  - **lease/loan maturity**: term end date, so outreach can start 6/3/1 months before
- **interactions**: channel, direction, timestamp, transcript/text, attachments, summary, intent, urgency, link to source
- **follow_ups**: cadence step (day 1/3/7/14/30, post-delivery, maturity), due_at, status, draft_id
- **drafts**: channel, body, status (pending/approved/edited/rejected/sent), approved_at
- **audit_log**: every read, draft, approval and send

**"And more" extras** (suggested, confirm later): birthdays and purchase anniversaries, service-visit triggers, household or co-buyer links, and lost-lead re-engagement after 60/90 days.

---

## 8. Daily workflows

1. **7:30 am brief (email, work days):**
   - new leads overnight, by source
   - customers waiting on you, oldest first
   - today's appointments
   - overdue follow-ups
   - deliveries this week
   - maturities coming up
   - yesterday's response-time stats
2. **Urgent push** (within ~1 min) for: new leads, complaints or upset tone, anything time-sensitive (today, tomorrow, cancel, reschedule), and VIP customers.
3. **Approval queue:** Claude drafts first responses and follow-ups in your voice. You tap approve, edit or reject on iPhone (Claude app or Quo for now; the dashboard later).
4. **Capture Shortcuts** (Action Button / Back Tap / share sheet):
   - Voice note
   - Screenshot / Share
   - Card / Photo
   - Quick note
   - Who-is-this (paste a number and get the customer card)
5. **Ask Claude anything** on any device. For example: "Who did I promise a Tundra quote to last week?" or "Which leads from AutoTrader haven't shown?"
6. **End of shift (automatic):** anything unanswered gets a follow-up task and a drafted message for the morning queue.

---

## 9. Web dashboard and secure login (PLANNED, NOT BUILT)

- **Form:** a responsive PWA installed to the Home Screen on iPhone and iPad, full-screen on the iMac, with web push for urgent alerts.
- **Login (your top priority): passkeys with Face ID / Touch ID.**
  - WebAuthn passkeys synced by iCloud Keychain, with no passwords.
  - Re-check Face ID before **sending**, **exporting** or **viewing full transcripts**.
  - Recovery: a second passkey on the iPad plus printed backup codes.
  - Short sessions, a device list with "sign out everywhere", and an optional Cloudflare Access front door.
- **Screens (later):**
  - Inbox (all channels)
  - Customer timeline
  - Lead pipeline board by stage
  - Approval queue
  - Follow-ups
  - Stats: response time, appointments set/shown, close rate by source
  - Settings/connections
- **Stack:**
  - Next.js PWA
  - Supabase in the **Canada (Central)** region (Postgres + Row-Level Security, Auth with passkeys, Edge Functions)
  - n8n on the iMac
  - a Claude API worker
  - a custom remote MCP server (OAuth) so the Claude apps share the same data
- **Security baseline:**
  - channel tokens server-side only
  - the Claude worker has **no send tool** (prompt-injection defence)
  - encrypted at rest and in transit
  - daily backups
  - an audit log
  - retention: interactions kept 24 months, then summarised and deleted (confirm with dealership policy)

---

## 10. Compliance (Canada: BC/Alberta)

- **CASL:**
  - commercial texts and emails need express or implied consent (an inquiry gives implied consent for 6 months; a purchase gives 2 years)
  - every message identifies you and the dealership and offers an unsubscribe
  - track consent dates per customer, and let the follow-up cadence stop automatically when consent lapses
- **PIPA (BC/Alberta) / PIPEDA:**
  - tell customers about collection, use and AI processing
  - storage outside Canada is allowed with notice, but this plan keeps the hub in Canada anyway
  - vendors (Quo, Meta, Google, Anthropic) process data in the US, so IT must sign off
- **Call recording:**
  - one-party consent is lawful under the Criminal Code
  - privacy law still expects notice, so a Quo greeting says the call "may be recorded for quality and follow-up"
- **Dealership/Toyota policy:**
  - written IT approval checklist (below)
  - DealerSocket remains the system of record
- **IT/management checklist:**
  1. Approval to connect: Quo, Gmail, the Text MCP (LiveChat), dealership Meta pages, and DealerSocket (API or reports).
  2. Can DealerSocket API access be granted (L1)? If not, confirm the lead-intake email address (L2) and scheduled report access (L3).
  3. Sign-off on data flows to US-hosted vendors (Anthropic, Quo, Meta, Google).
  4. Required retention period and deletion rules.
  5. Whether a personal Quo number for customer contact is allowed, and whether the dealership would pay for it.

---

## 11. Tools and costs, three ways

> Approximate monthly prices, mostly **USD**, to be verified. Usage assumes ~150 contacts/day (~4,500/month).

### Option 1: Lean
Your own number plus free and self-hosted tools. Most of the work runs on the iMac.

| Tool | Purpose | Est./mo |
|---|---|---|
| Quo **Starter** (1 user, CA number) | Business calls/SMS, voicemail. *No call transcripts on this plan.* | ~$15–19 |
| n8n self-hosted on iMac + Cloudflare Tunnel | All ingestion and automation | $0 |
| Supabase **Free** (Canada region) | Hub DB, auth | $0 (limits: pauses when idle, 500 MB) |
| whisper.cpp on iMac | Voice-note transcription | $0 |
| Claude API (Haiku for sorting, Sonnet for drafts) | Summaries, urgency, drafts | ~$20–40 |
| Gmail, Text MCP, iMessage plugin | Existing / free | $0 |
| Instagram/Messenger | Direct in Chatwoot self-host on iMac | $0 |
| **Total** | | **≈ $35–60 / month** |

Trade-offs: no call transcripts; Supabase Free isn't meant for production; more for you to look after.

### Option 2: Recommended
A balance of reliability and cost.

| Tool | Purpose | Est./mo |
|---|---|---|
| Quo **Business** (1 user) | Calls + **transcripts & AI summaries**, SMS, official Claude connector | ~$23–33 |
| Supabase **Pro** (Canada Central) | Production hub, backups, passkey auth later | $25 |
| n8n self-hosted on iMac (or Canadian VPS ~$10) | Ingestion | $0–10 |
| Chatwoot self-hosted (iMac/VPS) or Chatwoot Cloud Startups | Instagram/Messenger inbox | $0–19 |
| Claude API | Summaries, urgency, drafts, vision parsing | ~$30–60 |
| Quo SMS usage / call minutes | Included allowances, plus top-ups | ~$0–10 |
| **Total** | | **≈ $80–160 / month** |

### Option 3: Least effort
Paid hubs with official Claude connectors, less self-hosting.

| Tool | Purpose | Est./mo |
|---|---|---|
| Quo **Scale** | Calls, transcripts, summaries, action items | ~$47 |
| respond.io (Growth) | Instagram + Messenger + WhatsApp + more, official MCP | ~$159+ |
| Supabase Pro (Canada) | Hub | $25 |
| n8n Cloud or Make | Hosted automation (no iMac dependency) | ~$20–60 |
| Claude API | As above, higher usage | ~$40–80 |
| **Total** | | **≈ $290–370 / month** |

> **Not included:** your existing Claude plan; WhatsApp per-message fees (not in scope yet); a one-off Mac mini (not needed, since the iMac is on 24/7); and any dealership-paid DealerSocket API or vendor fees.
> **Data-residency caveat:** respond.io, n8n Cloud and Zapier/Airtable are not Canada-hosted, which is one reason Options 1–2 self-host ingestion.
> **Why not Zapier at this volume:** ~4,500 contacts × ~3 steps ≈ 13,500 tasks/month, which costs well over $100/month on Zapier alone.

---

## 12. Roadmap

| Phase | Weeks | Deliverables | Exit test |
|---|---|---|---|
| **0. Approvals & accounts** | 0–1 | IT checklist (§10) signed; Quo second number (Canadian area code); LiveChat transcript forwarding; ask about DealerSocket API / intake email / scheduled reports; **follow-up research: DealerSocket API & ADF/XML specifics** | Written approvals |
| **1. Capture + read** | 1–3 | Supabase (Canada) hub + n8n on iMac; Gmail parsing of Toyota.ca/AutoTrader/CarGurus/LiveChat; Quo webhooks; voice-note and screenshot Shortcuts; Claude connectors (Quo, Gmail, Text MCP, hub MCP) | A test lead from each source shows up under the right customer, with response time measured |
| **2. Brief + urgency + cadence** | 3–5 | 7:30 am email brief; urgent push; day 1/3/7/14/30 follow-ups; Claude skills: `lead-triage`, `reply-drafter` (your voice, CASL-compliant footer), `followup-cadence`, `call-summariser`, `marketplace-screenshot-parser` | One week with zero missed follow-ups |
| **3. DealerSocket sync** | 5–7 | Highest level IT allows: L3 report import + L2 ADF push + L4 copy-assist (L1 if granted) | Hub and DealerSocket agree on stage for 20 sampled leads |
| **4. Social + personal iMessage** | 6–8 | Chatwoot (own IG/FB), dealer pages if admin approves; iMac iMessage allowlist sync; card/photo Shortcut | Messages from IG, FB and iMessage appear in the timeline within 2 min |
| **5. Approved replies** | 8–10 | Draft → approve → send through Quo, Chatwoot and Gmail; Face ID check on send (in the Claude/Shortcut flow) | 50 approved sends, zero wrong-recipient |
| **6. Dashboard** | later | PWA + passkeys per §9 | Security review passed |

---

## 13. Decisions log (interview, 2026-11-14)

| # | Question | Decision |
|---|---|---|
| 1 | Users | Just me |
| 2 | Main channels | Text/iMessage, Facebook/Instagram, LiveChat, phone/email |
| 3 | Current number | Personal iPhone number |
| 4 | Quo | **Second Quo number**, migrate customers gradually |
| 5 | iMac | On 24/7, so it's the local hub |
| 6 | Volume | Busy, ~75–200 contacts/day |
| 7 | Budget | Not fixed; show tiered options with totals (§11) |
| 8 | Reply autonomy | **Draft, I approve all** |
| 9 | Urgent triggers | New leads, complaints, time-sensitive, VIPs |
| 10 | Location/role | Canada, internet sales manager at a Toyota dealership |
| 11 | Brief | Email only, **7:30 am** (shift starts 8:00) |
| 12 | CRM | Unknown; **start with DealerSocket**, plan adapters for others |
| 13 | Permission | AI use approved; CRM/MCP access uncertain, so **export/import fallbacks** required |
| 14 | Province | BC/Alberta |
| 15 | Account ownership | Mix of dealership and personal |
| 16 | Lead sources | Toyota.ca/dealer site, AutoTrader.ca/CarGurus, Facebook Marketplace |
| 17 | Tracking | Sales stage, vehicle, response time, follow-up cadence, delivery & after-sale, lease/loan maturity, "and more" |
| 18 | Informal capture | Voice note, screenshot/share, card/photo, typed quick note |
| 19 | Data location | **Canada only** (hub) |
| 20 | Dashboard priority | Face ID / passkey login |

### Still open
- Exact province (BC or Alberta): minor differences under PIPA.
- Response-time target (default 15 min).
- Days off, so the brief is skipped.
- Which "and more" extras to keep (§7).
- Which cost option (§11).
- Whether the dealership pays for Quo.
