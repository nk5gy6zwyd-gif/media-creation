# Phone Calls & Voicemail (Voice Channel)

> Research date: 2026-11-14 · Researcher: subagent (Sonnet) · Status: draft
> Confidence note: vendor-doc facts (Quo, Dialpad MCP, Twilio) were read from vendor pages. Items marked **[unverified]** came from search-result snippets, aggregator sites or my background knowledge and were not confirmed against vendor docs. Legal sections are research notes, not legal advice.

## 1. TL;DR
- **Best path for a small business: move business calls onto Quo (formerly OpenPhone).** It has an official, OAuth-based, hosted MCP server at `https://mcp.quo.com/mcp` and a Claude connector directory listing. Webhooks cover `call.completed`, `call.voicemail.completed`, `call.transcript.completed` and `call.summary.completed`. [1][2][5]
- **Plan gate:** Quo says the MCP is on every plan. Call transcripts and AI summaries need **Business or Scale** with call recording enabled, and AI action items need Scale. Transcripts are not generated retroactively. [1][3]
- **Runner-up for builders:** Twilio Voice, using `recordingStatusCallback` plus Voice Intelligence transcription, gives full control. It has no official MCP server that I verified, so you build the webhook receiver yourself. [8]
- **Enterprise-ish systems:** Dialpad has an official MCP, but it is Early Access and needs your Customer Success Manager. [4] Aircall's official MCP is a private beta **[unverified, blog only]**. RingCentral's official MCP exposes call logs, and I found no transcript tool.
- **Personal mobile numbers are the weak spot.** Neither iOS nor Android exposes an API for call data. The realistic fix is to route business calls through a second-line app (Quo, or Google Voice with forwarding), because that makes them capturable.
- **Main blockers:** recording consent law, which varies by state and country, and plan or beta gating. See section 9.

## 2. How calls and voicemail arrive today
- **Personal mobile or landline:** calls happen in the carrier or phone app. Voicemail sits in the carrier or visual-voicemail inbox. Nothing is machine-readable by default, so this is the biggest untracked channel.
- **Business VoIP (Quo, Dialpad, etc.):** calls are logged centrally. Recording, transcripts, voicemail transcripts and AI summaries are plan-dependent, and webhooks and APIs are available.
- **Google Voice:** voicemail transcripts can be emailed or texted. There is no official API, so wrappers like voice.js are unofficial and can break. Recording is manual and covers incoming calls only (press 4). [10][11]
- **iPhone (iOS 18.1+):** native call recording and transcription, stored in the Notes app, with an automatic notification to all parties. It needs Apple Intelligence on a supported device and region. The sources I found were secondary (MacRumors, AppleInsider, TechSpot). [12]
- **Android:** Google's 2022 Play policy blocked third-party recorders that use the Accessibility API. The Google Phone app has native recording on Pixel 6+ and some other devices. The heise report on version requirements is inconsistent, so check Google's help pages. [13]
- **Voicemail transcription on iPhone:** I found no source on this in the search. Carrier-dependent visual voicemail transcription exists but is **[unverified]**.

## 3. Official API / platform access

### Provider comparison table
| Provider | Recording | Transcription | Voicemail transcript | Webhooks | MCP | Plan / price | Notes |
|---|---|---|---|---|---|---|---|
| **Quo (OpenPhone)** | Yes, per number; required for transcripts | Business and Scale, 40+ languages, on by default per number [3] | Yes (voicemail transcript in `call.voicemail.completed`) [2][9] | `call.completed`, `call.recording.completed`, `call.voicemail.completed`, `call.transcript.completed`, `call.summary.completed`, plus message events [2] | **Official**, hosted OAuth MCP [1][5] | Starter $15, Business $23, Scale $35 per user per month, annual billing. $19, $33, $47 monthly (third-party price sources, verify on quo.com/pricing) [14] | Best fit. Port-in supported **[unverified]**. Scale adds AI action items. |
| **Dialpad** | Yes | Yes (Dialpad AI) | **[unverified]** | Call events webhooks, including AI recaps per third-party connector pages [15] | **Official, Early Access** (US and EU endpoints). Needs CSM enrollment, about a week of review. OAuth only, 24-hour sessions. [4] | Plan not stated in MCP docs. Needs a Dialpad Phone account. | Third-party sources conflict on whether AI recaps are readable via the REST API. One lists `/api/v2/call/{id}/ai_recap`. Verify in the Dialpad API reference. |
| **Aircall** | Yes | Needs AI Assist add-on **[unverified]** | **[unverified]** | Yes (REST API and webhooks) **[unverified]** | Official **private beta by invitation** [16]. Community: `themobilefirstco/aircall-mcp-server` (2 stars, pushed 2026-02-06, unofficial, 73 tools) | Not researched | Aimed at sales and support teams. |
| **RingCentral** | Yes | Via AI products, not researched | Not researched | Yes (platform webhooks) **[unverified]** | Official, 4 active servers. The call log tool is documented. I found **no transcript tool** [17] | Not researched | Third-party managed servers claim transcripts (Truto). |
| **JustCall** | Yes **[unverified]** | JustCall AI **[unverified]** | **[unverified]** | Yes **[unverified]** | `saaslabsco/justcall-mcp-server`, labelled "official" (26 stars, last push 2026-03-12) | Not researched | Check ownership. The org name is "saaslabsco", so "official" is unverified. |
| **Zoom Phone** | Yes (call recording policy) | Not confirmed for Phone | Not confirmed | Zoom webhooks exist **[unverified]** | Official Zoom MCP covers meetings, not Phone [18]. A community Zoom Phone MCP covers call logs and routing, with a write tool. | Not researched | Community Zoom servers are meeting-oriented. |
| **Google Voice** | Manual, incoming only | None for recorded calls [10] | Yes, emailed or texted [10] | None | None (no official API) | Free personal. Workspace tiers vary. | Workaround: forward voicemail email to a parser or to Claude via the Gmail connector. |
| **Twilio Voice** | Yes, `recordingStatusCallback`, `recordingChannels: "dual"` for speaker separation [8] | Voice Intelligence (V2), and a V3 batch transcription API in public beta (May 2026) [8] | Build with `<Record transcribe>` or a recording plus Voice Intelligence **[unverified detail]** | Yes, recording and transcript webhooks, with `CustomerKey` echoed back [8] | No official MCP verified. Community MCP projects exist. | Usage based | Most flexible, most code. Verify endpoint names against current docs. |
| **Ringover, Grasshopper, Nextiva, 8x8, Vonage Business** | Not researched | Not researched | Not researched | Not researched | Not researched | Not researched | Gaps. See open questions. Grasshopper is now part of Quo's family per a Quo blog URL ("grasshopper-ai") **[unverified]**. |

### Quo detail (verified)
- MCP tool areas: conversations (message history, call transcripts, missed calls, voicemail transcripts), messaging (send text, group messages), contacts (search, create, update), activity and usage, tasks, inbox settings (including call recording), A2P 10DLC registration, and Sona (beta AI agent, can place calls). [1]
- The MCP page has no dedicated summary tool. Summaries arrive through webhooks. [1][2]
- Auth uses OAuth with dynamic client registration and PKCE. The connection only sees data the authorising account can see. [1]
- Webhooks created with the `Quo-Api-Version: 2026-03-30` header use a Standard Webhooks style signature. Older webhooks still use the `openphone-signature` header. [19]

## 4. Webhooks / real-time ingestion
Quo event payloads (from the 2026-03-30 docs) [2]:
- `call.completed`: `id`, `direction`, `status` (answered, unanswered, failed, forwarded, abandoned, ai-handled, unknown), timestamps, `duration`, `hasVoicemail`.
- `call.voicemail.completed`: `voicemailId`, `callId`, `from`, `to`, `duration`, `transcript` (nullable), `recordingUrl` (nullable).
- `call.transcript.completed`: `callId`, `processingStatus`, `dialogue[]` with speaker, content, start and end.
- `call.summary.completed`: `callId`, `summary[]`, `nextSteps[]`, `handledByAiAgent`, `answeredByUserId`.
- All call events include a `context` object (org, number, conversation, contacts, participants) and a `links.quo` URL.

Delivery guarantees and retry policy for Quo were not confirmed. A third-party guide on Hookdeck covers best practices. [19]

Vapi sends an `end-of-call-report` POST about 5 to 15 seconds after a call ends. It includes the transcript, and Vapi retries up to 3 times **[unverified, third-party blog]**. [20]

Twilio: the recording callback fires on `completed`. A separate transcript webhook is configured on the Voice Intelligence Service. [8]

## 5. MCP servers
| Name | Repo / URL | Official? | Read/Write | Stars / last commit | Auth | Notes |
|---|---|---|---|---|---|---|
| Quo MCP | `https://mcp.quo.com/mcp` (docs [1][5]) | **Official** | Read and write | n/a (hosted) | OAuth, PKCE | One-click Claude connector. Voicemail transcripts and call transcripts readable. |
| openphone-mcp | github.com/studio-corsair/openphone-mcp | Community | Read SMS, list calls, send messages | 1 star, pushed 2026-03-02 | API key **[unverified]** | Low maturity. Prefer the official server. |
| Dialpad MCP | `https://mcp.dialpad.com/mcp` [4] | **Official, Early Access** | Whatever the Public API allows | n/a (hosted) | OAuth, 24h sessions | Needs CSM enrollment, otherwise 403. |
| Dialpad community | github.com/Cititec-Pty-Ltd/dialpad-mcp, aldotempo/dialpad-mcp, cprfirstaid/dialpad-mcp | Community | Mostly read | 0 stars each (Cititec pushed 2026-11-14) | API key | Very immature. |
| Aircall (unofficial) | github.com/themobilefirstco/aircall-mcp-server | Community, unofficial | Read and write (73 tools) | 2 stars, pushed 2026-02-06 | API credentials | Transcripts need Aircall AI add-on **[unverified]**. |
| Aircall (read-only) | github.com/growth-box/aircall-mcp | Community | Read-only | 0 stars, pushed 2026-11-13 | API key | Safer for a first test. |
| JustCall MCP | github.com/saaslabsco/justcall-mcp-server | Claims "official" | Read and write **[unverified]** | 26 stars, pushed 2026-03-12 | API key | Most stars of the third-party phone servers I found. |
| RingCentral MCP | developers.ringcentral.com/mcp/servers/ [17] | Official | Call log read, presence, SMS | n/a | OAuth | No transcript tool found. |
| AgentLine | github.com/AgentLineHQ/AgentLine | Community | Calls, SMS, transcripts via MCP | 62 stars, pushed 2026-10-21 | API key | Open-source AI phone API (FastAPI, SignalWire, Deepgram). Self-hostable. Unvetted. |
| Vapi MCP | Several community servers on Glama. Vapi official docs MCP at `docs.vapi.ai/_mcp/server` [20] | Docs server is official | Community servers read call details, transcripts, recordings | Not checked | API key | The official server is for documentation, not for call data. |
| Retell MCP | Scalekit-hosted connector, community node [21] | Not official **[unverified]** | Not checked | Not checked | Access token | Treat as unverified. |

## 6. Claude skills & agent tooling
- `call-summary` skill in `anthropics/knowledge-work-plugins`: takes pasted notes or a transcript and returns action items with owners, an internal summary, and a customer follow-up email. The search snippet reported 33.3K stars and 2.9K installs, last updated 2026-08-21. **[unverified: aggregator listing, did not open the repo]** [22]
- `call` community skill (openclaw/skills): includes a follow-up tracking script. **[unverified]** [22]
- ZoomInfo call-recap skill: needs the ZoomInfo MCP plugin, so it is sales-CRM-specific. [22]
- No ready-made Quo-plus-Claude follow-up tracker was found. The simple build: Quo webhooks into a small receiver (or n8n), which stores `call.summary.completed` and `call.voicemail.completed` rows. A scheduled Claude task then lists open `nextSteps` per contact. Alternatively, use Claude with the Quo MCP tools (tasks plus conversations) directly, since the Quo MCP can create and complete tasks tied to calls. [1]

## 7. AI voice agents & answering services
| Service | Logs calls? | Webhooks / API | MCP / Claude route | Notes |
|---|---|---|---|---|
| Vapi | Yes, transcript and recording | `end-of-call-report` webhook [20] | Community MCP servers, n8n MCP tutorials | Build-your-own receptionist. Voice is the product. |
| Retell | Yes | Webhook events **[unverified]** | Community or Scalekit connector | Details unverified. |
| Bland | Not covered by my searches | Not confirmed | Not confirmed | Gap. |
| Synthflow | Post-call webhooks only **[unverified, vendor comparison blog]** | Yes | No MCP found **[unverified]** | |
| Smith.ai | Human plus AI receptionists, call summaries | Zapier integration with API key [23] | Zapier MCP listing mentions Smith.ai, details unconfirmed [23] | Good "no-code" route: Smith.ai to Zapier to Claude. |
| Ruby | Human receptionists | Zapier integration. Cannot be used with Ruby's HIPAA service [24] | None found | |
| Goodcall, Rosie | Not researched | Not researched | Not researched | Gaps. |
| Quo Sona (beta) | Yes, calls appear in Quo with `handledByAiAgent` | Same Quo webhooks | Sona tools in Quo MCP [1] | Keeps everything in one system. 10 free Sona calls per month on paid plans [14]. |

## 8. YouTube / tutorials / community
No YouTube video was verified via oEmbed. The search tool returned no direct video URLs for Quo plus Claude. Leads only:
| Title | Channel / Author | URL | What it shows |
|---|---|---|---|
| I Built an AI Voice Receptionist with Vapi and n8n MCP (digest page) | Nate Herk (AI Automation) per Recapio | https://recapio.com/digest/i-built-an-ai-voice-receptionist-with-vapi-and-n8n-mcp-by-nate-herk-ai-automation | Vapi plus n8n MCP receptionist. **Digest page, not the video. Video URL and title unverified.** |
| Quo MCP setup guide (text) | Quo | https://www.quo.com/product/ai/mcp.md | Claude Settings, Connectors, Quo, Connect. Quo's AI blog mentions a Sona video, but no link was exposed. [5] |
| Claude and Quo integration guide (text) | usecarly.com (third party) | https://www.usecarly.com/blog/claude-quo-integration/ | Claude Code HTTP transport setup. URL differs from Quo's own, so cross-check. |

## 9. Compliance, limits & gotchas

**US**
- Federal law (ECPA) is one-party consent. Several states require all-party consent. One source lists California, Delaware, Florida, Illinois, Maryland, Massachusetts, Montana, Nevada, New Hampshire, Pennsylvania, Washington. Counts differ between sources (11 to 13), so check current statutes. [25]
- For interstate calls, the safe rule is to follow the stricter state. [25]
- Apple's native recording and Google Voice both announce recording to participants. The Google Voice claim comes from third-party sources. [10][12]

**UK / EU / AU** (sources are vendor and law-firm blogs, not regulators)
- UK: recording is generally lawful, but a recording that identifies a person is personal data under UK GDPR and DPA 2018. You need a lawful basis (often legitimate interests, with a documented assessment), a notice to callers, and limited retention. [26]
- EU: GDPR applies on top of national rules. Germany reportedly requires all-party consent and France treats unauthorised recording as a privacy offence. No EU-wide retention period was found. [26]
- Australia: sources conflict. Federal law turns on whether a party knows of the recording. State surveillance-devices laws differ (one source says NSW is one-party, and Victoria, Queensland, WA, SA and Tasmania are stricter). Verify before recording. [26]

**Practical rules**
- Play a disclosure at the start ("This call may be recorded and transcribed for quality and record keeping").
- Voicemail is usually left knowingly, but still personal data.
- Feeding transcripts to an AI is a further processing purpose. Mention it in your privacy notice and check Claude plan data terms. **[not researched]**
- Define retention (for example, delete recordings after 90 days, keep summaries). Restrict webhook receivers and store signing secrets.
- Health or legal clients: check HIPAA or confidentiality obligations. Ruby's Zapier integration is explicitly disabled for its HIPAA tier. [24]

## 10. Recommended connection path
1. **Simplest:** create a Quo account on **Business** ($23 per user per month, annual). Port or forward the business number (porting rules not verified, so check Quo's help). Turn on call recording and transcription on that number and add a recording disclosure greeting.
2. In Claude: Settings, Connectors, add **Quo**, and authorise via OAuth. Ask Claude to summarise yesterday's calls and voicemails, find missed calls without follow-up, and create Quo tasks. [1][5]
3. **Robust:** register a Quo webhook for `call.completed`, `call.voicemail.completed`, `call.transcript.completed`, `call.summary.completed`. Verify the signature, store events in a database, and run a daily Claude job that extracts follow-ups. Optionally have Claude draft a reply and send a text through the Quo MCP, with human approval. [2][19]
4. **Personal mobile:** give clients the Quo number instead (or call-forward a business line to it). For voicemail from a mobile you must keep, forward carrier voicemail transcripts to email and let Claude read them via the Gmail connector. **[carrier-dependent, unverified]**
5. **Alternative stack:** Twilio number, `recordingStatusCallback`, Voice Intelligence transcript webhook, then a Claude summary. [8] Or an answering service (Smith.ai) piped through Zapier. [23]

## 11. Open questions
- Does Quo's `call.voicemail.completed` transcript depend on plan, as call transcripts do? The support page does not state a plan for voicemail transcription. [3]
- Quo number porting steps, fees and timelines were not researched.
- Quo webhook retry and delivery guarantees are unconfirmed.
- Dialpad AI recap via REST or webhook only: sources conflict.
- Ringover, Grasshopper, Nextiva, 8x8, Vonage Business, Bland, Goodcall and Rosie were not researched. Aircall plan requirements and RingCentral transcripts were not verified in vendor docs.
- Official Google help pages on Voice recording and Pixel recording disclosure were not fetched.
- UK ICO, Ofcom and Australian statute text were not read directly.
- No YouTube video titles were confirmed by oEmbed.
- Whether the `anthropics/knowledge-work-plugins` star and install counts are accurate.

## 12. Sources
1. https://www.quo.com/docs/mcp/overview.md (fetched)
2. https://www.quo.com/docs/2026-03-30/webhooks-event-payloads (fetched)
3. https://support.quo.com/core-concepts/calling/summaries-and-transcripts (fetched)
4. https://developers.dialpad.com/docs/dialpad-mcp-server (fetched)
5. https://www.quo.com/product/ai/mcp.md (search result)
6. https://support.openphone.com/core-concepts/integrations/mcp (search result)
7. https://support.quo.com/core-concepts/integrations/mcp.md (search result)
8. https://twilio.com/docs/voice/intelligence/voice-intelligence-best-practices ; https://www.twilio.com/en-us/changelog/v3-batch-transcription-configuration-api ; https://dev.classmethod.jp/en/articles/twilio-conversational-intelligence-functions-voice-recording-transcript-summary-sentiment/ (search results)
9. https://www.quo.com/docs/2026-03-30/webhooks-overview (search result)
10. https://www.quo.com/blog/google-voice-api/ ; https://policies.google.com/technologies/voice (search results)
11. https://npmjs.org/package/voice.js (search result)
12. https://www.macrumors.com/how-to/ios-record-your-phone-calls/ ; https://appleinsider.com/articles/24/07/30/how-to-use-call-recording-and-transcription-in-ios-181 ; https://www.techspot.com/news/103345-apple-introduce-call-recording-transcription-features-ios-18.html (search results)
13. https://heise.de/-11203807 ; https://www.techspot.com/news/94312-google-play-store-bans-call-recording-apps.html (search results)
14. https://www.quo.com/blog/quo-pricing/ ; https://krispcall.com/general/quo-pricing/ ; https://www.usecarly.com/blog/quo-pricing/ (search results)
15. https://developers.dialpad.com/docs/call-events ; https://www.stackone.com/connectors/dialpad/ ; https://www.withone.ai/knowledge/dialpad/conn_mod_def%3A%3AGOB39cHGOkA%3A%3AhB4JhNAeQcuM9kZDmGA9LQ (search results)
16. https://vantagepoint.io/blog/integrations/aircall/aircall-mcp-beta-claude-chatgpt ; https://lobehub.com/mcp/themobilefirstco-aircall-mcp-server (search results)
17. https://developers.ringcentral.com/mcp/servers/ ; https://developers.ringcentral.com/mcp/solutions/communication-analytics/ (search results)
18. https://glama.ai/mcp/servers/Justin-Gladiator/zoom-phone-va-mcp-server (search result)
19. https://hookdeck.com/webhooks/platforms/guide-to-quo-webhooks-features-and-best-practices (search result)
20. https://www.ciela.ai/blogs/vapi-end-of-call-report-webhook ; https://glama.ai/mcp/servers/ageborn-dev/vapi-mcp-server (search results)
21. https://www.scalekit.com/connectors/retell ; https://trillet.ai/blogs/voice-ai-platform-api-comparison-for-developers (search results)
22. https://claudemarketplaces.com/skills/anthropics/knowledge-work-plugins/call-summary ; https://openskillindex.com/skills/clawdbot-skills-call ; https://gtm.ai/marketplace/gtm-skills/zoominfo-call-recap (search results)
23. https://www.smith.ai/integrates-with/zapier ; https://zapier.com/apps/thryv/integrations/smithai-zapier-app-da100556 (search results)
24. https://rubyhelpcenter.helpjuice.com/zapier-integration (search result)
25. https://www.rev.com/blog/productivity/phone-call-recording-laws-state ; https://www.metaverselaw.com/two-party-consent-requirements-for-recording-calls/ ; https://vibe.us/blog/one-party-two-party-consent-states/ (search results)
26. https://sprintlaw.co.uk/articles/business-call-recording-laws-in-the-uk/ ; https://gdprlocal.com/call-centre-gdpr/ ; https://justcall.io/blog/call-recording-laws-by-country-2026.html (search results)
27. GitHub search API results via `gh api search/repositories` (2026-11-14) for the repo star counts and push dates cited in section 5.
