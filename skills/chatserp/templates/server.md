# MCP server template (server.mjs)

Write the code block below to `server.mjs` in the product repo (Node 22+), then `npm i @modelcontextprotocol/sdk zod express`. Run locally with `PORT=8787 node server.mjs`.

```js
// ChatGPT plugin MCP server template (Node 22+).
// npm i @modelcontextprotocol/sdk zod express
// Stateless Streamable HTTP at POST /mcp, JSON responses. Replace the TOOLS section with real data.
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { z } from "zod";

const PRODUCT_URL = process.env.PRODUCT_URL || "https://example.com";
const CHALLENGE_TOKEN = process.env.OPENAI_APPS_CHALLENGE || ""; // from the Plugins portal
const WIDGET_URI = "ui://widget/card.html";

function link(path) {
  const u = new URL(path, PRODUCT_URL);
  u.searchParams.set("utm_source", "chatgpt");
  return u.toString();
}

function buildServer() {
  const server = new McpServer({ name: "product-name", version: "1.0.0" });

  // Optional widget: real brand art/colours, CSP restricted to what it needs.
  server.registerResource("card", WIDGET_URI, {}, async () => ({
    contents: [{
      uri: WIDGET_URI,
      mimeType: "text/html;profile=mcp-app",
      text: `<!doctype html><meta charset="utf-8"><div id="root"></div>
<script>/* read the tool result via the MCP Apps bridge / window.openai.toolOutput and render */</script>`,
      _meta: { ui: { csp: { connectDomains: [], resourceDomains: [] } } },
    }],
  }));

  // ---------------- TOOLS (replace) ----------------
  server.registerTool(
    "search_items",
    {
      title: "Search items",
      description:
        "Use this when the user asks about <concrete situation> and wants <concrete answer>. " +
        "Returns matching entries from <product>'s own database. Do not use for general questions.",
      inputSchema: { query: z.string().min(2).max(200).describe("What to look for, e.g. 'peek advantage'") },
      annotations: { readOnlyHint: true, openWorldHint: false },
      _meta: { ui: { resourceUri: WIDGET_URI } },
    },
    async ({ query }) => {
      const results = []; // TODO real lookup; never invent data
      return {
        structuredContent: { query, results, more: link("/") },
        content: [{ type: "text", text: results.length ? `${results.length} results` : "No match in the database." }],
      };
    }
  );
  // --------------------------------------------------

  return server;
}

const app = express();
app.use(express.json({ limit: "64kb" }));

// tiny per-IP rate limit (60 req/min)
const hits = new Map();
app.use((req, res, next) => {
  const ip = req.headers["x-forwarded-for"]?.toString().split(",")[0] || req.ip;
  const now = Date.now(), w = hits.get(ip)?.filter(t => now - t < 60000) || [];
  w.push(now); hits.set(ip, w);
  if (w.length > 60) return res.status(429).json({ error: "rate_limited" });
  next();
});

app.get("/.well-known/openai-apps-challenge", (_req, res) =>
  CHALLENGE_TOKEN ? res.type("text/plain").send(CHALLENGE_TOKEN) : res.sendStatus(404));

app.post("/mcp", async (req, res) => {
  const server = buildServer();
  const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined, enableJsonResponse: true });
  res.on("close", () => { transport.close(); server.close(); });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});
app.get("/mcp", (_req, res) => res.status(405).json({ error: "POST only" }));

export default app; // Firebase: export const mcp = onRequest({ memory: "256MiB", minInstances: 0 }, app)
if (process.argv[1] && import.meta.url.endsWith(process.argv[1].split(/[\\/]/).pop())) {
  app.listen(process.env.PORT || 8787, () => console.log("MCP on :" + (process.env.PORT || 8787) + "/mcp"));
}
```
