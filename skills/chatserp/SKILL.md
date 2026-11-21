---
name: chatserp
description: Turn a product, website or API into a ChatGPT plugin (OpenAI Apps SDK / MCP "plugins") that passes OpenAI review and brings users back to the product - value test, tool design from real data, MCP server (Node template), hosting, live JSON-RPC verification, privacy page, submission pack and portal submission. Invoke when the user says "ChatGPT plugin", "ChatGPT app", "Apps SDK", "get ChatGPT to recommend my product", or wants this done for a client's product.
---

# ChatSERP - get your product into ChatGPT

Builds a ChatGPT plugin (OpenAI calls them "plugins" since late 2026; the docs used to say "apps")
for a product and takes it through OpenAI's review. Works for your own products or a client's.

Docs (read the live pages before each build; rules change):
- https://developers.openai.com/plugins/build/app-quickstart.md
- https://developers.openai.com/apps-sdk/app-submission-guidelines
- https://developers.openai.com/plugins/deploy/app-review.md
- https://developers.openai.com/apps-sdk/guides/optimize-metadata
- https://developers.openai.com/plugins/concepts/ui-guidelines.md
- https://developers.openai.com/plugins/build/monetization.md
- https://developers.openai.com/plugins/build/auth.md

## Facts that kill most ideas (tell the user first)

1. **No free recommendations at launch.** A new plugin is found only by name search or direct link.
   Proactive suggestions in conversation and directory placement are "enhanced distribution": OpenAI
   picks, "Developers cannot request enhanced distribution." Usefulness + good metadata is the only lever.
2. **No ad-only plugins.** "must not exist primarily as an advertising vehicle", needs "standalone
   value", no "static frames with no meaningful interaction", "Trial or demo plugins will not be
   accepted". A plugin that only says "download our game" is rejected.
3. **Money:** commerce only for physical goods. No selling subscriptions, digital goods, credits,
   no freemium upsell, no link to checkout/upgrade pages (an informational plans page is OK).
   Listing must not mention pricing, trials, discounts, promotions.
4. **Other hard rules:** no real-money gambling; content OK for ages 13-17; minimum data, no
   precise location; only authorized APIs (no scraping, no "unofficial connectors"); tool
   descriptions must not trigger broadly or tell the model to prefer this plugin; no tool names
   like `best`, `official`, `pick_me`; don't put "MCP"/"Plugin" in the name.
5. **Developer must be verified** (platform.openai.com > Settings > Organization > Verifications).
   An identity check: only the human does it. EU-data-residency projects can't submit.

## Step 1 - Value test (go / no-go)

For each product write one line: *"A ChatGPT user asks ___; the plugin answers with ___ that ChatGPT
cannot produce alone, because it comes from ___ (our engine / our data / our account)."*
- Good: a real engine (move solver, calculator, converter), proprietary data (findings DB, catalog,
  prices), the user's own account data (OAuth), a playable interactive widget.
- No-go: marketing pages, mobile games with no web-playable core, anything whose only output is a link.
  Say so plainly and put it under Later on the board; don't build something that will be rejected.

## Step 2 - Tool design (2-4 tools)

- Every tool returns real data. Never invent stats, facts or rankings.
- Description starts "Use this when ..." and names the concrete situation; add "Do not use for ...".
- Inputs: small, validated (zod), documented format with an example. Reject bad input with a
  clear message the model can relay.
- Annotations: `readOnlyHint: true` for lookups (most tools), `openWorldHint` false unless it reaches out.
- Output: `structuredContent` (for the model/widget) + a short `content` text. Include an
  informational link back to the product page that matches the answer (never checkout).
- Optional widget (inline card / carousel / fullscreen / PiP for games): an HTML resource with mime
  `text/html;profile=mcp-app`, linked from the tool via `_meta.ui.resourceUri`, with a CSP. Use the
  product's real art and colours; no generic AI-looking gradients.

## Step 3 - Server

Template: `templates/server.md` (write its code block to `server.mjs`; Node 22, `@modelcontextprotocol/sdk`, `zod`, stateless Streamable
HTTP at `/mcp`, JSON responses, per-IP rate limit, CORS for chatgpt.com). Copy into the product repo,
replace the TOOLS section with real data access.
- Existing C#/.NET backend: NuGet `ModelContextProtocol.AspNetCore` (or a hand-written JSON-RPC
  endpoint: `initialize`, `tools/list`, `tools/call`, `resources/list`, `resources/read`).
- Python: `fastmcp` / the openai-apps-sdk-examples repo.

Hosting (cheapest first, never add paid services without the user's OK):
- Firebase Functions 2nd gen in the product's project + hosting rewrite `/mcp` -> function
  (minInstances 0, 256 MB). Keeps the product's own domain, which also hosts the domain-check file.
- Cloud Run on an existing service; Vercel/Netlify functions for static sites.
- Must be public HTTPS on a real domain for review (no tunnels/localhost).
- Domain check: serve `/.well-known/openai-apps-challenge` with the token the portal shows. Build the
  route now (token from env/config), drop the token in at submission time.

Respect the repo's own rules (CLAUDE.md / AGENTS.md): release jobs (Jenkins etc.), billing caps,
branch policy, commit identity. Never ship other sessions' uncommitted work with your release:
if the deploy packages the working tree, release from a clean worktree or wait.

## Step 4 - Verify live (mandatory before submitting)

```
python check_mcp.py https://<domain>/mcp --call <tool> '<json args>'
```
(`check_mcp.py` = the code block in `tools/check_mcp.md`, written to a file.)
It runs initialize -> tools/list -> tools/call and fails on any JSON-RPC error, a missing
"Use this when" description or an empty result. Paste the real output into the report. Also try
the 3 negative cases (bad input must give a clean error, not a 500).
Test inside ChatGPT (web, no Developer-mode toggle any more): https://chatgpt.com/plugins > **+** >
**Add custom MCP server** > name + description > Connection: public endpoint `https://<domain>/mcp` >
auth (No authentication) > accept the risk warning > **Create as a plugin** > install > new chat > type `@`
and pick it. After server changes: **Refresh** on its connection. Run the 5 positive prompts there.

## Step 5 - Privacy + listing

- Privacy page: ADD a "ChatGPT plugin" section (what is received: the tool arguments; what is
  logged and for how long; no sale/sharing). Add lines, never rewrite existing copy.
- Support contact: an existing support email/page of the product.
- Icon: square PNG from real brand assets (512x512+).

## Step 6 - Submission pack (`docs/chatgpt-plugin/` in the product repo)

- `plugin.json` + `mcp.json` (formats in app-review.md; zip them as `plugin.zip`)
- `listing.md`: name (no "MCP"/"Plugin"), one-line + full description (what it does, no pricing),
  categories, countries allowlist if needed
- `tests.md`: 5 positive prompts with the expected tool + result, 3 negative prompts where the
  plugin must NOT trigger
- `release-notes.md`, `STEPS.md` (portal clicks), icon, live check output
- Every tool annotation (`readOnlyHint`, `openWorldHint`, `destructiveHint`) with a one-line
  justification (the docs conflict on this; the submission checker can require it)
- `review.demo_recording_url` in plugin.json: a HOSTED link reviewers can open (e.g. an unlisted video)
  showing the main tools and the test cases on desktop AND mobile; record it in the ChatGPT test setup above
- Test account only if auth is used: full features, no MFA / email codes / magic links

## Step 7 - Submit (platform.openai.com > Plugins)

1. Confirm the org shows Verified.
2. Upload `plugin.zip`, fill the listing, privacy URL, support, icon, test cases, release notes.
3. Copy the domain-challenge token -> deploy it -> press verify.
4. Accepting OpenAI's plugin/developer terms is the user's click: stop and ask.
5. Submit. Review time is not published and can't be expedited. Record the submission on the
   project tracker; check status every few days, fix rejections from the review notes.

## Doing this for someone else's product (client work)

- Get written OK from the product owner and their API/data access. Their OpenAI org must be
  verified by them; submit under THEIR org, never under the user's.
- Never enter their passwords or keys into forms; they paste secrets into their own config.
- Deliver: the repo branch/PR with the server, the live endpoint, the submission pack, the check
  output, and a one-page "what ChatGPT users can now do" summary.

## After launch

- Log tool calls (count per tool, no personal data) to see which prompts actually land.
- Iterate on descriptions with the metadata guide (direct / indirect / negative prompts).
- The goal is traffic back to the product: tag links with `utm_source=chatgpt` so the product's
  analytics shows it.
