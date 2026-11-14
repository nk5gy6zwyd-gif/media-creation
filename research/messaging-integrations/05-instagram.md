# Instagram DMs (plus comments/mentions that lead to DMs)

> Research date: 2026-11-14 · Researcher: subagent (Sonnet) · Status: draft
> Confidence note: Meta's own pages were fetched through a summarising fetcher, and several details came from third-party blogs. Items marked (unverified) need checking against Meta's current docs before you build on them. I could not see GitHub repo contents, since `gh` access to individual repos was blocked, so star counts, dates and descriptions come from search metadata only. No YouTube results surfaced.

## 1. TL;DR
- Yes, Instagram DMs can be connected to Claude. The sound route is the official **Instagram API with Instagram Login** (Meta app, professional account, webhooks in, `/messages` out). It works with a Business or Creator account and does not need a Facebook Page.
- For a small business running only its own account, **Standard Access** may be enough, so app review and business verification are probably not needed. Advanced Access (review plus business verification) is required only when serving accounts you don't own or manage.
- Simplest working path today is a **hosted tool that already has Meta approval and exposes MCP/API/webhooks**: Inrō (hosted MCP connector for Claude), ManyChat (REST API plus External Request), or respond.io (REST API plus webhooks). Next is a small self-hosted webhook receiver feeding Claude. No-code (n8n/Make/Zapier with ManyChat or Meta webhooks) sits in between.
- **No official Meta MCP for Instagram exists.** Meta's official MCP servers cover Ads, Social Technologies and WhatsApp Business (not DMs). Community servers fall into two groups. The official-API ones are young, tiny and unaudited. The unofficial ones (instagrapi, cookies, passwords) carry real account-ban and ToS risk. Avoid those on a business account.
- Main blockers: the 24-hour reply window (a human-agent tag extends it to 7 days and needs review), and Meta's review burden if you ever serve other people's accounts.

## 2. How messages arrive today
Clients message the business's Instagram professional account (Business or Creator). Messages land in the Instagram app inbox, Meta Business Suite inbox, or a third-party inbox such as ManyChat, Inrō or respond.io. Strangers who don't follow the account land in "Requests". Comments and story replies/mentions can also lead into DMs. There is no email or export by default. A one-off bulk history needs the Conversations API or Instagram's "Download your information" (not researched in detail).

## 3. Official API / platform access
Two flavours exist:

| | API with Instagram Login (newer) | API with Facebook Login (older) |
|---|---|---|
| Host | graph.instagram.com | graph.facebook.com |
| Linked Facebook Page | Not required | Required |
| Scopes | `instagram_business_basic`, `instagram_business_manage_messages` (also `..._manage_comments`) | `instagram_manage_messages` plus `pages_manage_metadata` for webhooks |
| Account type | Professional only (Business or Creator) | Professional only |

Key facts:
- **Personal accounts are out.** The API works only for professional accounts. Switching is free in Instagram settings.
- The old short scope names (`business_manage_messages` etc.) were replaced by the `instagram_business_*` names in January 2025 (secondary source, unverified).
- Send: `POST /<IG_ID>/messages` or `/me/messages` with `recipient` (IGSID) and `message`. Media limits: images up to 8MB, audio/video/files up to 25MB.
- Auth: an Instagram User access token from someone who can send from that account.
- **Access levels:** Standard Access works for accounts you own, manage or add in the App Dashboard. Advanced Access (App Review plus Business Verification) is required to serve accounts you don't own or manage. The app must be set to **Live** to receive webhooks for app users.
- **Conversations API:** lists conversations and messages for history (the rate limit is documented, but the endpoint details were not retrieved. Check the Conversations API reference).
- **24-hour window:** you can message a user only after they message you. Usually 24 hours. A Click-to-Direct ad can extend it up to 7 days.
- **HUMAN_AGENT tag:** a reply tagged `human_agent` can go out within 7 days of the user's message. It is restricted to genuine human support and requires App Review.
- **Private replies to comments:** one message per commenter, within 7 days of the comment. Follow-ups are allowed only after the person responds, and within 24 hours of that response. It goes to the main inbox if the commenter follows you, otherwise to Requests. Comment webhooks fire only for public accounts.
- **Ice breakers:** not covered in the pages I retrieved. Check Meta's Instagram messaging docs for the ice breakers/persistent menu pages.
- **Rate limits** (flowgent's summary of Meta's docs): Send API text/links/reactions 100 calls per second; audio/video 10 per second; private replies on post/Reel comments 750 per hour; Live comments 100 per second; Conversations API 2 per second; other endpoints use a rolling 24-hour allowance of 4,800 times impressions.
- Pricing: the API itself is free. No changes found for 2026 beyond the doc updates (Meta page is dated October 2026). Meta updates these pages often.

## 4. Webhooks / real-time ingestion
- Subscribe to: `messages`, `messaging_optins`, `messaging_postbacks`, `messaging_reactions`, `messaging_referrals`, `messaging_seen`, plus `comments` (and mentions/story-related fields).
- Your own sent messages come back as echoes flagged `is_echo: true`. Filter them to avoid reply loops.
- You must host an HTTPS endpoint that answers Meta's verification challenge and ideally verifies the `X-Hub-Signature-256` signature (standard Meta practice).
- App must be Live. Delivery guarantees: Meta retries on failure (standard behaviour, not verified here). Keep your endpoint fast and store events durably.
- respond.io caveat: Instagram sends no webhook for message requests, so chats must be accepted into the main inbox to be captured.

## 5. MCP servers
There is no official Meta Instagram MCP. Star counts and dates come from GitHub search metadata (2026-11-14).

| Name | Repo / URL | Official? | Read/Write | Stars / last commit | Auth | Notes |
|---|---|---|---|---|---|---|
| Inrō MCP (hosted) | https://inro.social/blog/mcp-for-instagram ; repo inro-app/instagram-mcp | Vendor-official (Inrō is a Meta Tech Provider), not Meta | Read conversations/contacts; send messages, campaigns, scenarios (25 tools) | 0 stars / 2026-11-05 | OAuth custom connector in Claude (Pro or higher) | Easiest Claude path. Write tools default to "needs approval". No sandbox mode. Requires Inrō account. |
| mcp-dir/instagram-mcp | github.com/mcp-dir/instagram-mcp | Third-party, official Graph API | Publish, comments, DMs, insights | 6 / 2026-08-30 | Meta OAuth | Paid "Advanced" tier is described for likes/follows (those would be non-official features; check). Unvetted. |
| setsmart-git/instagram-mcp | github.com/setsmart-git/instagram-mcp | Third-party, official Meta APIs | Send/reply DMs, comments, templates (Instagram, WhatsApp, Messenger) | 0 / 2026-08-26 | Meta tokens | Unvetted. |
| inyogeshwar/meta-social-technologies-mcp | github.com/inyogeshwar/meta-social-technologies-mcp | Third-party | FB Pages and Instagram DM | 0 / 2026-10-30 | Meta tokens | Unvetted. |
| commentkeyword/instagram-mcp-server | github.com/commentkeyword/instagram-mcp-server | Third-party, official API | Comment-to-DM automations | 1 / 2026-11-13 | API key | Includes agent skill and Claude Code plugin. Vendor-linked. |
| mewisme/ig-mcp | github.com/mewisme/ig-mcp | Unknown | DMs, one tool per operation | 0 / 2026-11-14 | Unknown | Not checked whether official API or private API. |
| selvin-paul-raj/Instagram-Alpic-mcp | github.com/selvin-paul-raj/Instagram-Alpic-mcp | Third-party, via Composio | Graph API tools, insights, messaging | 1 / 2026-07-21 | Composio OAuth | Adds Composio as middleman. |
| trypeggy/instagram_dm_mcp | https://glama.ai/mcp/servers/trypeggy/instagram_dm_mcp | **Unofficial** (instagrapi, username/password) | list_chats, list_messages, send, mark seen, etc. | 190 / 2025-08-13 | IG credentials | **Ban/ToS risk.** Most popular, stale. |
| eyaltoledano/insta-mcp | https://awesome.ecosyste.ms/projects/github.com%2Feyaltoledano%2Finsta-mcp | **Unofficial** (instagrapi, session cookies) | Read/send DMs | not retrieved | Cookies | **Ban/ToS risk.** |
| kynuxdev/mcp-instagram-dm | https://mcpservers.org/servers/kynuxdev/mcp-instagram-dm | **Unofficial** (private web API, cookies) | Read, send, search, manage DMs | not retrieved | Cookies | **Ban/ToS risk.** |
| Other (Zernio, InstantDM, Composio, Apify actors) | see sources | Vendor APIs | DM and comment APIs/webhooks, some with MCP | n/a | Vendor key | Hosted intermediaries; check what they store. |

Registry check: I surfaced listings on glama.ai, mcpservers.org, lobehub.com and playbooks.com via search. I did not directly query registry.modelcontextprotocol.io, smithery.ai or pulsemcp, so there may be more entries.

Risk rule: anything that asks for your Instagram password, session cookies, or uses `instagrapi` (https://github.com/subzeroid/instagrapi, ~7.1k stars, active) automates the private mobile/web API. That violates Meta's terms and can get the business account restricted or banned. Don't point it at the business account.

## 6. Claude skills & agent tooling
Found via GitHub search (names and descriptions only, not reviewed):
- inro-app/instagram-agent-skills: Claude skills for agencies running Inrō DM automation.
- LinktoDM/linktodm-dm-automation-skill: Claude/agent skill for LinktoDM (DMs, comment-to-DM, story replies, flows, contacts, webhooks).
- epictaste/agentchat-skills: Instagram comment-to-DM skills for Claude Code and Claude.
- commentkeyword/instagram-mcp-server: MCP plus Agent Skill plus Claude Code plugin.
- 0xgani/insta-dm-agent: Instagram DM auto-reply bot using Claude.
- georgeding/meta-social-ai-agent: Claude-powered Instagram DM and WhatsApp customer-service agent on Cloudflare Workers, with knowledge bank and human escalation. A useful reference for the self-hosted Agent SDK pattern.
Most are vendor-tied and very new (0 to 5 stars). Treat as examples, not dependencies.

## 7. Notable GitHub projects (non-MCP)
- Forgemind-git/ForgeGrowth-OSS (24 stars, MIT): self-hosted Instagram and WhatsApp automation with CRM, comment-to-DM and AI agents.
- n8n workflow repos: SulimanRoshid/InstagramAutomationDMs (DeepSeek agent), Ayyan18709/n8n-instagram-ai-chatbot (n8n + ManyChat + OpenAI), sebaskauf/guide-builder-agent (comment-to-DM in n8n).
- subzeroid/instagrapi and aiograpi: unofficial private-API libraries (see risk above).

## 8. YouTube / tutorials / community
The `site:youtube.com` queries returned no YouTube URLs (search tool did not honour the filter). Non-video guides found:

| Title | Channel / Author | URL | What it shows |
|---|---|---|---|
| Instagram Messaging API: Access, Permissions, and a Working Example (2026) | flowgent.ai | https://flowgent.ai/blog/instagram-messaging-api | Setup, rate limits, access levels |
| Instagram Messaging API Not Working? | DEV Community (conferbot) | https://dev.to/conferbot/instagram-messaging-api-not-working-the-toggle-the-window-8-more-causes-lib | Troubleshooting |
| Instagram MCP Server for Claude & ChatGPT (2026) | Inrō | https://inro.social/blog/mcp-for-instagram | Hosted MCP walkthrough |
| Instagram MCP: No Official Meta Server, and What Works | usecarly.com | https://www.usecarly.com/blog/instagram-mcp/ | Landscape of MCP options |
| Connecting AI Agents to Instagram | creatorflow.so | https://creatorflow.so/blog/connect-ai-agents-instagram-workflow/ | Bridging local agents via webhook or MCP |
| n8n template: Instagram DM with ManyChat | n8n | https://n8n.io/workflows/2718 | No-code reply loop |
| Manychat API and MCP | Blotato | https://www.blotato.com/blog/manychat-api | ManyChat API scope |
Follow-up: search YouTube directly for "Instagram DM n8n AI agent", "ManyChat external request", "Instagram Graph API webhook setup".

## 9. Compliance, limits & gotchas
- Professional account required. Personal accounts cannot use the API.
- You can only message users who messaged you first (24h, or 7 days via human_agent tag or ads). No cold outreach.
- Private replies: one per comment, within 7 days.
- Automation must be disclosed appropriately. Meta rules on AI/bot use of messaging: check current Platform Terms. Offer a way to reach a human. EU/UK GDPR: client messages sent to Claude are personal data. Review Anthropic's data terms and your privacy notice.
- Unofficial libraries and cookie/password MCPs risk bans. Never use them on the business account.
- App must be Live for webhooks, which sometimes needs a privacy policy URL and data-deletion callback.
- Echo events and message requests (see section 4) cause surprises.
- Tokens: long-lived Instagram tokens need periodic refresh (about 60 days, standard Meta behaviour).
- Third-party tools charge per active contact (ManyChat: Free, Essential $14, Pro $29, Business $69, Advanced $139 per month annual, per secondary sources). Which tier includes the API is disputed between sources.
- InstantDM docs say events are sent once with no retry (unlike Meta's own retries): make receivers durable.

## 10. Recommended connection path
**A. Simplest (no coding):** Make sure the account is Business/Creator. Sign up for Inrō and connect the account. Add Inrō's MCP URL as a custom connector in Claude, with write tools set to "Needs approval". Ask Claude to summarise conversations and draft replies, and send only after you approve. Check Inrō's pricing and data handling first.

**B. No-code automation:** ManyChat (or respond.io) holds the Meta approval. Use ManyChat External Request or respond.io webhooks (respond.io: Advanced plan or above) to post each message to n8n/Make/Zapier. Call the Claude API there to summarise/draft, then write back via the tool's API or log to a Google Sheet/Notion for review. ManyChat API plan tier needs confirming.

**C. Robust/self-hosted:**
1. Create a Meta developer app (Business type), add Instagram, choose "API setup with Instagram Login".
2. Add your own account as a tester/owner. Standard Access should be enough for your own account.
3. Request `instagram_business_basic` and `instagram_business_manage_messages` (plus comments scope if wanted).
4. Deploy an HTTPS webhook receiver (Cloudflare Workers or a small server). Verify the signature, ignore echoes, store events in a DB.
5. Run Claude (API or Agent SDK) over stored messages: summaries, tagging, draft replies.
6. Send replies via `/me/messages` after human approval, within 24 hours (or 7 days with human_agent if approved).
7. Optionally wrap your store and send function in your own small MCP server so Claude can use it from chat.

## 11. Open questions
- Does Standard Access really cover a single-business, own-account DM use without App Review? Verify in the app dashboard (Meta's page implies yes).
- Exact Conversations API endpoint and fields; ice breakers coverage; 2026 changes (none found, but not a thorough changelog check).
- Which ManyChat plan includes API access; Inrō pricing and MCP availability on free tier.
- Contents and security of the community official-API MCP repos (not audited).
- Directly check registry.modelcontextprotocol.io, smithery.ai, pulsemcp.
- YouTube tutorials not located.

## 12. Sources
1. https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/messaging-api (via fetch summary)
2. https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/
3. https://developers.facebook.com/documentation/instagram-platform/instagram-api-with-instagram-login/business-login
4. https://flowgent.ai/blog/instagram-messaging-api
5. https://www.postmcpai.com/instagram-messaging-api
6. https://dev.to/conferbot/instagram-messaging-api-not-working-the-toggle-the-window-8-more-causes-lib
7. https://inro.social/blog/mcp-for-instagram
8. https://www.usecarly.com/blog/instagram-mcp/
9. https://flowgent.ai/blog/instagram-mcp-server
10. https://postkey.pro/instagram-mcp
11. https://glama.ai/mcp/servers/trypeggy/instagram_dm_mcp
12. https://mcpservers.org/servers/kynuxdev/mcp-instagram-dm
13. https://awesome.ecosyste.ms/projects/github.com%2Feyaltoledano%2Finsta-mcp
14. https://creatorflow.so/blog/connect-ai-agents-instagram-workflow/
15. https://github.com/subzeroid/instagrapi
16. GitHub search results (repo names above, queried via `gh api search/repositories`)
17. https://help.respond.io/l/en/instagram/instagram-overview
18. https://github.com/api-evangelist/respond
19. https://respond.io/integrations/webhook
20. https://www.blotato.com/blog/manychat-api
21. https://flowgent.ai/blog/manychat-pricing
22. https://n8n.io/workflows/2718
23. https://instantdm.com/how-to/webhooks-and-api.md
24. https://community.make.com/t/how-to-instagram-dm-autoreply-with-manychat/85499
25. https://navoto.com/blog/meta-mcp-integrations/
