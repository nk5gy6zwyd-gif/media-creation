# Unified inbox hubs, automation glue and email

> Research date: 2026-11-14 · Researcher: subagent (Sonnet) · Status: draft
> Confidence note: most facts come from WebSearch result summaries and a few fetched pages. Vendor and aggregator pricing is flagged where unverified. Re-check pricing pages before buying.

## 1. TL;DR
- Yes, every channel can be funnelled into one hub and exposed to Claude. The hub approach beats nine separate integrations because Meta (Messenger, Instagram) and WhatsApp app review/BSP onboarding is done once, inside the hub.
- Best hosted MCP-ready hubs for a small business: **respond.io** (official hosted MCP, OAuth), **Front** (official MCP, in Claude connector directory, can send), **Wati** (WhatsApp + Instagram + Messenger MCP), **Missive** (cheap, many channels, community MCP only), **Chatwoot** (open source, community MCPs only).
- Chatwoot is the best open-source/self-host hub. Its MCP servers are all small community projects (6-12 GitHub stars each), so budget for an n8n/Pipedream wrapper or a thin custom server over its REST API.
- Email is already solved. The Gmail connector is attached in this user's Claude session (search, read, labels, drafts, reply, send). Use email as a catch-all sink by forwarding LiveChat transcripts and platform notifications into a Gmail label.
- Automation glue (n8n, Make, Zapier, Pipedream, Activepieces, Composio) is the fallback for any channel whose hub lacks MCP. n8n and Activepieces are self-hostable. Zapier MCP is metered (2 tasks per successful call).
- Main blockers: Meta/WhatsApp business verification and 24-hour reply window (see section 9), and pricing/MCP maturity differences across vendors.

## 2. How messages arrive today
Per channel (SMS, Messenger, Instagram DM, WhatsApp, LiveChat chat + transcript emails, email) messages live in separate vendor apps. A hub connects each channel once (Meta OAuth for Messenger/Instagram, WhatsApp Cloud API or BSP, Twilio or similar for SMS, a website widget or LiveChat integration, IMAP/OAuth for email) and shows one conversation list. Claude then talks to the hub (MCP/API/webhooks) rather than to each channel.

## 3. Platform x channel support matrix
Legend: Y = native, P = via partner/connector or API channel, - = not offered/unconfirmed. "MCP" column: O = official, C = community only, - = none found. Channel entries are from vendor pages/search summaries, not hands-on tests.

| Platform | SMS | Messenger | Instagram | WhatsApp | Live chat (own widget) | Email | MCP | Self-host | Small-biz price (verify) |
|---|---|---|---|---|---|---|---|---|---|
| Chatwoot | Y (Twilio etc.) | Y | Y | Y | Y | Y | C (many) | Yes (MIT core) | Cloud $0 (2 agents, live chat only), $19, $39, $99 per agent/mo; all channels from $19 |
| respond.io | Y | Y | Y | Y | Y | Y | O (hosted, OAuth) | No | Higher tier; verify. Workflows need Growth+ |
| Front | Y | Y | Y | Y | Y | Y | O (27 tools claimed by a competitor blog) | No | Per seat, verify |
| Missive | Y (Twilio/SignalWire/Dialpad) | Y | Y | Y | Y | Y | C (several, 0-4 stars) | No | About $18 Starter, $30 Productive (API), $45 Business per user/mo (Capterra, unverified) |
| Trengo | Y | Y | Y | Y | Y | Y | - found | No | Reported EUR 299/mo Boost (no API), EUR 499+/mo Pro (API); verify |
| Kommo | P | Y | Y | Y | Y | Y | C (Miguelgbastos 16 stars) | No | verify |
| GoHighLevel | Y | Y | Y | Y | Y (chat widget) | Y | O (OAuth) + C (127-196 star community) | No | $97 / $297 / $497 per mo |
| Twilio (Conversations/Flex) | Y | P | P | Y | P (webchat) | P | O (alpha, @twilio-alpha) | No | Pay per message plus platform fees |
| Unipile | - | soon | Y | Y | - | Y | O (unipile/unipile-mcp) + C | No | About EUR 5 per connected account/mo, EUR 49 min (aggregators disagree) |
| Wati | - | Y | Y | Y | - | - | O (hosted, OAuth) | No | verify |
| Intercom | P | P | P | Y | Y | Y | O (mcp.intercom.com/mcp) | No | Per seat, expensive for small shops |
| HubSpot inbox | P | Y | P | Y | Y | Y | O (mcp.hubspot.com, GA Apr 2026) | No | Free CRM tier has chat/inbox basics; verify |
| Zendesk / Sunshine | P | Y | Y | Y | Y | Y | C (no official server confirmed) | No | verify |
| Beeper | - | Y | Y | Y | - | - | O (local Desktop API MCP, experimental) | Local app | Personal use only |
| Superchat, Freshchat | not researched in depth | | | | | | | | |

## 4. Platform notes (API, webhooks, MCP, maturity)

**Chatwoot (recommended open-source hub).** Cloud plans: Hacker $0 (2 agents, 500 conversations/mo, live chat only), Startups $19, Business $39, Enterprise $99 per agent/month; Captain AI credits $20 per 1,000 [1]. Self-host is free (MIT core). REST API and webhooks (message_created, conversation_status_changed, etc.) are well documented upstream (not re-fetched here). MCP landscape (all community): fazer-ai/mcp-chatwoot (TypeScript+Bun, 12 stars, claims 129 tools, pushed 2026-03), hugoblanc/chatwoot-mcp (10 stars, 2026-03), StackLab-Digital/chatwoot_mcp (11 stars, stale since 2025-07), serversmx/mcp-chatwoot (6 stars, 2026-05), mohnori/chatwoot-mcp (Glama listed) [2][3][4]. Official registry has dev.cotlet/chatwoot, io.github.askmojood/chatwoot-mcp, io.usefulapi/chatwoot [5]. Maturity: low (hobby-level), but the API is simple so risk is moderate. Read the source before giving it an API token.

**respond.io.** Official hosted MCP at `https://mcp.respond.io/mcp`, OAuth, scoped to your role, one workspace per connection. Tools: contacts, send messages/templates, conversations (assign/close/comment), AI agents, workflows (Growth plan+), workspace settings. MCP calls do not consume AI credits. A self-hosted open-source server (github.com/respond-io/mcp-server, npx `@respond-io/mcp-server`, API token) also exists [6][7][8]. Page does not enumerate channels. Strongest "just works with Claude" hub found.

**Front.** Official MCP, available in the Claude connector directory, or custom via developer app at `mcp.frontapp.com/mcp`; admin caps read/write/send scopes [9]. A competitor blog (Drag) claims 27 tools and send capability, treat as unverified [10].

**Missive.** Channels: SMS (Twilio/SignalWire/Dialpad), WhatsApp Business, live chat, Messenger, Instagram, custom API channels [11][12]. API on Productive tier and above. Community MCPs only: grncdr/missive-mcp (remote, 4 stars), kanjidoc/missive-mcp (36 tools, pushed 2026-11-06), pranavgade20/missive-mcp. Pricing sources conflict.

**Trengo.** Channels cover email, WhatsApp, Messenger, Instagram, live chat, SMS; API reportedly only on Pro (~EUR 499/mo annual); no MCP found [13]. Too costly for a small shop.

**Kommo.** Community MCP only (Miguelgbastos/Kommo-MCP, 16 stars). Not researched beyond that.

**GoHighLevel.** Official OAuth MCP (marketplace.gohighlevel.com/docs/other/mcp per a third-party guide) plus big community servers (BusyBee3333 Go-High-Level-MCP-2026, 927 tools, 127 stars; mastanley13, 196 stars). Plans $97/$297/$497 [14]. Good if the business also wants CRM/pipelines/booking; overkill otherwise.

**Twilio.** Official alpha MCP (`@twilio-alpha/mcp`) exposing about 1,800 APIs including Conversations; use `--services/--tags` filters [15][16]. Still alpha. Best for SMS/WhatsApp send and message-log reading, not a UI inbox.

**Unipile.** Unified messaging API priced per connected account (about EUR 5/account/mo; EUR 49/mo minimum per one aggregator) [17]. Official MCP `com.unipile/unipile-mcp` in the MCP registry (v1.0.0, repo unipile/unipile-mcp, 2 stars) [18], channels listed: LinkedIn, WhatsApp, Instagram, Telegram, email; Messenger "coming soon" per one source. Tutorial with n8n exists [19]. Note: it connects personal/unofficial sessions for some channels (ToS risk, verify), unlike Meta-approved Cloud API.

**Wati.** Official hosted MCP `mcp.wati.io/mcp` (EU: `eu-mcp.wati.io/mcp`), OAuth, covers WhatsApp plus Instagram DM and Messenger text replies; about 58 tools per a directory [20][21]. Instagram comment tools rolling out.

**Intercom.** Official remote MCP `https://mcp.intercom.com/mcp` (OAuth, US-hosted workspaces) [22]; sources disagree on write scope. **HubSpot.** Official remote MCP, inbox/threads readable subject to inbox permissions; a community hubspot-conversations-mcp adds reply tools [23]. **Zendesk/Sunshine.** No confirmed official server; community servers only [24]. **Freshdesk/Freshchat, Superchat, Sunshine Conversations:** not covered (budget); treat as API-only until checked.

**Beeper.** Official local Desktop API plus built-in MCP (Settings > Developers, localhost:23373, Beeper Desktop 4.1.169+), experimental and "personal use" [25]. Bridges WhatsApp, Instagram, Messenger. Unofficial Matrix route (jkobject/unbridled) breaks easily. Not suitable as a business system of record, but a fast way for the owner to read their own chats.

## 5. MCP servers (hub-level and glue)
| Name | Repo / URL | Official? | Read/Write | Stars / last commit | Auth | Notes |
|---|---|---|---|---|---|---|
| respond.io MCP | https://mcp.respond.io/mcp ; github.com/respond-io/mcp-server | Yes | R/W | n/a | OAuth / API token | Best hosted hub MCP |
| Front MCP | mcp.frontapp.com/mcp | Yes | R/W/send | n/a | OAuth | In Claude connector directory |
| Wati MCP | mcp.wati.io/mcp | Yes | R/W | n/a | OAuth | WhatsApp+IG+Messenger |
| Intercom MCP | mcp.intercom.com/mcp | Yes | R (+notes) | n/a | OAuth | US only |
| HubSpot MCP | mcp.hubspot.com | Yes | R mostly | n/a | OAuth | GA 2026-04 |
| GoHighLevel MCP | marketplace.gohighlevel.com/docs/other/mcp | Yes | R/W | n/a | OAuth | plus community 127-196 star servers |
| Twilio MCP | npm @twilio-alpha/mcp | Yes (alpha) | R/W | n/a | API key | Filter services |
| Unipile MCP | github.com/unipile/unipile-mcp | Yes | R/W | 2 stars, 2026-10-05 | API key | Registry-listed |
| fazer-ai/mcp-chatwoot | github.com/fazer-ai/mcp-chatwoot | No | R/W | 12, 2026-03-09 | API token | 129 tools claimed |
| hugoblanc/chatwoot-mcp | github.com/hugoblanc/chatwoot-mcp | No | R/W | 10, 2026-03-24 | API token | |
| kanjidoc/missive-mcp | github.com/kanjidoc/missive-mcp | No | R/W | 1, 2026-11-06 | PAT | 36 tools |
| grncdr/missive-mcp | github.com/grncdr/missive-mcp | No | R/W | 4, 2026-03-06 | | remote |
| Wh1isper/mcp-email-server | github.com/Wh1isper/mcp-email-server | No | R/W | 394, 2026-11-09 | IMAP/SMTP | multi-account |
| codefuturist/email-mcp | github.com/codefuturist/email-mcp | No | R/W | 159, 2026-11-09 | IMAP/SMTP | |
| nikolausm/imap-mcp-server | github.com/nikolausm/imap-mcp-server | No | R/W | 130, 2026-11-13 | IMAP/SMTP | |
| Softeria/ms-365-mcp-server | npm / glama | No | R/W | n/a | Graph OAuth | Outlook, OneDrive, Teams |
| LiveChat/Text MCP | https://mcp.text.com/ ; github.com/livechat | Yes | R | n/a | OAuth | Search archives, transcripts, tags; Text App required [26][27] |
| Beeper Desktop MCP | localhost:23373 | Yes | R/W | n/a | local token | experimental |
Directory presence: Glama and mcpservers.org list Chatwoot, GoHighLevel, Twilio, Outlook servers; the official registry (registry.modelcontextprotocol.io) lists the Chatwoot and Unipile entries above (a second registry query returned HTTP 502). mcp.so, smithery.ai (WATI listed) and pulsemcp were not checked individually beyond search hits. Star counts from GitHub search on 2026-11-14.

## 6. Claude skills & agent tooling
- Community inbox-triage skills (all third-party, none official): agricidaniel/claude-email "email-check" (scores 0-100, categorises, drafts replies), himalaya-mcp triage skill, alirezarezvani inbox-triage (draft-only), onewave-ai cowork-inbox-triage (scheduled-task friendly), pedrohcgs triage-inbox [28]. Common pattern: classify, draft, human approves, never auto-send.
- No official Anthropic Agent SDK inbox example was found. A custom skill can be small: a SKILL.md with triage rules (urgent/sales/support/spam), reply tone guide, and a rule "draft only, never send without approval", wired to the Gmail connector plus a hub MCP.
- Claude scheduled tasks can run a daily digest ("summarise all conversations since yesterday per channel, list unanswered > 4h").

## 7. Notable GitHub projects (non-MCP)
- czlonkowski/n8n-mcp (23k stars, lets Claude build/operate n8n workflows), nerding-io/n8n-nodes-mcp (3k, n8n MCP client node), leonardsellem/n8n-mcp-server (1.6k) (GitHub search 2026-11-14).
- aldinokemal/go-whatsapp-web-multidevice (5k stars, REST+MCP, Chatwoot bridge; unofficial WhatsApp Web, ToS risk).
- melgarafael/DeskcommCRM (5.4k, self-hosted CRM with AI agents + WhatsApp).
- saloprj/dialogbrain-claude-plugin (0 stars, Claude Code plugin: WhatsApp, Telegram, email, website chat; treat as unvetted).

## 8. Automation glue: how each relays channels to Claude
| Tool | Role | Notes |
|---|---|---|
| n8n | Triggers (WhatsApp Trigger node, webhooks, IMAP, Chatwoot/LiveChat webhooks) feed an AI Agent node (Claude model). Also "MCP Server Trigger" exposes workflows as tools to Claude; MCP Client node calls external MCP [29]. | Self-host free (fair-code), cloud paid. Best flexibility. |
| Make | Official hosted MCP at mcp.make.com exposes scenarios as tools (OAuth or token; 25 s limit on OAuth scenario-run tools) [30]. | Good for scheduled/pull scenarios. |
| Zapier MCP | Zapier actions as MCP tools. No separate price; each successful call = 2 tasks, free tier 100 tasks/mo; since June 2026 one shared task pool [31]. | Zapier LiveChat app can capture transcripts on chat end and push to Front/email/webhook [26]. |
| Pipedream | Remote MCP for many apps (claim not verified in this research). | Dev-friendly, generous free tier (unverified). |
| Activepieces | MIT open source, MCP server on all plans incl. self-hosted; vendor claims 400-760 apps [32]. | Chatwoot piece exists [32]. |
| Composio | Hosted toolkits incl. Respond.io, LiveChat, Unipile [33][34]. | Pro about $29/mo per one comparison (unverified). |
Pattern: webhook in (platform event) -> glue writes to a Gmail label, Google Sheet/Airtable, or a hub -> Claude reads via Gmail connector or MCP. For real-time auto-reply, glue calls the Claude API directly; for human-in-the-loop, Claude drafts only.

## 9. Email
- **Gmail:** already connected in this user's Claude session (tools seen in the environment: search_threads, get_thread, create_draft, reply, send_message, label/unlabel, forward). This is the lowest-effort channel and the best catch-all sink. Not exercised in this research (instructions said not to use Gmail tools).
- **Catch-all pattern:** create Gmail labels per source (e.g. `Inbox/LiveChat`, `Inbox/Meta-notifications`, `Inbox/Transcripts`). LiveChat can email chat transcripts and Meta/WhatsApp tools send email notifications; add Gmail filters to auto-label them. Claude then summarises each label daily and drafts replies. Parsing: transcript emails are templated, so Claude can extract name, contact info, question and sentiment directly; no custom parser needed, but a stable subject prefix helps.
- **Google Workspace:** same Gmail API via Workspace account. Shared mailbox via Google Group/Collaborative Inbox (verify delegation works with the connector).
- **Outlook/M365:** a Claude-hosted Microsoft 365 connector (`microsoft365.mcp.claude.com/mcp`) is reported by one directory site with search-oriented tools; verify in the Claude connector directory [35]. Community: Softeria/ms-365-mcp-server (Graph API), Norcim133/OutlookMCPServer, hosted PortEden (draft-only mode).
- **Generic IMAP:** Wh1isper/mcp-email-server (394 stars), codefuturist/email-mcp (159), nikolausm/imap-mcp-server (130). Use app passwords; prefer read + draft over send.
- **Gotchas:** prompt injection via email bodies (restrict send permission; require approval), Gmail "restricted scope" issues apply only to custom apps, not the built-in connector.

## 10. YouTube / tutorials / community
| Title | Channel / Author | URL | What it shows |
|---|---|---|---|
| Unipile + n8n Tutorial | n/a | https://www.youtube.com/watch?v=QjdbdrU6q-k | Unified inbox (email, SMS, LinkedIn, WhatsApp) automation via Unipile in n8n |
| How to Connect WhatsApp to Claude? | n/a | https://www.youtube.com/watch?v=IWG-hDTYLeY | Uses inbox-mcp.com service (promotes Breakcold) |
| Meta MCP Just Made WhatsApp API Setup Way Easier | n/a | https://www.youtube.com/watch?v=uHR8Lex_gKY | Claude + Meta WhatsApp Business MCP, n8n send/receive |
| How to Connect Claude to n8n Using MCP | n/a | https://www.youtube.com/watch?v=Bd_Hpk7h-BM | Beginner n8n MCP |
| Claude + n8n MCP Tutorial (2026) | n/a | https://www.youtube.com/watch?v=ct2ZTHNvnYo | Build workflows from prompts |
| Ultimate Beginner's Guide: Claude MCP WhatsApp Automation | n/a | https://www.youtube.com/watch?v=o27IR4adHeM | No-code WhatsApp |
Titles/URLs are from search results only, not watched. No Chatwoot-MCP YouTube video found (`site:youtube.com Chatwoot MCP` returned none).

## 11. Compliance, limits & gotchas
- Meta platforms: Messenger/Instagram need a Meta app with approved permissions or a hub that already has app approval; Instagram requires a professional account linked to a Facebook Page. WhatsApp Cloud API requires business verification; free-form replies only within 24 h of the customer's last message, otherwise approved templates. (General Meta policy knowledge; confirm in channel-specific reports.)
- Unofficial WhatsApp Web bridges and Unipile personal-session style connections risk account bans.
- GDPR/consent: transcripts and DMs contain personal data; check DPA with the hub and with Anthropic data handling for the plan used.
- MCP safety: give the MCP token the least scope (read-only first), draft-only replies, log approvals.
- Pricing data is volatile (Zapier changed metering June 2026; Missive/Trengo sources conflict).

## 12. Recommended architectures

**A. Budget no-code (about $30-80/mo).** Gmail connector (already connected) + Zapier or Make forwarding LiveChat transcripts and notifications to a Gmail label + respond.io or Missive Starter only if multi-channel replies are needed. Claude reads the Gmail labels and drafts replies. Cost: Zapier Professional about $20-30/mo (MCP calls cost 2 tasks each, 750 tasks) [31] or Make Core; Missive about $18/user; LiveChat existing. Weak: replies to Meta/WhatsApp still done in native apps.

**B. Open-source self-host (about $10-40/mo infra, plus WhatsApp conversation fees).** Chatwoot self-hosted (VPS $10-20/mo, free software) with channels: website widget, email, Messenger, Instagram, WhatsApp Cloud API, SMS via Twilio. n8n self-hosted receives Chatwoot webhooks, calls Claude API for classification/draft (as private notes), and exposes a "Chatwoot tools" MCP server to Claude via a community Chatwoot MCP or n8n MCP trigger. Gmail connector for email sink. Highest control, highest setup effort; Chatwoot MCPs are low-maturity so prefer n8n-wrapped REST calls. Alternatively Chatwoot Cloud Startups $19/agent.

**C. Premium SaaS (about $150-500+/mo).** respond.io (official hosted MCP, OAuth, all channels) or Front (official MCP incl. send) as the hub; Claude connects via the official connector; Gmail/M365 connector for email. Least maintenance, best vendor support, highest cost. GoHighLevel ($97-297) fits if CRM/booking is also needed. Verify respond.io/Front per-seat pricing, which was not retrieved.

Suggested path: start with A this week (zero risk), pilot B or C on one channel (WhatsApp) before committing.

## 13. Open questions
- Exact respond.io, Front, Wati, Kommo, Superchat, Freshchat pricing and which channels each supports at the entry tier.
- Whether LiveChat (Text) transcripts can be emailed natively or need Zapier/webhook; Text MCP availability on the user's plan.
- Pipedream MCP and Composio specifics were not verified.
- mcp.so, smithery and pulsemcp were not directly browsed.
- Official Anthropic inbox-triage Agent SDK example: none found.

## 14. Sources
1. https://www.chatwoot.com/pricing
2. https://glama.ai/mcp/servers/mohnori/chatwoot-mcp
3. https://www.pkgstats.com/pkg:@fazer-ai/mcp-chatwoot
4. https://lobehub.com/mcp/hugoblanc-chatwoot-mcp
5. https://registry.modelcontextprotocol.io/v0/servers?search=chatwoot&limit=10 and ?search=unipile (fetched)
6. https://respond.io/help/integrations/model-context-protocol-mcp.md
7. https://github.com/respond-io/mcp-server
8. https://roadmap.respond.io/changelog/connect-ai-tools-faster-with-our-new-mcp-server
9. https://help.front.com/en/articles/4883136
10. https://www.dragapp.com/blog/shared-inbox-mcp/
11. https://missiveapp.com/features/whatsapp-twitter-facebook-and-sms-accounts
12. https://www.capterra.com/p/152884/Missive/pricing/ ; https://www.dragapp.com/blog/missive-pricing/
13. https://www.trengo.com/pricing ; https://www.eesel.ai/blog/trengo-pricing
14. https://revenuegeeks.com/guides/gohighlevel-mcp ; https://claudemarketplaces.com/mcp/mastanley13/gohighlevel-mcp
15. https://twilio.com/en-us/blog/developers/introducing-twilio-mcp-skills
16. https://chatforest.com/reviews/twilio-mcp-server/
17. https://www.unipile.com/pricing/ ; https://developer.unipile.com/v2.0/docs/billing
18. https://github.com/unipile/unipile-mcp
19. https://www.youtube.com/watch?v=QjdbdrU6q-k
20. https://www.wati.io/developer/whatsapp-mcp
21. https://support.wati.io/en/articles/15899958-how-to-manage-your-wati-workspace-using-the-wati-mcp-server
22. https://www.intercom.com/integrations/connectors/intercom-mcp-server
23. https://developers.hubspot.com/docs/apps/developer-platform/build-apps/integrate-with-the-remote-hubspot-mcp-server
24. https://github.com/michaelrice/zendesk-mcp ; https://tldv.io/de/blog/zendesk-mcp/
25. https://developers.beeper.com/desktop-api/ ; https://github.com/jkobject/unbridled
26. https://www.chatbot.com/help/connect-chatgpt-or-claude-to-your-chats-with-mcp/
27. https://github.com/livechat ; https://marketplace.accounts.livechat.com/apps/zapier/
28. https://takusaotome.github.io/claude-skills-library/en/skills/meta/email-triage-responder/ ; https://claudeskills.info/skills/onewave-ai/claude-skills/cowork-inbox-triage/ ; https://skills.cat/skills/pedrohcgs/claude-code-my-workflow/triage-inbox
29. https://github.com/czlonkowski/n8n-mcp ; https://github.com/nerding-io/n8n-nodes-mcp
30. https://developers.make.com/mcp-server/make-mcp-server
31. https://zapier.com/pricing.md ; https://latenode.com/blog/zapier-mcp-pricing
32. https://www.activepieces.com/mcp ; https://www.activepieces.com/mcp/chatwoot
33. https://composio.dev/toolkits/livechat
34. https://composio.dev/toolkits/unipile ; https://composio.dev/toolkits/respond_io/framework/claude-code
35. https://skillsplayground.com/connectors/microsoft-365/ ; https://github.com/Softeria/ms-365-mcp-server (via https://mcpservers.org/servers/Softeria/ms-365-mcp-server)
36. https://github.com/Wh1isper/mcp-email-server ; https://github.com/codefuturist/email-mcp ; https://github.com/nikolausm/imap-mcp-server
37. YouTube search results: https://www.youtube.com/watch?v=IWG-hDTYLeY , https://www.youtube.com/watch?v=uHR8Lex_gKY , https://www.youtube.com/watch?v=Bd_Hpk7h-BM , https://www.youtube.com/watch?v=ct2ZTHNvnYo , https://www.youtube.com/watch?v=o27IR4adHeM
38. GitHub repository search via `gh api search/repositories` (2026-11-14) for star counts.
