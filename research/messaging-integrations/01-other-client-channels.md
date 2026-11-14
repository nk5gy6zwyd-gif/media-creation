# Other Client Communication Channels (phone, social DMs, reviews, forms, bookings, meetings, CRMs, informal)

> Research date: 2026-11-14 · Researcher: subagent (Sonnet) · Status: draft
> Method: ~12 web searches (snippets, mostly third-party directories/blogs). Few vendor pages were fetched directly, so every "official" claim is flagged "verify". No GitHub API searches or YouTube fetches were done. WebSearch ignores `site:youtube.com`, so the YouTube section is thin. Out of scope (other agents): SMS, Messenger, Instagram, LiveChat, WhatsApp, Gmail, unified-inbox tools.

## 1. TL;DR
- Highest value beyond the known channels: **phone calls/voicemail** (Quo/OpenPhone, Dialpad, Aircall, RingCentral, Ringover), **meeting transcripts** (Fireflies, Fathom, Zoom), **web forms + bookings** (Typeform/Jotform/Calendly via Zapier or hosted MCP), and **reviews** (Google Business Profile).
- Most "real" platforms now have an MCP server (official or community): Quo (official), Dialpad (official, gated early access), Fireflies (official), HubSpot (official), Intercom (official, read-only), Airtable/Notion (official), Slack/Microsoft 365/Zoom (Anthropic connector directory). Zendesk, Aircall, Otter, GoHighLevel and Yelp/Trustpilot rely on community servers or scrapers.
- Truly informal channels (personal iMessage, Signal, Telegram, LinkedIn, X DMs, in-person) need a bridge: **Beeper's local MCP server** or Unipile (paid) cover many at once. Everything else falls back to forwarding/voice memos/manual logging.
- Best architecture: one **client communication log** (Airtable, or HubSpot/Notion if already used) with one row per interaction (client, channel, direction, timestamp, text/transcript, link). Zapier/Make or webhooks feed it. Claude reads and writes it through that tool's MCP server.
- Deep-dive candidates: (a) phone/voicemail/call transcription, (b) the single log/CRM architecture plus Zapier/Make ingestion, (c) Beeper/Unipile bridges for personal messaging.

## 2. Master comparison table
Priority: P1 do first, P2 next, P3 only if the channel matters to this business. "MCP" means what the search turned up; verify before relying on it.

| Channel | API? | MCP? | Capture method | Difficulty | Priority |
|---|---|---|---|---|---|
| Phone calls/voicemail: Quo (OpenPhone) | Yes | Official hosted MCP, included on all plans; transcripts need recording plus Business/Scale plan (per third-party directory) | Official MCP, or Zapier | Easy | P1 |
| Dialpad | Yes | Official hosted MCP, early access via CSM; search/fetch call transcripts. Community server too | Official MCP if granted | Medium | P1 if on Dialpad |
| Aircall | Yes | Community only (not endorsed by Aircall) | Community MCP or webhooks to log | Medium | P2 |
| RingCentral | Yes | Official MCP catalog (transcript coverage unconfirmed) | MCP or API | Medium | P2 |
| Ringover | Yes | MCP, 36 read-only tools incl. transcripts | MCP | Easy | P2 |
| Any Twilio/VoIP number | Yes | No | Voicemail webhook with transcription callback, forward to log | Medium (dev) | P2 |
| Google Voice | No official API | Via Beeper (listed) | Beeper, or email notifications (forward) | Medium | P3 |
| Personal iMessage | No (local DB) | Community macOS servers (imessage-max, jons-mcp-imessage); Beeper | Local MCP reading Messages DB, or Beeper | Medium; privacy-sensitive | P2 |
| Telegram / Signal / LinkedIn / X DMs | Restricted or none | Beeper local MCP (claims all), Unipile MCP (paid; no Signal/X shown) | Beeper or Unipile | Medium | P2/P3 |
| TikTok DMs/comments | Limited | Not found | Manual or notification forwarding | Hard | P3 |
| Google Business Profile reviews | Yes, approval needed (days to weeks) | Community (gbp-review-agent, ReputeMap, MewCP, Posterly) | Community MCP or API | Medium (approval) | P1 for local business |
| Yelp | Fusion API shows only a few review excerpts; reviews tier unclear (verify) | Only Apify actors | Email alerts from Yelp, or Apify | Hard | P3 |
| Facebook reviews | via Meta Graph (other agent) | n/a | See Meta agents | | n/a |
| Trustpilot | Yes with Business account | Apify MCP (third party) | API, or email notifications | Medium | P3 |
| Web forms: Typeform, Jotform | Yes | Hosted MCP listed (directory); Zapier MCP | Webhook/Zapier to log | Easy | P1 |
| Google Forms, WordPress, Webflow forms | Via Sheets/webhook | Via Zapier | Send to Sheet/Airtable or email | Easy | P1 |
| Calendly / Acuity | Yes | Calendly hosted MCP listed (directory); Zapier | MCP, or Zapier into log | Easy | P1 |
| Video calls: Zoom | Yes | Zoom connector in Anthropic directory (search/recap) | Connector | Easy | P1 if used |
| Meeting notetakers: Fireflies | Yes | Official remote MCP (OAuth, read-only in Claude directory) | MCP | Easy | P1 |
| Fathom | Yes | Open-source MCP plus hosted; no search endpoint | MCP | Easy | P1 |
| Otter | Unofficial | Community (stores your password; unofficial API) | Prefer switching to Fireflies/Fathom | Risky | P3 |
| Google Meet / Teams transcripts | Via Workspace/M365 | Granola, tl;dv bridge to Claude; M365 connector | Notetaker app | Medium | P2 |
| Slack (incl. Connect) | Yes | Official connector (send/search/read thread) | Connector | Easy | P2 |
| Discord | Yes | None found in directory results | Bot, or community MCP | Medium | P3 |
| Microsoft Teams / Outlook | Yes (Graph) | Microsoft 365 connector (SharePoint/OneDrive/Outlook/Teams); sources conflict on read-only vs write; needs work account and admin consent | Connector | Medium | P1 if Outlook used |
| HubSpot | Yes | Official (beta, plus remote); many community | MCP | Easy | P1 if CRM |
| Intercom | Yes | Official mcp.intercom.com (conversations/contacts; cannot reply; US-hosted only) | MCP | Easy | P2 |
| Zendesk | Yes | No official (as of Mar 2026 per sources); community | Community MCP | Medium | P2 |
| Freshdesk | Yes | Conflicting (GA vs Enterprise early access); community exists | Verify | Medium | P3 |
| GoHighLevel | Yes | Community (hundreds of tools); "official" claim doubtful | Community MCP | Medium | P2 if used |
| Invoicing/payment notes | Yes (Stripe, QuickBooks, etc.) | Not researched | Zapier | n/a | P3 |
| Voice notes / in-person | No | n/a | Voice memo to Whisper/Whisper Memos to log | Easy | P2 |
| Paper/mail | No | n/a | Phone scan with OCR, email to log | Easy | P3 |

## 3. Phone calls and voicemail
- **Quo (formerly OpenPhone)**: official MCP connecting to Claude/ChatGPT, included on every plan. Two server URLs appeared in sources (`mcp.quo.com/sse` and `/mcp`), so use the one in current Quo docs. Transcripts and summaries need call recording and a Business or Scale plan per a third-party directory. Sources: https://www.quo.com/product/ai/mcp.md, https://www.quo.com/llm-info, https://glama.ai/mcp/servers/EfrainTorres/armavita-quo-mcp/tools/get_call_transcription, https://www.usecarly.com/blog/quo-api/
- **Dialpad**: official hosted MCP, early access (ask your Customer Success Manager). Tools include search-call-transcripts and fetch-call-transcripts. Community server also exists. https://developers.dialpad.com/docs/dialpad-mcp-server, https://glama.ai/mcp/servers/lczgr9ulgp
- **Aircall**: community MCPs only, explicitly not endorsed (one claims 73 tools; one gives read access to calls, transcripts, summaries). https://lobehub.com/mcp/itaygadot111-aircall-mcp-server, https://lobehub.com/mcp/themobilefirstco-aircall-mcp-server
- **RingCentral**: MCP catalog with 4 servers and 30+ tools. https://mcp.labs.ringcentral.com/docs/
- **Ringover**: MCP with 36 read-only tools incl. transcripts. https://www.ringover.com/blog/mcp-phone
- **Generic VoIP/Twilio**: a recording's `transcribeCallback` fires a webhook with the transcript; forward to email/Slack or log. Failed webhooks can lose the transcript unless a conversation configuration is set. https://www.twilio.com/en-us/blog/transcribe-voice-message-python-django, https://blog.deepgram.com/email-phone-call-transcripts-with-twilio-studio-and-pipedream, https://www.twilio.com/docs/voice/api/batch-transcription-resource
- Recommendation: if the business already uses one of these, enable its MCP. If it uses a personal mobile number for client calls, moving to a business line (Quo is the cheapest-to-connect) is the single biggest capture win. Recording consent laws apply (see section 11).

## 4. Meetings and video calls
- **Fireflies**: official remote MCP `https://api.fireflies.ai/mcp` (OAuth or API key); transcripts, summaries, action items; verified read-only listing in Claude directory. https://docs.fireflies.ai/getting-started/mcp-configuration
- **Fathom**: open-source `agencyenterprise/fathom-mcp-server` and hosted versions; no search endpoint, so filtering is client-side. https://github.com/agencyenterprise/fathom-mcp-server
- **Otter**: community `otter-mcp` on an unofficial API, stores email/password in config. Avoid. https://glama.ai/mcp/servers/DarrenZal/otter-mcp
- **Zoom**: connector in Anthropic's directory (search/recap meetings). https://claude.com/connectors (listed via tldv/granola guides below)
- Google Meet/Teams: Granola and tl;dv pull transcripts into Claude. https://www.granola.ai/blog/best-claude-connectors-for-work-in-2026-and-how-to-add-them, https://tldv.io/blog/best-claude-connectors/
- Roundups: https://fellow.ai/blog/ai-meeting-notetaker-claude-connector/, https://sonix.ai/resources/transcription-mcp-servers-meeting-notes/, https://speakapp.com/blog/best-meetings-mcp

## 5. Social, reviews, community
- **Google Business Profile reviews**: GBP API needs approval (verified profile, active 60+ days, access request; days to weeks). Q&A API shut down Nov 2025. Community MCPs: `satheeshds/gbp-review-agent` (read, draft and post replies; OAuth), ReputeMap, MewCP, Posterly. https://github.com/satheeshds/gbp-review-agent, https://slashpost.ai/blogs/google-business-profile/google-business-profile-api-documentation-2026, https://developers.google.com/my-business/content/implement-oauth, https://mcpservers.org/servers/reputemap/mcp, https://mcpservers.org/th/servers/asthetech/mewcp-google-business, https://www.poster.ly/connectors/google-business-mcp
- **Yelp**: Fusion API exposes only a few review excerpts; scrapers (Apify) used instead, with ToS risk. Practical capture: Yelp's own email notifications forwarded to the log. https://scrapfly.io/blog/guide-to-yelp-api/, https://apify.com/renzomacar/yelp-reviews.md
- **Trustpilot**: official API needs Business account/Connect module; third parties scrape. https://documentation-apidocumentation.trustpilot.com/introduction/, https://apify.com/onetapstudio/review-reputation-mcp (covers Google, Yelp, Trustpilot; ~$5/1,000 reviews)
- **LinkedIn / X / Telegram / Signal DMs**: see bridges in section 7. LinkedIn and X have restrictive official DM APIs; Unipile (paid) provides LinkedIn messaging via its own layer. https://glama.ai/mcp/servers/dkqt2x71qs, https://www.trykondo.com/docs/imessage-mcp-server (Kondo is a commercial DM inbox, iMessage MCP "coming soon")
- **TikTok, Discord**: no MCP found. Treat as manual or notification-forwarded.
- **Slack / Teams**: official connectors cover Slack and Microsoft 365. https://claude.com/marketplace/connectors/slack, https://www.fast.io/resources/claude-integrations-guide.md

## 6. Forms, bookings, CRMs and helpdesks
- **Zapier MCP** (`mcp.zapier.com`, personal URL from zapier.com/mcp) exposes 5,000+ app actions to Claude, including Typeform, Jotform, Calendly. https://mcpservers.org/servers/zapier/mcp, https://zapier.com/mcp/jotform
- Directory-listed hosted MCPs: Typeform (OAuth, streamable HTTP), Jotform, Calendly (API v2). Directory only; verify with vendors. https://contextbolt.com/tools/mcp-server-directory/zapier/
- **HubSpot**: official MCP (developers.hubspot.com/mcp), plus community. https://www.fast.io/resources/best-mcp-servers-crm/, https://obot.ai/mcp-catalog/hubspot/
- **Intercom**: official `mcp.intercom.com`, read-only on conversations, US-hosted workspaces only. **Zendesk**: no official; `reminia/zendesk-mcp-server` community. **Freshdesk**: conflicting reports. **GoHighLevel**: community `mastanley13/GoHighLevel-MCP`. https://dev.to/grove_chatforest/customer-support-helpdesk-mcp-servers-zendesk-intercom-freshdesk-servicenow-plain-and-more-31lh, https://toolradar.com/blog/best-mcp-servers-customer-support, https://glama.ai/mcp/servers/@mastanley13/GoHighLevel-MCP, https://jc-aiwiki.pages.dev/gohighlevel/mcp-integration
- Small-business CRMs with Claude connectors: HoneyBook, Daylite, Capsule, Baserow. https://www.artificialintelligence-news.com/news/honeybook-bets-agentic-ai-small-business-operations-claude-connector/, https://capsulecrm.com/blog/what-is-mcp-in-crm/, https://baserow.io/blog/build-crm-with-claude

## 7. Informal / no-API channels
| Approach | How | Notes |
|---|---|---|
| Beeper local API + MCP | Beeper desktop app bridges WhatsApp, Instagram, Telegram, Signal, Messenger, X, Google Messages/Voice/Chat, LinkedIn, Discord, Slack; runs on-device (Settings > Developers); experimental beta | Best single bridge for personal accounts. Signal and similar run through Matrix bridges. Check its limits and ToS. https://www.beeper.com/mcp, https://www.beeper.com/desktop-api, https://developers.beeper.com/open-source |
| Community Beeper MCP | `mimen/beeper-mcp` read-only on Beeper's macOS DB | Search/read history only. https://glama.ai/mcp/servers/@mimen/beeper-mcp/blob/008fabf16ec086029ff70db10893470c538dd486/README.md |
| Unipile MCP | Paid API for WhatsApp, LinkedIn, Instagram, Messenger, Telegram | https://mcpservers.org/servers/baryhuang/mcp-unipile |
| iMessage MCPs | Read macOS Messages database; need Full Disk Access | Reads private data; vet code. https://lobehub.com/mcp/robdezendorf-imessage-max, https://lobehub.com/es/mcp/jonmmease-jons-mcp-imessage |
| Forward to email | Forward screenshot/text to a dedicated address (e.g. log@yourdomain), parse with Zapier Email Parser/Gmail MCP | Lowest tech; works for anything |
| Screenshots + OCR | Claude reads the image directly (vision), no OCR step needed | Manual per message |
| Phone backup/export | WhatsApp "export chat", iMessage DB, Android SMS backup XML | Periodic bulk loads, not real time |
| Voice memos | Record after a conversation, transcribe (Whisper, Whisper Memos app), file into log | Covers in-person. https://github.com/michaelgriscom/voice-memo-transcription, https://whispermemos.com/ |
| Notification mirroring | Android notification forwarders, or Beeper | Not researched in depth |
| Manual logging | A 10-second form (Typeform/Airtable form) after each interaction | Reliable and cheap, needs discipline |

## 8. Single client communication log: architecture
1. **Store**: one table `interactions` with columns: id, client (linked to a `clients` table), channel, direction (in/out), timestamp, summary, full_text/transcript, source_url, status (needs-reply/done), owner, sentiment, raw_payload. Choose one of: **Airtable** (official MCP at `mcp.airtable.com/mcp`, OAuth or PAT), **Notion** (official hosted MCP), **HubSpot** (official MCP) or **Baserow**. Airtable fits best for a small business: a service business describes one record per client with a full timeline, fed by Zapier. https://www.mcpbundles.com/skills/airtable-mcp, https://hub.docker.com/mcp/server/notion/config, https://aimaker.substack.com/p/how-i-finally-turned-ai-into-managing-actual-personal-operating-system-workflow-mcp-model-context-protocol-guide-claude, https://baserow.io/blog/build-crm-with-claude
2. **Ingest**: platforms with their own MCP/connector are read live by Claude. Others write rows via Zapier/Make: new Typeform submission, Calendly booking, Quo/Aircall call-with-transcript, Fireflies meeting, GBP review, forwarded email.
3. **Normalise**: a Claude scheduled task or Zap step summarises each new item and tags client, intent and urgency.
4. **Use**: Claude reads the log through the Airtable/Notion MCP for daily digests, "who is waiting on a reply", and drafts replies. Drafts go to the human before sending until trusted.
5. Zapier + Airtable + Claude tutorial: https://www.xray.tech/post/zapier-mcp-claude ; WhatsApp-to-CRM recipe: https://vinkius.com/mcp-recipes/turn-whatsapp-chats-into-crm-data-via-mcp ; small-business user report: https://community.getjobber.com/discussions/operations-forum/i-built-an-ai-assistant-that-runs-my-office/10573/replies/10689 ; overview: https://mcp-find-web.vercel.app/blog/mcp-for-small-business-owners

## 9. YouTube / tutorials
The search tool did not return YouTube URLs (it ignores `site:`). Only two video-summary pages surfaced: https://www.recall.it/summary/artificial-intelligence/make-claude-the-control-panel-for-your-productivity-stack (Claude as control panel across tools) and the Zapier tutorial above mentioning a video. Gap: retry with a different tool.

## 10. Ranked priorities
1. Phone calls and voicemail onto a recordable business line (Quo, Dialpad or Aircall), with the vendor MCP.
2. Web forms and booking tools (Typeform/Jotform/Calendly/website forms) into the log via Zapier.
3. The log itself (Airtable or the existing CRM) plus Claude MCP access.
4. Meeting transcripts (Fireflies or Fathom, Zoom connector).
5. Google Business Profile reviews (start the API access request early).
6. CRM/helpdesk already in use (HubSpot, Intercom, Zendesk, GoHighLevel).
7. Personal messaging bridge via Beeper (iMessage, Telegram, Signal, LinkedIn, X).
8. Voice-memo capture for in-person, forwarding for paper/mail.
9. Yelp, Trustpilot, TikTok, Discord.

## 11. Deep-dive flags
- **Phone/voicemail/call transcription**: recommended; likely the largest untracked channel.
- **Client communication log + ingestion (Zapier/Make/webhooks)**: recommended; it ties every channel together.
- **Beeper/Unipile bridges**: recommended if the owner uses personal accounts (Telegram, Signal, LinkedIn, iMessage).
- Reviews (Google Business Profile) could be a smaller follow-up.

## 12. Gotchas and open questions
- Call-recording consent laws (two-party consent states/countries) and message privacy: personal-account bridges (Beeper, iMessage DB readers) read private third-party data. Check ToS, since unofficial bridges can violate platform terms.
- Claims of "official" for Freshdesk, GoHighLevel, Microsoft 365 write access, and the Quo URL were conflicting. Verify against vendor docs.
- Transcripts often require paid plans and recording turned on.
- Open: which phone/CRM/forms/booking tools does the business actually use? Which channels have the most volume? Is a Mac available for iMessage/Beeper? The answers decide the build order.
- Not researched: invoicing tools, Voxer, GitHub repo stars/last-commit dates, the official MCP registry and smithery directly.

## 13. Sources (consulted via search results; most not fetched directly)
Quo https://www.quo.com/product/ai/mcp.md · Dialpad https://developers.dialpad.com/docs/dialpad-mcp-server · RingCentral https://mcp.labs.ringcentral.com/docs/ · Ringover https://www.ringover.com/blog/mcp-phone · Fireflies https://docs.fireflies.ai/getting-started/mcp-configuration · Fathom https://github.com/agencyenterprise/fathom-mcp-server · Beeper https://www.beeper.com/mcp · Unipile https://glama.ai/mcp/servers/dkqt2x71qs · GBP https://github.com/satheeshds/gbp-review-agent · Google OAuth https://developers.google.com/my-business/content/implement-oauth · Trustpilot https://documentation-apidocumentation.trustpilot.com/introduction/ · Zapier MCP https://mcpservers.org/servers/zapier/mcp · Helpdesk roundup https://dev.to/grove_chatforest/customer-support-helpdesk-mcp-servers-zendesk-intercom-freshdesk-servicenow-plain-and-more-31lh · Airtable https://www.mcpbundles.com/skills/airtable-mcp · Twilio https://www.twilio.com/docs/voice/api/batch-transcription-resource · Claude connectors https://claude.com/connectors plus the other URLs cited inline above.
