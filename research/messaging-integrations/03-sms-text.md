# SMS / Text Messages (business SMS + personal phone texting)

> Research date: 2026-11-14 · Researcher: subagent (Sonnet) · Status: draft
> Confidence note: most facts come from vendor pages, aggregators and web-search summaries. Items marked (unverified) were not confirmed against the primary source. Star counts and last-commit dates were mostly not retrievable.

## 1. TL;DR
- Yes, SMS can be connected to Claude. Every serious provider has a REST send API plus an inbound webhook. That is the robust path: webhook -> small service -> Claude API / Agent SDK -> reply or log.
- Simplest path for a small business with no dev work: use **Quo (formerly OpenPhone)**. It has an official MCP connector for Claude, built-in texting, and signed webhooks. It reads message history, summarises threads and sends texts [5][6][7].
- Most flexible path: **Twilio** (or Telnyx) numbers, with your own webhook endpoint. Twilio's official MCP (`twilio-labs/mcp`, alpha) exposes the whole API, but it is large and some community reports say it has schema problems with Claude (unverified) [1][2].
- Biggest blocker for US business SMS is **carrier registration**: A2P 10DLC (local numbers) or toll-free verification. Unregistered traffic gets filtered. Budget days to weeks [12][13].
- Personal-phone texting (iMessage/Android) is workable for reading and drafting through macOS chat.db MCP servers or Anthropic's own Claude Code iMessage channel plugin. It is not a business-grade or ToS-safe channel for client work: it needs Full Disk Access to your whole message history [17][18].
- Compliance (TCPA, STOP handling, quiet hours) applies as soon as Claude sends anything. Keep a human approval step for replies at first.

## 2. How messages arrive today
Clients text a business number, either a landline/mobile line or a VoIP/CPaaS number. Options for the owner today:
- **SMB phone app** (Quo, Dialpad, RingCentral, Aircall, Google Voice): texts show up in an app inbox shared by a team.
- **CPaaS number** (Twilio, Telnyx, Vonage, Plivo, Sinch, Bird): texts arrive only as API events (webhooks) you must handle or forward.
- **Personal phone**: texts live in iMessage/Messages on the phone, synced to a Mac's `~/Library/Messages/chat.db`, or in Google Messages on Android.
For Claude, the meaningful ingestion options are: live webhook, MCP read of an inbox, or periodic export (chat.db, Android backup XML).

## 3. Official API / platform access

### Provider comparison
| Provider | Inbound webhook | Send API | Numbers | US price (indicative) | US registration | Non-US notes | Port-in | MCP |
|---|---|---|---|---|---|---|---|---|
| **Twilio** | Yes, per-number webhook (HTTP POST, TwiML reply or REST), signature validation | REST Messages API | Local, toll-free, short code, alphanumeric sender in some countries | SMS $0.0083/segment in/out plus carrier fees ($0.0025-$0.005); MMS out $0.022; local number $1.15/mo, toll-free $2.15/mo [3] | 10DLC: Sole-prop brand $4.50 one-time, $2/mo campaign; Standard brand about $46 plus campaign fee (third-party figures) [12]. Toll-free verification required | UK/AU sender-ID registration docs exist [15] | Yes (bring your own number, $0.50/mo per Twilio pricing) [3] | Official alpha `twilio-labs/mcp` [1] |
| **Telnyx** | Yes (messaging profile webhook) | REST | Local, toll-free, short code | Not retrieved | 10DLC and toll-free same US rules | Multi-country | Yes (not verified here) | Official; local Python (deprecated) and TS in `telnyx-node` [4][8] |
| **Vonage** | Yes | Messages / SMS API | Virtual numbers | about $0.0081 out / $0.0065 in per third-party guides (unverified) [11] | Same US rules | UK restrictions page exists [15] | Yes (unverified) | Official docs-only MCP; telephony server is community [9] |
| **Plivo** | Yes | REST | Local, toll-free | Not retrieved | Same | UK guideline page [15] | Not verified | Community only [10] |
| **Sinch** | Yes | REST / Conversation API | Local, toll-free | Not retrieved | Same | UK sender ID page [15] | Not verified | Official, developer preview, about 60 tools [14] |
| **Bird (MessageBird)** | Yes | REST | Local, toll-free | Not retrieved | Bird says its MCP can register 10DLC brands (vendor claim) | n/a | Not verified | Hosted `mcp.bird.com` (vendor claim); a Rework article says SMS is "coming soon" on Bird's pricing page, so availability is unclear [10] |
| **Quo (OpenPhone)** | Yes, signed (Standard Webhooks, HMAC-SHA256), `message.received` etc. [6] | REST; sending needs prepaid credits [5] | Local and toll-free business lines | Subscription per user plus credits (not retrieved) | Quo handles carrier registration for its numbers (not verified) | US/Canada focus (unverified) | Yes, number porting supported (unverified) | Official MCP, works with Claude [5] |
| **Dialpad** | Yes (events) | REST | Business numbers | Not retrieved | Handled by vendor | Multi-country | Yes | Official MCP, Early Access via CSM; Streamable HTTP [10b] |
| **RingCentral** | Yes (subscriptions) | REST (SMS endpoint) | Business numbers | Not retrieved | Vendor-handled | Multi-country | Yes | Official RingEX Phone MCP in "Labs", includes send-SMS tool, no SLA [10c] |
| **Aircall** | Yes | SMS send via API on Professional plan, US/CA/DE/FR/AU only | Business numbers | Not retrieved | Vendor-handled | Limited | Yes | No official; community (Allo) with SMS [10d] |
| **Google Voice** | No | None. Automation violates Google's terms (per Quo competitor blog, verify) [16] | Personal/Workspace | n/a | n/a | n/a | Port in is possible but not useful | None viable |

### Registration and sender equivalents
- **US**: A2P 10DLC brand + campaign registration for local numbers (days to about 2 weeks; sources disagree), or toll-free verification (reports range from days to several weeks). Sole-proprietor path is cheap but has low throughput limits [12].
- **UK**: no central sender-ID registry mandated by Ofcom (July 2026 rules lean on provider KYC and sender ID consistency); providers such as Twilio, Sinch, Plivo ask you to register alphanumeric sender IDs [15].
- **Australia**: the ACMA SMS Sender ID Register is live. Unregistered alphanumeric IDs have been shown as "Unverified" since 1 July 2026, so register through your provider [15b].
- Alphanumeric senders are one-way: they cannot receive replies. Use a real number if you want two-way.

## 4. Webhooks / real-time ingestion
- **Twilio**: configure the number's "A message comes in" URL (POST, form-encoded: From, To, Body, MessageSid, NumMedia, MediaUrlN). Validate the `X-Twilio-Signature`. Reply with TwiML or via REST. Status callbacks report delivery.
- **Quo**: `message.received` and other `message.*` events; envelope `id/apiVersion/createdAt/type/data`; signature headers `webhook-id/timestamp/signature`; up to 8 retries over about 27.5 hours, so deduplicate by `webhook-id`; max 50 webhooks per workspace [6].
- **BlueBubbles** (personal iMessage on a Mac): REST API plus webhooks with a server password [19].
- General: you need a public HTTPS endpoint (ngrok/Cloudflare tunnel in development). Respond 2xx quickly and process asynchronously. Providers retry, so make handlers idempotent.
- Claude cannot receive webhooks directly. Either run a small service that calls the Claude API/Agent SDK, or let an MCP server poll/read on demand when you ask.

## 5. MCP servers
| Name | Repo / URL | Official? | Read/Write | Stars / last commit | Auth | Notes |
|---|---|---|---|---|---|---|
| Twilio MCP (`@twilio-alpha/mcp`) | github.com/twilio-labs/mcp [1] | Yes (Twilio Alpha) | Read+write across entire Twilio API; filter with `--services/--tags` | 114 stars, 101 commits; last commit date not retrieved | `AccountSID/APIKey:Secret` | Alpha; large tool surface can overflow context, so filter to Messaging. Twilio warns against running community servers alongside it. One community listing claims schema bugs with Claude (unverified) [2] |
| Telnyx MCP | github.com/team-telnyx/telnyx-mcp-server (Python, deprecated) and `telnyx-node` `packages/mcp-server` [4][8] | Yes | Send SMS/MMS, get message, WhatsApp, numbers | n/a | `TELNYX_API_KEY` | Hosted remote option reported but not confirmed on docs page |
| Quo MCP | support.quo.com MCP page [5] | Yes | Read inbox/messages/calls/transcripts/contacts; send texts; add/update contacts | n/a | Not stated on page (one listing mentions API key) | Texts need prepaid credits. Third-party sources conflict on endpoint (`mcp.quo.com/sse` vs `/mcp`) |
| Quo community | glama.ai armavita-quo-mcp [7] | No | Includes webhook tools | n/a | API key | Unreviewed |
| Sinch MCP | github.com/sinch/sinch-mcp-server [14] | Yes (developer preview) | `send-text-message` plus about 60 tools | 1 star | Project key/secret env vars | Very early |
| Vonage Documentation MCP | github.com/Vonage/vonage-mcp-server-documentation [9] | Yes | Docs only | n/a | none | Not a messaging tool |
| Vonage Telephony MCP | github.com/Vonage-Community/telephony-mcp-server [9] | Community | SMS and voice | n/a | Vonage creds | Python |
| Plivo MCP | mcp.so/servers/plivo-mcp-server [10] | Community | Send SMS | n/a | Plivo auth id/token | |
| Bird MCP | mcp.bird.com (vendor claim) [10] | Yes (claimed) | Send SMS, delivery status, 10DLC brand registration | n/a | n/a | Verify availability |
| Dialpad MCP | developers.dialpad.com/docs/dialpad-mcp-server [10b] | Yes | Calls, messages, meetings, transcripts | n/a | Dialpad auth | Early Access only |
| RingCentral RingEX Phone MCP | mcp.labs.ringcentral.com [10c] | Yes (Labs) | Includes SMS send tool | n/a | RingCentral OAuth | No SLA, tools may change |
| Aircall (Allo) | lobehub.com/mcp/themobilefirstco-aircall-mcp-server [10d] | No | 73 tools incl. SMS | n/a | API key | Not endorsed by Aircall |
| Twilio SMS community (YiyangLi/sms-mcp-server) | glama.ai/mcp/servers/@YiyangLi/sms-mcp-server [2] | No | Send SMS/MMS | n/a | Twilio creds | Small, send-focused |
| Twilio community (griffinwork40/twilio-mcp) | glama.ai/mcp/servers/@griffinwork40/twilio-mcp [2] | No | SMS with threading plus Express webhook for inbound | n/a | Twilio creds | Closest to "inbound + reply" |
| Anthropic iMessage channel plugin | anthropics/claude-plugins-official (`imessage`) [17] | Yes (Anthropic) | Reads chat.db, replies via Messages; allowlist access | n/a | Local, Full Disk Access | macOS, Claude Code 2.1.80+; messages can carry prompt injection |
| iMCP | youtube demo [20]; app by Loopwork | Community | Messages, Contacts, more | n/a | macOS permissions | Native macOS app |
| adelaidasofia/imessage-mcp | github.com/adelaidasofia/imessage-mcp [18] | No | Read, FTS5 search, send via AppleScript with confirm | 2 stars | Full Disk Access | Obsidian export |
| tim0120, cyberpapiii/imessage-max, anipotts/imessage-mcp, hannesrudolph/imessage-query, carterlasalle/mac_messages_mcp, DavidGreigQC | various [18] | No | Mostly read (chat.db); some send via AppleScript; Mac Messages MCP falls back to SMS | low stars | Full Disk Access | Review code before granting access |
| Beeper `mcp-android` | developers.beeper.com/open-source [19] | Beeper | Android content providers | n/a | on-device | Listed by Beeper; details not verified |
| Hosted aggregators | Pipedream, Composio, Zapier, Vinkius | Third party | Varies | n/a | Via aggregator | Adds a third party that sees your client messages |

Registry check: searches surfaced Glama, LobeHub, mcp.so, mcpservers.org, playbooks.com. I did not query registry.modelcontextprotocol.io, smithery.ai or pulsemcp directly, so coverage there is unverified.

## 6. Claude skills & agent tooling
- `twilio/ai` skills repo (small, about 26 stars): `twilio-messaging-webhooks`, `twilio-webhook-architecture`. Install: `npx skills add twilio/ai --skill twilio-messaging-webhooks --agent claude-code` [21].
- `tdimino/claude-code-minoan` `twilio-api` skill: webhook signature validation, TwiML examples [21].
- Claude Code **channels** (research preview docs) let Claude Code receive messages from Telegram, Discord and iMessage while a session runs [17].
- No official Agent SDK + Twilio example was found. Build: Express/FastAPI webhook -> validate signature -> look up client history -> `@anthropic-ai/claude-agent-sdk` or Messages API with tools (send_sms, log note) -> reply via TwiML or REST [21].
- Sendblue (iMessage/SMS API, paid) markets Claude Agent SDK integration [21].
- Composio publishes Telnyx and other SMS toolkits for the Claude Agent SDK [21].

## 7. Notable GitHub projects (non-MCP)
- BlueBubbles server (Mac relay for iMessage, REST + webhooks) [19].
- `mautrix-imessage` and `mautrix-gmessages` Matrix bridges; `beeper/bridge-manager` to self-host bridges [19b].
- `beeper/imessage`, `libragen/imessage` (iMessage libraries) [19b].
- Android: SMS Backup & Restore (XML export) and Pushbullet/Join were in the task brief but I did not research them and have no sources. Treat as open.

## 8. YouTube / tutorials / community
| Title | Channel / Author | URL | What it shows |
|---|---|---|---|
| iMCP: Connect Claude to your Messages, Contacts, & more | Loopwork (Mattt) per search summary | https://www.youtube.com/watch?v=ney6KTq_3mc | iMessage via macOS MCP app |
| Set Up Claude Code iMessage Plugin | not retrieved | https://www.youtube.com/watch?v=FIFonDfRBjQ | Official channel plugin setup, Full Disk Access |
| Claude Code with iMessage, SMS, and Email in 1 Minute | not retrieved | https://www.youtube.com/watch?v=MTVkW_EG-Y4 | Quick SMS/iMessage/email setup |
| Send SMS from Claude Code using an SMPP SMS MCP Server | not retrieved | https://www.youtube.com/watch?v=3t_lxbL7-ac | SMS MCP over SMPP |
| Connect Claude with Twilio / Build voice Agents 10x faster | not retrieved | https://www.youtube.com/watch?v=61vvmZswTbY | Twilio + Claude (voice focus) |
| Claude Can Now Email, Text, and iMessage You! | not retrieved | https://www.youtube.com/watch?v=Z9jgFpCwzS0 | Inkbox-style connector |
| Claude Code Really Killed Openclaw With This iMessage Update | not retrieved | https://www.youtube.com/watch?v=pbe3iQxGXp8 | iMessage channel commentary |
| Texting Claude Code From Your iPhone (blog) | Dave Swift | https://daveswift.com/imessage-claude-code-channel/ | Always-on daemon setup |
| Twilio MCP Server Setup for Claude and AI Agents 2026 (blog) | Activepieces | https://www.activepieces.com/blog/twilio-mcp-server-setup-for-claude-and-ai-agents-2026.md | Twilio MCP walkthrough |
Titles and URLs come from search results; I did not watch the videos, so content descriptions are from titles only.

## 9. Compliance, limits & gotchas
- **TCPA / consent (US)**: marketing texts need prior express written consent. The FCC one-to-one consent rule was vacated by the 11th Circuit in January 2025 (sources still conflict in 2026 guides). Transactional/customer-service replies to a client who texted first are lower risk, but get legal advice [22].
- **Opt-out**: honour STOP, QUIT, END, REVOKE, OPT-OUT, CANCEL, UNSUBSCRIBE; revocation by any reasonable means; honour within 10 business days at most. Parts of the revoke-all rule were delayed (possibly to 2027; unconfirmed) [22]. Providers like Twilio auto-handle STOP on their numbers; do not let Claude send to opted-out numbers.
- **Quiet hours**: federal 8am-9pm recipient local time for marketing; some states stricter [22].
- **Carrier filtering**: unregistered 10DLC traffic is blocked or heavily filtered; keep message content matching your registered campaign. Avoid shared links/URL shorteners and high-risk content [12].
- **Provider fees**: per-segment pricing, long messages split into segments; MMS costs more; failed messages carry a small fee [3].
- **Data/privacy**: sending client messages to Claude/third-party MCP aggregators shares client data. Prefer first-party MCP or your own webhook; scrub PII where possible; log retention policy.
- **Prompt injection**: inbound client texts are untrusted input. Claude must not be able to change access lists, send to arbitrary numbers or run destructive tools from message content (Anthropic's plugin explicitly warns) [17].
- **Personal phone**: iMessage MCP servers need Full Disk Access (whole chat history, including unrelated personal chats). Apple has no official iMessage API; Beeper Mini's iMessage-on-Android was shut down by Apple in December 2023, and sending via AppleScript or bridges risks account issues [19b]. Google Voice automation is against its terms and has no API [16].
- **Twilio MCP**: Twilio says do not combine it with community MCPs; apply minimal API-key scopes (Restricted API keys).

## 10. Recommended connection path
**Tier 1 (no code, read/summarise/draft):** Put business texting on Quo (port your existing number in if needed), enable the Quo connector in Claude, and ask Claude to summarise threads and draft replies. Send only with your approval. Note that API-sent texts consume prepaid credits.

**Tier 2 (automated ingestion, stays in your control):**
1. Get a Twilio (or Telnyx) number; port your existing number in; start A2P 10DLC (or toll-free verification) immediately because approval takes days to weeks.
2. Webhook endpoint (small serverless function) validates the signature, stores each message (e.g. in a database/spreadsheet) keyed by phone number, and returns 200 quickly.
3. Claude (Agent SDK or API) runs on a schedule or per message to summarise, tag, and write a draft reply to a queue.
4. Human approves, then the service sends via REST. Add STOP handling and quiet-hours checks in code.
5. Optionally add Twilio MCP filtered to Messaging (and a restricted API key) so you can ask Claude ad hoc questions in Claude Desktop.

**Tier 3 (personal phone only, if essential):** Use Anthropic's Claude Code iMessage channel or a read-only chat.db MCP on a dedicated Mac, restricted to an allowlist; expect privacy and ToS caveats.

## 11. Open questions
- Exact current pricing for Telnyx, Plivo, Sinch, Bird, Quo plans and Quo API-credit costs.
- Quo MCP endpoint and auth method (sources conflict), and whether it works in Claude web/mobile.
- Current Twilio toll-free verification cost and time; sole-prop 10DLC throughput limits.
- Whether Twilio's MCP currently works cleanly in Claude (reported schema bug unverified).
- Android personal texting (SMS Backup & Restore, Pushbullet/Join, Google Messages bridges): not researched.
- Registry coverage (registry.modelcontextprotocol.io, smithery, pulsemcp) and GitHub search API were not queried directly; star counts and last commits mostly missing.
- Legal review of TCPA status (revoke-all rule timing, one-to-one consent).

## 12. Sources
Numbered keys used above; all consulted via WebSearch/WebFetch this session.
1. https://github.com/twilio-labs/mcp
2. https://glama.ai/mcp/servers/@YiyangLi/sms-mcp-server ; https://glama.ai/mcp/servers/dkecc42h7a ; https://glama.ai/mcp/servers/@griffinwork40/twilio-mcp/tree/5bff3758bd6c09ee44f3be9904ed2cc4f86692f4 ; https://lobehub.com/mcp/griffinlong-twilio-mcp
3. https://www.twilio.com/en-us/sms/pricing/us
4. https://developers.telnyx.com/docs/development/mcp
5. https://support.quo.com/core-concepts/integrations/mcp (and https://support.openphone.com/core-concepts/integrations/mcp)
6. https://www.quo.com/docs/2026-03-30/webhooks-overview ; https://www.quo.com/docs/mdx/beta/webhooks-quickstart
7. https://glama.ai/mcp/servers/EfrainTorres/armavita-quo-mcp/tools/list_webhooks ; https://supergood.ai/api-report-card/openphone ; https://www.usecarly.com/blog/quo-api/
8. https://glama.ai/mcp/servers/team-telnyx/telnyx-mcp-server/tools/send_message ; https://pypi.org/project/telnyx-mcp
9. https://developer.vonage.com/en/blog/telephony-mcp-server-for-agentic-ai-and-language-models ; https://github.com/Vonage/vonage-mcp-server-documentation
10. https://mcp.so/servers/plivo-mcp-server ; https://bird.com/products/sms/vs/plivo ; https://resources.rework.com/tools/messaging-sms/best-messagebird-alternatives ; https://kudosity.com/resources/articles/kudosity-vs-sinch-vs-infobip-mcp-sms-servers-compared
10b. https://developers.dialpad.com/docs/dialpad-mcp-server
10c. https://mcp.labs.ringcentral.com/docs/servers/ringex-phone/
10d. https://lobehub.com/mcp/themobilefirstco-aircall-mcp-server ; https://aircall.io/en-au/call-center-software-features/messaging-api-webhooks/
11. https://www.vonage.com/communications-apis/sms/pricing/ ; https://apidog.com/blog/vonage-sms-api-cost
12. https://support.twilio.com/hc/en-us/articles/1260803965530 ; https://www.sociocs.com/post/twilio-10dlc-explained/ ; https://signalhouse.io/guides/a2p-carrier-fees-2026/ ; https://dev.to/alfredo_romero_fc4afdcb68/a2p-10dlc-survival-guide-for-voice-agencies-2026-carrier-filtering-is-now-real-time-29mp
13. https://www.twilio.com/en-us/a2p-10dlc
14. https://github.com/sinch/sinch-mcp-server
15. https://support.twilio.com/hc/en-us/articles/23737742623387 ; https://support.app.sinch.com/hc/en-us/articles/11464132571663-UK-Sender-ID-Requirements ; https://old-support.plivo.com/hc/en-us/articles/29367458653721-UK-SMS-Regulations-Guidelines-to-Register-Your-Sender-ID ; https://mobileecosystemforum.com/2026/07/15/ofcoms-new-rules-on-mobile-messaging-scams-raising-standards-while-preserving-a-proportionate-framework/
15b. https://www.acma.gov.au/articles/2026-06/sms-sender-id-register-goes-live-help-protect-australians-scams ; https://sinch.com/engage/resources/sms-compliance/acma-sms-sender-id-register/
16. https://www.quo.com/blog/google-voice-api/ (competitor, biased) ; https://support.google.com/voice/answer/9266964
17. https://code.claude.com/docs/en/channels.md ; https://daveswift.com/imessage-claude-code-channel/ ; https://www.claudepluginhub.com/plugins/anthropics-imessage-external-plugins-imessage
18. https://github.com/adelaidasofia/imessage-mcp ; https://mcpservers.org/servers/hannesrudolph/imessage-query-fastmcp-mcp-server ; https://claudemarketplaces.com/mcp/cyberpapiii/imessage-max ; https://claudemarketplaces.com/mcp/anipotts/imessage-mcp ; https://glama.ai/mcp/servers/@carterlasalle/mac_messages_mcp/blob/697d3e304a3c209e60032bb1367784345f76254c/README.md
19. https://docs.bluebubbles.app/server/developer-guides/rest-api-and-webhooks
19b. https://docs.mau.fi/bridges/go/imessage/index.html ; https://developers.beeper.com/open-source ; https://discuss.privacyguides.net/t/beeper-mini-native-imessage-for-android/15414.md
20. YouTube links listed in section 8 (from search results)
21. https://claudeskills.info/ja/skills/twilio/ai/twilio-messaging-webhooks/ ; https://skills.cat/skills/tdimino/claude-code-minoan/twilio-api ; https://www.sendblue.com/integrations/claude-agent-sdk ; https://composio.dev/toolkits/telnyx/framework/claude-agents-sdk
22. https://www.infobip.com/blog/tcpa-compliance-sms (via cdn-web.infobip.com markdown export) ; https://www.nelsonmullins.com/insights/alerts/fcc-download/all/fcc-adopts-overhaul-of-tcpa-consent-revocation-rules-revoke-all-narrowed-exclusive-opt-out-methods-permitted-and-fraud-alert-exemption-expanded ; https://www.faegredrinker.com/zh/insights/publications/2026/10/new-fcc-rule-gives-callers-more-control-over-tcpa-opt-outs-if-they-act ; https://www.mondaq.com:443/unitedstates/advertising-marketing-branding/1751936/fcc-delays-revoke-all-consent-rule-for-robocalls-and-text-messages-until-2027
