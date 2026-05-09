# Building MCP Servers on Cloudflare Workers for ChatGPT

This guide details how to build and deploy a stateless Model Context Protocol (MCP) server on Cloudflare Workers. This approach is highly optimized for ChatGPT Custom Apps because it requires **zero dependencies**, operates with **0ms cold starts**, and fits perfectly into Cloudflare's generous free tier (100,000 requests/day).

## Overview

Unlike standard local MCP servers that use `stdio` (Standard I/O) or stateful Server-Sent Events (SSE), ChatGPT Custom Apps communicate via **Stateless HTTP**. 

This means every interaction from ChatGPT is a simple HTTP `POST` request containing a JSON-RPC payload. We can implement this directly in a Cloudflare Worker without needing heavy SDKs.

## 1. Project Setup

Create a minimal directory structure:

```bash
mkdir my-mcp-server
cd my-mcp-server
npm init -y
npm install -D wrangler
```

Create a `wrangler.toml` file to configure the Cloudflare Worker:

```toml
name = "my-mcp-server"
main = "src/worker.js"
compatibility_date = "2025-05-01"

[observability]
enabled = true
```

## 2. The Worker Boilerplate (`src/worker.js`)

This is the universal boilerplate for a stateless MCP server. You only need to modify the `TOOLS` array and the `handleToolCall` function to add your own custom logic.

```javascript
/**
 * Cloudflare Workers MCP Server Boilerplate
 * Implements MCP Stateless HTTP protocol directly.
 */

// ============================================================================
// 1. DEFINE YOUR TOOLS (JSON Schema format)
// ============================================================================

const TOOLS = [
  {
    name: "hello_world",
    description: "Returns a simple greeting.",
    inputSchema: {
      type: "object",
      properties: {
        name: {
          type: "string",
          description: "The name to greet",
        },
      },
      required: ["name"],
      additionalProperties: false,
    },
  }
];

// ============================================================================
// 2. IMPLEMENT YOUR TOOL LOGIC
// ============================================================================

async function handleToolCall(toolName, args) {
  switch (toolName) {
    case "hello_world":
      return {
        content: [{ type: "text", text: `Hello, ${args.name}! I am running on Cloudflare Edge.` }]
      };
      
    // Add more cases for other tools here...

    default:
      throw new Error(`Unknown tool: ${toolName}`);
  }
}

// ============================================================================
// 3. MCP JSON-RPC ROUTER (DO NOT MODIFY UNLESS NECESSARY)
// ============================================================================

async function handleMcpRequest(body) {
  const { jsonrpc, id, method, params } = body;

  if (jsonrpc !== "2.0") {
    return { jsonrpc: "2.0", id, error: { code: -32600, message: "Invalid JSON-RPC version" } };
  }

  switch (method) {
    case "initialize":
      // ChatGPT sends this during the initial connection handshake
      return {
        jsonrpc: "2.0", id,
        result: {
          protocolVersion: "2025-03-26",
          capabilities: { tools: { listChanged: false } },
          serverInfo: { name: "cf-worker-mcp", version: "1.0.0" },
        },
      };

    case "notifications/initialized":
      // Client ack \u2014 no response needed
      return null;

    case "tools/list":
      // ChatGPT asks for available tools
      return { jsonrpc: "2.0", id, result: { tools: TOOLS } };

    case "tools/call": {
      // ChatGPT executes a tool
      const toolName = params?.name;
      const args = params?.arguments || {};
      try {
        const result = await handleToolCall(toolName, args);
        return { jsonrpc: "2.0", id, result };
      } catch (err) {
        return {
          jsonrpc: "2.0", id,
          result: { content: [{ type: "text", text: `Error: ${err.message}` }], isError: true },
        };
      }
    }

    case "ping":
      // Health check from client
      return { jsonrpc: "2.0", id, result: {} };

    default:
      return { jsonrpc: "2.0", id, error: { code: -32601, message: `Method not found: ${method}` } };
  }
}

// ============================================================================
// 4. CLOUDFLARE FETCH HANDLER
// ============================================================================

function corsHeaders() {
  return {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "POST, GET, DELETE, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type, Accept, Mcp-Session-Id",
    "Access-Control-Expose-Headers": "Mcp-Session-Id",
  };
}

export default {
  async fetch(request) {
    const url = new URL(request.url);

    // Handle CORS Preflight
    if (request.method === "OPTIONS") {
      return new Response(null, { status: 204, headers: corsHeaders() });
    }

    // Basic health check endpoint
    if (request.method === "GET" && url.pathname === "/") {
      return new Response("Cloudflare Workers MCP Server is running.", {
        headers: { "Content-Type": "text/plain", ...corsHeaders() },
      });
    }

    // Main MCP Endpoint
    if (url.pathname === "/mcp" && request.method === "POST") {
      let body;
      try {
        body = await request.json();
      } catch {
        return Response.json(
          { jsonrpc: "2.0", id: null, error: { code: -32700, message: "Parse error" } },
          { status: 400, headers: corsHeaders() }
        );
      }

      // Handle batch requests (ChatGPT sometimes sends multiple requests in an array)
      if (Array.isArray(body)) {
        const responses = [];
        for (const msg of body) {
          const res = await handleMcpRequest(msg);
          if (res) responses.push(res);
        }
        return Response.json(responses, { headers: corsHeaders() });
      }

      // Handle single request
      const result = await handleMcpRequest(body);
      if (!result) {
        // Notifications don't expect a response body
        return new Response(null, { status: 204, headers: corsHeaders() });
      }
      return Response.json(result, { headers: corsHeaders() });
    }

    // Handle standard SSE check (returns 405 to force stateless fallback)
    if (url.pathname === "/mcp" && request.method === "GET") {
      return new Response("SSE sessions not supported (stateless server)", {
        status: 405, headers: corsHeaders(),
      });
    }

    return new Response("Not Found", { status: 404 });
  },
};
```

## 3. Development and Testing

Test your worker locally:

```bash
npx wrangler dev --port 8787
```

You can simulate what ChatGPT sends using `curl`:

**Test initialization:**
```bash
curl -s -X POST http://localhost:8787/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

**Test a tool call:**
```bash
curl -s -X POST http://localhost:8787/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"hello_world","arguments":{"name":"Alice"}}}'
```

## 4. Deployment

Deploy to Cloudflare's global edge network:

```bash
npx wrangler deploy
```

Once deployed, you will get a URL like `https://my-mcp-server.<your-username>.workers.dev`.

## 5. Connecting to ChatGPT

1. In ChatGPT, navigate to **Settings** > **Apps & Connectors** > **Create**.
2. **Name**: `Your App Name`
3. **MCP Server URL**: `https://my-mcp-server.<your-username>.workers.dev/mcp` *(Don't forget the `/mcp` path)*
4. **Authentication**: `None` (or configure API keys if you add custom middleware).
5. Accept the warning and click **Create**.

## Best Practices
1. **Secrets:** If your MCP server needs API keys (e.g., calling OpenAI or a Database), store them using `wrangler secret put MY_API_KEY` and access them via the `env` parameter in your fetch handler: `async fetch(request, env)`.
2. **Custom Domains:** You can map custom domains (like `mcp.yourdomain.com`) in the Cloudflare Dashboard under your Worker's settings > Triggers > Custom Domains.
3. **Logs:** Use `npx wrangler tail` to monitor live incoming requests from ChatGPT if tools fail to execute.
