# Facebook Messenger and WhatsApp

> Research date: 2026-11-14 · Researcher: subagent (Sonnet) · Status: draft
>
> Confidence note: Meta's own docs were only partly reachable (several developers.facebook.com fetches failed or were not attempted). Many facts below come from search summaries and third-party vendor blogs. Items marked **[verify]** need checking against Meta's docs before you rely on them.

## 1. TL;DR
- **Messenger (Facebook Page inbox):** Can be connected to Claude via the official Messenger Platform (Meta developer app, Page access token, webhooks, Send API, Conversations API). There is **no official Meta Messenger MCP server**. Community Facebook Pages MCP servers exist but are tiny (a few stars). Realistic path: small webhook service, or Zapier/Make/n8n, that stores messages and calls Claude.
- **WhatsApp:** Official route is the WhatsApp Business Platform (Cloud API). Meta launched a **"WhatsApp Business Tools MCP"** on 2026-09-15, but it is for setup and admin (accounts, numbers, templates, webhook tests), not day-to-day inbox reading or replying [verify details].
- **Easiest read-and-summarise route for a WhatsApp Business app user:** Cloud API **coexistence** (keeps the phone app working, mirrors messages to webhooks, optional history sync of up to ~6 months) via a BSP or Meta Embedded Signup. This is the most promising "no switching" path.
- **Popular community MCP** `lharries/whatsapp-mcp` (about 6.9k stars) works through the unofficial WhatsApp Web protocol. It is read/write and handy for personal experiments. It breaches WhatsApp ToS for business use and carries ban and prompt-injection risk. Not recommended for a client-facing business number.
- **Main blockers:** Meta app review / Business Verification for Messenger production access; the 24-hour reply window on both channels; per-message WhatsApp fees, which changed again on 2026-10-01 (sources conflict); and Meta's 2026 policy against general-purpose AI assistants on WhatsApp (support/sales bots remain allowed).

## 2. How messages arrive today
**Messenger:** Customers message the Facebook Page. The business reads and replies in the Meta Business Suite inbox (web/app) or Messenger app. Meta Business Suite is a human UI; there is no export of the inbox other than via the API (Conversations API, webhooks).

**WhatsApp:** Two modes:
1. **WhatsApp Business app** (phone, up to linked devices): free, manual, no official read API.
2. **WhatsApp Business Platform (Cloud API)**: messages arrive as webhooks to your server or a BSP; replies sent via API. A number normally cannot be in both modes, except with **coexistence** (section 3).

## 3. Official API / platform access

### 3a. Messenger Platform
- **Setup:** Create a Meta developer app (Business type), add the Messenger product, link your Facebook Page, generate a **Page access token** in Messenger settings. Requests are authenticated with the page-level token in `access_token`. Exchange for a long-lived token for production. [Meta quick start](https://developers.facebook.com/docs/messenger-platform/getting-started/quick-start)
- **Sending:** `POST https://graph.facebook.com/<version>/me/messages` (Send API). [Send API](https://developers.facebook.com/docs/messenger-platform/reference/send-api)
- **Permissions:** `pages_messaging` is needed to send/receive. Related: `pages_show_list`, `pages_manage_metadata` (webhook subscription), `pages_read_engagement`. Exact requirement for reading conversations **[verify]** (the permissions section of the Conversations reference was not captured).
- **Development mode vs live:** In dev mode only people with a role on the app (admins, developers, testers) can message the bot. Serving the public needs **App Review** for the permissions and, typically, **Business Verification**. For a business that only wants to read its *own* Page, the app owner usually has a role on the app and Page, which can work in dev mode / standard access for own assets **[verify]**.
- **Conversations API (history):** `GET /PAGE-ID/conversations?platform=messenger` lists conversations; `GET /CONVERSATION-ID?fields=messages` returns message IDs; `GET /MESSAGE-ID` gives message details. Only the **20 most recent messages** per conversation have details and time-based pagination is not available. Needs a Page token from someone with MESSAGING or MODERATE task. [Conversations API](https://developers.facebook.com/documentation/business-messaging/messenger-platform/conversations), [Graph reference](https://developers.facebook.com/docs/graph-api/reference/page/conversations/)
- **Messaging window:** 24 hours from the user's last message. After that, the **HUMAN_AGENT tag** allows a human reply for up to 7 days; requires approved "Human Agent" access and is for manual human replies only (automated use gets flagged). [Send API reference](https://developers.facebook.com/docs/messenger-platform/reference/send-api), [third-party summary](https://singhamandeep.com/?p=1404). One blog claims that from 2026-09-28 Click-to-Messenger ad chats get a 7-day window; single secondary source, **[verify]** ([Spurnow](https://www.spurnow.com/en/blogs/instagram-messenger-ads-7-day-messaging-window)).
- **Handover protocol / conversation routing:** Lets multiple apps share a Page inbox (primary receiver bot, secondary receiver, e.g. Page Inbox, with pass/take/request thread control). The docs URL I tried returned 404 (moved), so details **[verify]** at the current Meta docs.
- **Rate limits / pricing:** Messenger messaging has no per-message fee. Send API rate limits exist and are per-Page (see Send API "limits" section) **[verify exact numbers]**.

### 3b. WhatsApp Business Platform
- **Business app vs Cloud API:** App = manual, free. Cloud API = Meta-hosted API, requires a Meta Business portfolio, WhatsApp Business Account (WABA), phone number, and usually Business Verification for higher limits.
- **BSPs:** Twilio, 360dialog, Wati, Telnyx, Positus/Robbu, Jelou, etc. offer onboarding, inboxes, and webhooks; they add their own fees on top of Meta's.
- **Pricing (as of Nov 2026, low confidence):** Meta moved from per-conversation to **per-message** billing on 2025-07-01. Templates are billed by category (marketing, utility, authentication). Third-party sources quote US marketing about $0.025, utility about $0.004 (or $0.0034), authentication higher in some sources. Sources **disagree** on the 2026-10-01 change: some say service (non-template) replies in the 24h window are now billed after 1,000 free per number per month, others say billing now applies with no free allowance. The Click-to-WhatsApp free-entry window is quoted as 72 hours or up to 7 days. **Check Meta's official pricing page before budgeting.** [Engagelab](https://www.engagelab.com/blog/whatsapp-pricing-2026-service-message-cost), [Blueticks](https://blueticks.co/blog/whatsapp-business-pricing-change-2026-per-message), [Chattigo](https://chattigo.com/en/blog/How-much-does-WhatsApp-Business-charge-per-message-in-2026/)
- **24h window and templates:** Free-form replies only within 24 hours of the customer's last message; outside it, only pre-approved template messages.
- **Coexistence:** Lets the same number run on the WhatsApp Business app and the Cloud API. Per BSP docs: optional history sync (up to ~6 months of individual chats; groups not synced; media only recent), app messages are mirrored to the API via the `smb_message_echoes` webhook, contact changes via `smb_app_state_sync`, history via the `history` webhook. Do not uninstall the Business app. Sources conflict on whether and how you can disconnect. [360dialog](https://docs.360dialog.com/partner/waba-management/phone-number-and-hosting/using-whatsapp-app-and-cloud-api-simultaneously), [Wati](https://www.wati.io/en/blog/whatsapp-coexistence-solution/), [Kapso](https://kapso.ai/blog/whatsapp-business-app-coexistence-cloud-api), [YCloud](https://www.ycloud.com/blog/whatsapp-business-app-coexistence-meta-update)
- **Reading history:** Cloud API has no "fetch old messages" endpoint; you store what webhooks deliver. Coexistence's one-time history import is the exception.
- **AI policy:** From 2026-01-15, Meta's terms bar general-purpose AI chatbot providers (assistant as the main product). Businesses using AI for support/sales/booking remain allowed; some exceptions for Brazil, Italy, EEA, Turkiye reported; regulators have opened probes. [TechCrunch](https://techcrunch.com/2025/10/18/whatssapp-changes-its-terms-to-bar-general-purpose-chatbots-from-its-platform), [respond.io](https://respond.io/blog/whatsapp-general-purpose-chatbots-ban)

### 3c. Unofficial libraries
| Library | Notes |
|---|---|
| whatsapp-web.js (Chromium-driven), Baileys (WebSocket, no browser), whatsmeow (Go; used by lharries bridge) | Reverse-engineered WhatsApp Web protocol. Violates WhatsApp ToS for automation; accounts can be banned suddenly and permanently. Maintainers say they are unofficial/not affiliated. Star counts could not be fetched (GitHub access to those repos was denied in this session). |

Sources: [Zylos](https://zylos.ai/zh/research/2026-01-26-whatsapp-api-automation), [checkleaked](https://whatsapp.checkleaked.cc/blog/avoid-whatsapp-ban). All vendor/third-party; no primary WhatsApp statement was retrieved **[verify]**. Using these on a main business number is a business risk.

## 4. Webhooks / real-time ingestion
**Messenger:** Subscribe the Page to webhook fields `messages`, `messaging_postbacks` (also `message_deliveries`, `message_reads`, `messaging_handovers` as needed). Setup: HTTPS callback URL, verify token handshake (GET `hub.challenge`), then POST events with `entry[].messaging[]` containing `sender.id`, `recipient.id`, `timestamp`, `message.text/attachments`. Must verify `X-Hub-Signature-256` with the app secret. Missing the `messages` subscription is the most common reason triggers do not fire ([DEV article](https://dev.to/techresolve/solved-issues-with-messenger-trigger-new-message-sent-to-page-2bi4), [Zapier community](https://community.zapier.com/troubleshooting-99/facebook-messenger-new-message-sent-to-page-works-in-test-trigger-but-live-messages-create-no-zap-run-53936?postid=206564)).

**WhatsApp:** Same Meta webhook mechanism (verify token + signature), subscribed on the WABA. Events: inbound `messages`, status updates, template status, plus coexistence echo/history events. Meta retries failed deliveries for a limited time **[verify]**; store payloads and dedupe by message ID.

## 5. MCP servers
| Name | Repo / URL | Official? | Read/Write | Stars / last commit | Auth | Notes |
|---|---|---|---|---|---|---|
| Meta WhatsApp Business Tools MCP | announced via developers.facebook.com blog 2026-09-15 (per [winzheng](https://www.winzheng.com/en/article/meta-whatsapp-business-mcp-ai-agents)) | Yes (Meta) | Admin/setup: create WABA, add/verify numbers, draft templates, test messages/webhooks, monitor ToS/payment/verification | n/a | Not described **[verify]** | Aimed at developers; reportedly not for day-to-day sending. Companion "Meta Social Technologies MCP" for docs/endpoint discovery; gained app/business-portfolio tools 2026-10-26 ([navoto](https://navoto.com/blog/meta-mcp-integrations/)). One Nov 2026 vendor post says no official WhatsApp MCP exists ([usecarly](https://www.usecarly.com/blog/whatsapp-mcp/)); conflicting, treat the Sept reports as more likely but unconfirmed. |
| Meta Ads MCP | mcp.facebook.com/ads | Yes | Ads management, 29 tools | open beta since 2026-04-29 | Meta Business OAuth | Ads only, not inbox ([usecarly](https://www.usecarly.com/blog/meta-ads-mcp/)) |
| lharries/whatsapp-mcp | https://github.com/lharries/whatsapp-mcp | No | R/W: 12 tools (search_contacts, list_messages, list_chats, get_chat, send_message, send_file, send_audio_message, download_media, etc.) | ~6.9k stars; MIT; 27 commits; last commit date not shown | QR link to personal WhatsApp via Web multi-device API (re-auth ~20 days) | Local Go bridge + Python MCP, SQLite store. README warns of "lethal trifecta" prompt-injection exfiltration. Unofficial, ban risk. |
| FelixIsaac/whatsapp-mcp-extended | github.com/FelixIsaac/whatsapp-mcp-extended | No | R/W, 26 tools, webhooks | 44 stars / 2026-11-13 | QR (unofficial) | Maintained fork of lharries, adds anti-ban claims |
| tihomir22/whatsapp-mcp-theorangecat, quauhtlimtz/whatsapp-mcp, ZeinKaskas/whatsapp-mcp, others | GitHub | No | R/W | 1-2 stars each | QR | Small forks of lharries |
| tkhattar14/whatsapp-business-mcp | github.com/tkhattar14/whatsapp-business-mcp | No | R/W via Cloud API, 50+ tools (templates, media, flows) | 5 / 2026-03-28 | Cloud API token | Sending/management; no history, since the API has none |
| spirit122/whatsapp-mcp-server | github.com/spirit122/whatsapp-mcp-server | No | Cloud API, 35 tools, Cloudflare Workers | 3 / 2026-03-26 | Cloud API token | Immature |
| saravanaspar/WBMCP | github.com/saravanaspar/WBMCP | No | Cloud API | 2 / 2026-07-02 | Cloud API token | Immature |
| metabusiness-mcp/meta-business-mcp | github.com/metabusiness-mcp/meta-business-mcp | No | Cloud API, 24 tools, compliance engine | 1 / 2026-07-01 | Cloud API token | Immature |
| kaewz-manga/facebook-pages-mcp-community | github.com/kaewz-manga/facebook-pages-mcp-community | No | R/W, 28 tools: posts, comments, photos, videos, insights, Messenger | 3 / 2026-02-16 | Page token | Only Messenger-capable Pages MCP found; unproven |
| oliverames/meta-mcp-server | github.com/oliverames/meta-mcp-server | No | 200+ tools across Pages, Instagram, ads | 68 / 2026-10-29 | Meta tokens | Broad; Messenger coverage unverified **[verify]** |
| Hosted/commercial: Wassenger, GREEN-API, Composio, Periskope, Klavis | [Wassenger](https://app.wassenger.com/help/mcp-claude), [GREEN-API](https://green-api.com/articles/en/connecting-green-api-mcp-server-to-claude-and-chatgpt/), [Composio](https://composio.dev/toolkits/whatsapp/framework/claude-code) | No | R/W | n/a | API key | Several are unofficial-session based (check each). |

Not checked: registry.modelcontextprotocol.io, mcp.so, glama.ai, smithery.ai, pulsemcp directly (search results only surfaced mcpservers.org). Repo search of GitHub topics was done via `gh` for stars/dates above. GitHub star data for whatsapp-web.js, Baileys, whatsmeow was blocked.

## 6. Claude skills & agent tooling
- No Messenger/WhatsApp-specific Claude skill found in this research (not exhaustively searched).
- Pattern that works: webhook receiver writes messages to a DB/sheet; Claude (Agent SDK or scheduled task) reads and summarises; replies drafted for human approval then sent via Send API/Cloud API. Meta's WhatsApp Business Tools MCP and Social Technologies MCP can be added to Claude for setup help.
- Claude Desktop config of local MCP servers is documented in the lharries guides ([ChatPRD](https://www.chatprd.ai/how-i-ai/workflows/how-to-connect-whatsapp-to-claude-ai-using-a-model-context-protocol-mcp), [Apidog](https://apidog.com/blog/whatsapp-mcp-server)).

## 7. Notable GitHub projects (non-MCP)
- whatsapp-web.js, Baileys, whatsmeow (unofficial; see 3c).
- gcarreno-dev/elizabeth-ai-agent (2 stars): WhatsApp Business API sales agent example, Groq-based.
- `@warriorteam/messenger-sdk` (npm) as a Messenger SDK ([unpkg](https://app.unpkg.com/@warriorteam/messenger-sdk@1.5.8/files/README.md)); opsdroid Messenger connector ([docs](https://docs.opsdroid.dev/en/latest/connectors/facebook.html)). Not vetted.
- EvolutionAPI (unofficial WhatsApp REST gateway) is used in a Portuguese Claude tutorial; same ToS risk.

## 8. YouTube / tutorials / community
| Title | Channel / Author | URL | What it shows |
|---|---|---|---|
| Setup WhatsApp MCP Server to connect AI Agents with your WhatsApp | unknown | https://www.youtube.com/watch?v=aHtpbVLT3eY | Periskope WhatsApp MCP with Cursor/Claude Desktop |
| Claude conectado a WhatsApp por MCP (and #2) | unknown (Spanish) | https://www.youtube.com/watch?v=4lGhYaF0Ex4 , https://www.youtube.com/watch?v=odoR__tU8fY | Claude Desktop + WhatsApp MCP |
| Conectamos Claude con WhatsApp usando MCP | unknown (Spanish) | https://www.youtube.com/watch?v=MQpLow1AD3g | Same, walkthrough |
| GitHub - lharries/whatsapp-mcp | unknown | https://www.youtube.com/watch?v=qY2UK3lU5UM | Repo-focused video |
| AGENTES no Claude com Whatsapp usando MCP Server e EvolutionAPI | unknown (Portuguese) | https://www.youtube.com/watch?v=OQvZ-41MMM4 | EvolutionAPI-based MCP |
No Messenger webhook YouTube tutorial was surfaced; use Meta's written [quick start](https://developers.facebook.com/docs/messenger-platform/getting-started/quick-start). Video contents were not watched; titles/channels come from search snippets only.

## 9. Compliance, limits & gotchas
- 24h windows on both channels; Messenger HUMAN_AGENT up to 7 days (human only); WhatsApp needs templates outside the window.
- WhatsApp fees per message; rule changed 2026-10-01; confirm on Meta's pricing page.
- Meta policy on general-purpose AI bots (since 2026-01-15); customer-service use allowed.
- Messenger production access needs App Review (and likely Business Verification); can take days/weeks.
- Unofficial libraries/MCPs: ToS violation, ban risk, personal-account scope, WhatsApp linked-device limits, and prompt injection (messages from strangers are untrusted input to Claude). Keep send tools off or human-approved.
- Privacy/consent: you are sending customer messages to an LLM; disclose in privacy policy; GDPR/UK rules apply as relevant.
- Conversations API returns only recent messages; start storing via webhooks early.
- Zapier community reports triggers that pass tests but do not fire live (Messenger and WhatsApp) — usually webhook subscription or number registration issues.

## 10. Recommended connection path
**Simplest (read/summarise only, minimal risk):**
1. Messenger: create Meta app, link Page, subscribe webhook `messages`; use Zapier "New Message Sent to Page" or Make "Facebook Messenger" trigger to append rows to a Google Sheet/Airtable. WhatsApp: if on the Business app, ask a BSP (e.g. 360dialog, Wati) to onboard via **coexistence** with history sync; use its Zapier/Make trigger ([Zapier WhatsApp Business](https://zapier.com/apps/whatsapp-business/integrations/whatsapp-notifications/255694493/send-whatsapp-notifications-messages-for-new-whatsapp-business-messages-received), [Make WhatsApp Business Cloud](https://www.make.com/en/help/app/whatsapp-business-cloud), n8n WhatsApp Trigger node).
2. Point Claude (Cowork/scheduled task, or a Sheets/Airtable MCP) at that store for daily summaries.

**More robust:**
3. Own small webhook service (n8n self-hosted works; one vendor claims native n8n/Make webhook triggers can fail Meta's verification handshake, unverified) that verifies signatures, stores events in a DB, exposes a read MCP to Claude.
4. Add draft-reply tool: Claude proposes, human approves, service sends via Send API / Cloud API, respecting windows and templates.
5. Optionally add Meta's WhatsApp Business Tools MCP to manage templates and webhook testing.
6. Avoid lharries-style QR bridges on the business number; if used at all, use a spare number for experiments.

## 11. Open questions
- Exact Meta pricing after 2026-10-01 and the free-window rules (official page).
- Exact permissions and App Review needs for reading *own* Page conversations (dev mode may suffice).
- Whether the Meta WhatsApp Business Tools MCP supports any message read/send; its auth model.
- Handover protocol current status/URL; Messenger rate-limit numbers.
- Whether coexistence is available in the user's country/BSP, and exact history-sync limits.
- Confirm the 2026-09-28 Click-to-Messenger 7-day window.
- Registries (mcp.so, glama, smithery, pulsemcp) not directly checked.

## 12. Sources
1. https://developers.facebook.com/docs/messenger-platform/getting-started/quick-start
2. https://developers.facebook.com/docs/messenger-platform/reference/send-api
3. https://developers.facebook.com/documentation/business-messaging/messenger-platform/conversations
4. https://developers.facebook.com/docs/graph-api/reference/page/conversations/
5. https://singhamandeep.com/?p=1404
6. https://www.spurnow.com/en/blogs/instagram-messenger-ads-7-day-messaging-window
7. https://www.engagelab.com/blog/whatsapp-pricing-2026-service-message-cost
8. https://blueticks.co/blog/whatsapp-business-pricing-change-2026-per-message
9. https://chattigo.com/en/blog/How-much-does-WhatsApp-Business-charge-per-message-in-2026/
10. https://docs.360dialog.com/partner/waba-management/phone-number-and-hosting/using-whatsapp-app-and-cloud-api-simultaneously
11. https://www.wati.io/en/blog/whatsapp-coexistence-solution/
12. https://kapso.ai/blog/whatsapp-business-app-coexistence-cloud-api
13. https://www.ycloud.com/blog/whatsapp-business-app-coexistence-meta-update
14. https://techcrunch.com/2025/10/18/whatssapp-changes-its-terms-to-bar-general-purpose-chatbots-from-its-platform
15. https://respond.io/blog/whatsapp-general-purpose-chatbots-ban
16. https://zylos.ai/zh/research/2026-01-26-whatsapp-api-automation
17. https://whatsapp.checkleaked.cc/blog/avoid-whatsapp-ban
18. https://www.winzheng.com/en/article/meta-whatsapp-business-mcp-ai-agents
19. https://navoto.com/blog/meta-mcp-integrations/
20. https://www.usecarly.com/blog/whatsapp-mcp/
21. https://www.usecarly.com/blog/meta-ads-mcp/
22. https://github.com/lharries/whatsapp-mcp
23. GitHub repo search results via `gh api search/repositories` (FelixIsaac, tkhattar14, spirit122, saravanaspar, metabusiness-mcp, kaewz-manga, oliverames, etc.), run 2026-11-14
24. https://dev.to/techresolve/solved-issues-with-messenger-trigger-new-message-sent-to-page-2bi4
25. https://community.zapier.com/troubleshooting-99/facebook-messenger-new-message-sent-to-page-works-in-test-trigger-but-live-messages-create-no-zap-run-53936?postid=206564
26. https://zapier.com/apps/whatsapp-business/integrations/whatsapp-notifications/255694493/send-whatsapp-notifications-messages-for-new-whatsapp-business-messages-received
27. https://www.make.com/en/help/app/whatsapp-business-cloud
28. https://www.chatprd.ai/how-i-ai/workflows/how-to-connect-whatsapp-to-claude-ai-using-a-model-context-protocol-mcp
29. https://apidog.com/blog/whatsapp-mcp-server
30. https://app.wassenger.com/help/mcp-claude
31. https://green-api.com/articles/en/connecting-green-api-mcp-server-to-claude-and-chatgpt/
32. https://composio.dev/toolkits/whatsapp/framework/claude-code
33. YouTube URLs listed in section 8 (from search results only)
