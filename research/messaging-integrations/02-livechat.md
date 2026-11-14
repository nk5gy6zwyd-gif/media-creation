# LiveChat (Text.com)

> Research date: 2026-11-14 · Researcher: subagent (Sonnet) · Status: draft

## 1. TL;DR
- **Yes, there is an official, remote MCP server: `https://mcp.text.com/`.** Text (formerly LiveChat Software) runs it. It works with Claude through Customize > Connectors > custom connector, and the login is browser-based OAuth with your Text account [1][2].
- It is **read-only and small**: `search`, `fetch`, `get-chat-transcript`, `list-archived-chats`, `list-chat-tags`, plus ticket tools (HelpDesk). It has **no send_message tool**, so Claude can summarise transcripts but can't reply through it [1][2].
- The help article says it is "available only with the Text App". I have not confirmed whether a legacy LiveChat-only plan counts. Check this on your account first [2].
- For **real-time capture and replies**, use the Platform APIs. A webhook on `chat_deactivated` fires when a chat ends. `list_archives` and `get_chat` fetch transcripts, and `send_event` posts replies. A personal access token (PAT) is enough for one business [3][4].
- No-code route: the Zapier "Chat Ended" trigger or a webhook into n8n/Make, then Claude/OpenAI, then email/Sheets/CRM [9][10].
- Biggest unknowns: rate-limit numbers, and which plans allow webhooks/API. I did not find these in the pages I could read.

## 2. How messages arrive today
LiveChat is a website chat widget. Agents reply in the LiveChat/Text app in real time. After a chat ends it is stored in **Archives** (searchable, with tags and ratings). Transcripts reach people in these ways [5]:
- **Manual:** open a chat in Archives, then three-dot menu > Send transcript > enter an email > Send copy.
- **Automatic forwarding:** Settings > Chat settings > Transcript forwarding. Enter one or more emails. It applies from the time you save and is not retroactive.
- **API or webhook:** the help article points to Platform Docs (single and bulk retrieval) and a "chat ends" webhook example.
- The article does not describe automatic transcript emails to the customer or agent. A pre-chat or post-chat survey setting may offer that. I did not verify it.
- Because forwarding goes to an address you choose, **parsing transcript emails in Gmail works**. It is the fallback path, not the best one. Use a dedicated alias such as `transcripts@yourdomain`.

## 3. Official API / platform access
Docs live at platform.text.com [3][6]. All APIs are under `https://api.livechatinc.com/v3.x/...`. v3.5 is documented, and the Configuration API v3.5 page calls itself legacy, so a newer version exists.
- **Agent Chat API** (Web API plus RTM websocket): chats, threads, events, `list_chats`, `list_archives`, `get_chat`, `send_event`, `send_message`-style events, tags, routing status [3][4].
  - `list_archives`: `POST /v3.5/agent/action/list_archives`. Filters: query text, date range, thread_ids (max 20), group_ids, agent, tag, properties, and more. `limit` defaults to 10 (max 100). Pagination uses `page_id`, which expires in at most a month. The response contains chat objects with one thread each, so the same chat can appear several times [4].
- **Customer Chat API:** the visitor side, for building your own widget or bot [7].
- **Configuration API:** agents, groups, bots, and `register_webhook`, `list_webhooks` [6].
- **Reports API:** aggregate statistics (chats, ratings, response times). It has no transcripts.
- **Auth:** PAT is "recommended" for a single account or integration. OAuth (via the Developer Console app) is for apps that act for many users. Bots can use their own tokens from v3.5 [4].
- **Scopes:** examples are `chats--all:ro/rw`, `chats--access:ro/rw`, `customers:ro/rw`. For transcript reading plus replying, use `chats--all:rw` or `chats--access:rw` [4].
- **Rate limits:** the docs say calls are rate limited and return `Too many requests` when exceeded. **The numbers were not on the pages I read** [4].
- **Plan requirements:** not stated in the docs I read. A third-party guide says raw-data export with scheduled delivery is Business/Enterprise only (unverified) [8].
- **Pricing:** see livechat.com/pricing, which I did not fetch. Starter is the cheapest tier.

## 4. Webhooks / real-time ingestion
(Docs: Configuration API `register_webhook` and the Webhooks reference [11].)
- **Events:** `incoming_chat` (new chat or new thread), `chat_deactivated` (chat ended), `incoming_event` (every message), `chat_transferred`, `thread_tagged`, `incoming_customer`, `routing_status_set`, agent/bot/group/tag config events, and more [11].
- **Envelope:** `{webhook_id, secret_key, action, organization_id, payload, additional_data}`. The `payload` varies by event, and `additional_data` may carry chat properties. `secret_key` is chosen by you at registration and echoed in each delivery, so verify it [11].
- **Registration:** use the Configuration API `register_webhook` with a URL, an action, a secret, and a type (`license` or `bot`). `register_webhook` also appears in Composio's tool list [12].
- **Not documented on the pages I read:** retry policy, delivery guarantees, required scopes. Treat delivery as at-least-once and non-guaranteed. Add a reconciliation poll of `list_archives` (for example, hourly) to catch missed chats.
- Recommended pattern: on `chat_deactivated`, call `get_chat` (or `list_archives` with `thread_ids`) to fetch the full transcript, then pass it to Claude.

## 5. MCP servers
| Name | Repo / URL | Official? | Read/Write | Stars / last commit | Auth | Notes |
|---|---|---|---|---|---|---|
| **Text MCP** | https://mcp.text.com/ docs: [1][2] | **Yes (Text)** | Read-only: search, fetch, get-chat-transcript, list-archived-chats, list-chat-tags, find-ticket, list-tickets, list-ticket-tags | Hosted, no repo (not in github.com/livechat's 176 repos) [13] | Browser OAuth with your Text account; follows your roles | Claude: custom connector. Claude Team/Enterprise: an Owner adds it under Organization settings > Connectors first. Others: `npx mcp-remote https://mcp.text.com <port>` with port 6274, 53531, 59123 or 60222. ChatGPT: only fetch/search. "Text App" only [1][2] |
| Composio LiveChat toolkit | https://composio.dev/toolkits/livechat | No (3rd party, hosted) | Read + write, 111 tools (Start Chat, Send Event, Register Webhook...) | n/a | OAuth2 with your own LiveChat OAuth app | Free tier: 100k tool calls/month. Good for replies without writing code [12] |
| Pipedream LiveChat MCP | https://mcp.pipedream.com/app/livechat | No | Likely read + write (not verified) | n/a | Pipedream-managed | Hosted [14] |
| Zapier MCP | https://zapier.com/mcp/livechat | No | Actions via Zapier | n/a | Zapier | Listed for Claude, ChatGPT, Cursor [15] |
| `@pedrohnas/livchat-mcp` | https://www.pkgstats.com/pkg:@pedrohnas/livchat-mcp | No | Unknown | Unknown | Unknown | Name suggests a different "LivChat" product. I did not inspect it. |
| brunocramos/livechat-mcp | https://github.com/brunocramos/livechat-mcp | **Unrelated** | n/a | 1 star | n/a | It is a voice-input tool, not LiveChat Software. |

GitHub repo search found no community LiveChat Software MCP server. GitHub org-level search was blocked in this environment, so I used the github.com/livechat org page instead [13]. I did not check registry.modelcontextprotocol.io, smithery.ai or the Anthropic connector directory directly. Please re-check them.

## 6. Claude skills & agent tooling
- I found no Claude skill specific to LiveChat.
- Practical setup: add the Text connector in Claude, then use a skill/prompt such as "list archived chats since <date>, fetch each transcript, summarise with sentiment, action items and follow-up needed". That fits the read-only tools directly.
- Built-in AI on the LiveChat side: **Copilot** (Sentiment, Rephrase, Suggested replies, Summarize), plus tag suggestions and chat summaries. Text says these are included in every LiveChat and Text plan. **AI Agent** (autonomous replies) is Text-plan only, not legacy plans [16].
- Marketplace apps: AI Summaries ($5.99/mo/license) and Chat Summary by SKlabs ($5/mo/agent, OpenAI) [16].
- The Agent SDK or API route can be used for auto-replies. See section 10.

## 7. Notable GitHub projects (non-MCP)
- github.com/livechat: 176 repos. SDKs: `lc-sdk-js` (8 stars), `lc-sdk-go` (14 stars), and a Python SDK (docs at livechat.github.io/lc-sdk-python [17]). Also `livechat-public-docs` (51 stars, docs source), `chat-widget-sample`, and `text-quickstart-customer-api`. Active as of Nov 2026 [13].
- Reference code for a custom MCP: `lc-sdk-js` or `lc-sdk-python`.
- Minimal-MCP starting points: the official MCP TypeScript or Python SDK (not fetched). For structure, `mcp-freescout` (31 stars) is a comparable helpdesk MCP I saw in search but did not inspect.

## 8. YouTube / tutorials / community
| Title | Channel / Author | URL | What it shows |
|---|---|---|---|
| No YouTube video found | n/a | n/a | Searches returned only vendor and template pages. Search YouTube directly. |
| Chat summary via webhook + ChatGPT (Zapier template) | Zapier | https://zapier.com/templates/details/2268bc205732497b5181fdd45a327b551ee8ef96e332c54557ef882794421619 | Webhook capture, ChatGPT summary, append to Zendesk ticket |
| LiveChat transcript to HubSpot on chat end | Zapier | https://zapier.com/templates/details/c22d4efaa47f8064dcee9d2e3f9aeb7f8f8315e9e88580cbdd54d43a0280b0c0 | Formatted transcript pushed to CRM |
| Getting started with LiveChat on Zapier | Zapier help | https://help.zapier.com/hc/en-us/articles/40859791500301-How-to-get-started-with-LiveChat-on-Zapier | PAT-based connection, "Chat Ended" trigger [9] |
| Connect ChatGPT or Claude to your chats with MCP | ChatBot help | https://www.chatbot.com/help/connect-chatgpt-or-claude-to-your-chats-with-mcp/ | Official step-by-step [2] |

## 9. Compliance, limits & gotchas
- Chat transcripts contain personal data. Tell visitors in the widget or privacy policy that chats are processed by an AI provider, and check GDPR/CCPA obligations. This is generic guidance, not from LiveChat's docs.
- The MCP only exposes what your Text user can see (roles and groups). Use an account with the right group access [1].
- Archives `page_id` expires within a month. The same chat may appear multiple times (one entry per thread) [4].
- Webhook endpoints must verify `secret_key`. A leaked URL allows forged events [10].
- Legacy-version risk: API versions v3.2 to v3.7 co-exist. Pick one and pin it. Domains differ (livechatinc.com, livechat.com, text.com) but the API host is `api.livechatinc.com`.
- Third-party MCP gateways (Composio, Pipedream, Zapier) route your chat data through their cloud.

## 10. Recommended connection path
1. **Today, 10 minutes (read and summarise):** Claude > Customize > Connectors > add custom connector, URL `https://mcp.text.com/`, log in with Text. Ask Claude to list archived chats and summarise them [2]. For Claude Code or Desktop, use `mcp-remote` or a remote server entry (not tested).
2. **Capture on chat end (no code):** Zapier "Chat Ended" trigger, then an AI step, then Sheets/Notion/email. Or n8n/Make via a webhook [9][10].
3. **Robust (own code):** create a PAT in the Developer Console with `chats--all:rw`. Register a `chat_deactivated` webhook to a small service. The service fetches the transcript with `list_archives` + `thread_ids`, summarises it with the Claude API, and stores it. Add an hourly `list_archives` reconciliation poll.
4. **Replies:** start with a draft-only flow, where Claude drafts a reply and a human sends it. To go further, post replies with the Agent Chat API (`send_event`) or Composio's Send Event tool [12]. Use a bot token for AI Agent-style responses.
5. **If you need a send tool in MCP:** build a small MCP (TypeScript SDK) with `list_chats` (list_archives), `get_transcript` (get_chat), `send_message` (send_event with type `message`), using the `lc-sdk-js` library and a PAT in an env var. Make `send_message` require confirmation. It is about 150 lines. I did not build or test it.

## 11. Open questions
- Is the Text MCP available on legacy LiveChat-only plans, or only the Text App?
- Rate limit numbers, and the plan tiers needed for webhooks and API access.
- Webhook retry and delivery guarantees.
- Does the Text MCP work in Claude Code and Desktop (claude.ai connectors sync to Desktop, but not tested)?
- Whether a tool-parameter schema is published for the MCP (the docs list tools without parameters).
- The Anthropic connector directory and mcp.so, smithery.ai and the official registry were not checked directly.
- Automatic customer-facing transcript emails: where is that setting?
- HelpDesk, KnowledgeBase and OpenWidget were not researched in depth. ChatBot has a similar MCP integration, per its help page [2], and HelpDesk has a tickets MCP article [18] (not fetched).

## 12. Sources
1. https://platform.text.com/docs/guides/mcp (official MCP docs; also search-result copy https://www.text.com/docs/getting-started/mcp)
2. https://www.chatbot.com/help/connect-chatgpt-or-claude-to-your-chats-with-mcp/
3. https://platform.text.com/docs/agent-chat-api/
4. https://platform.text.com/docs/messaging/agent-chat-api/v3.5
5. https://www.livechat.com/help/how-to-get-chat-transcripts/
6. https://developers.livechat.com/docs/management/configuration-api/v3.2 and https://platform.text.com/docs/management/configuration-api/v3.5
7. https://developers.labs.livechat.com/docs/customer-chat-api/
8. https://clonepartner.com/blog/how-to-export-data-from-livechat-methods-api-limits-formats/ (third party, unverified)
9. https://help.zapier.com/hc/en-us/articles/40859791500301-How-to-get-started-with-LiveChat-on-Zapier
10. https://agenbot.vercel.app/blog/connect-your-chat-to-zapier-n8n-make
11. https://platform.text.com/docs/management/webhooks/v3.5
12. https://composio.dev/toolkits/livechat
13. https://github.com/livechat
14. https://mcp.pipedream.com/app/livechat
15. https://zapier.com/mcp/livechat
16. https://livechat.com/features/ai , https://www.livechat.com/marketplace/apps/ai-summaries/ , https://marketplace.accounts.livechat.com/apps/chat-summary/
17. https://livechat.github.io/lc-sdk-python/agent/web/api/v37.html
18. https://www.helpdesk.com/help/connect-ai-assistants-to-tickets-with-mcp/ (appeared in search results, not fetched)
19. https://glama.ai/mcp/servers/brunocramos/livechat-mcp (unrelated project)
