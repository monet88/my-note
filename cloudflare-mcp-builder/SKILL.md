---
name: cloudflare-mcp-builder
description: Build and deploy a stateless Model Context Protocol (MCP) server on Cloudflare Workers for ChatGPT Custom Apps. Use this skill whenever the user asks to migrate an MCP server to Cloudflare, build a serverless MCP, create a ChatGPT Custom App using MCP, or deploy an MCP server without stateful SSE/stdio transports.
---

# Build Cloudflare MCP Server (Stateless)

To accomplish the deployment of an MCP server to Cloudflare Workers, you must bypass the standard MCP transport layers (stdio/SSE) and implement a lightweight JSON-RPC router that interfaces directly with the MCP SDK's internal registry. This is required because ChatGPT Custom Apps communicate via Stateless HTTP (JSON-RPC POST requests).

## Scope and Security
This skill handles the migration and creation of stateless MCP servers for Cloudflare Workers. 
Does NOT handle standard SSE or stdio MCP server creation.
Ensure all API keys are passed via Cloudflare Secrets (`env.SECRETS`) and not hardcoded. Do not log sensitive keys.

## 1. Initialize the Environment
Run these commands to set up the necessary dependencies:
```bash
npm install @modelcontextprotocol/sdk zod
npm install -D wrangler typescript @cloudflare/workers-types esbuild
```

## 2. Configure Cloudflare Worker
Create `wrangler.toml` at the project root. You MUST enable `nodejs_compat` to allow the MCP SDK to run in the V8 isolate.

```toml
name = "mcp-server"
main = "src/worker.ts"
compatibility_date = "2025-05-01"
compatibility_flags = ["nodejs_compat"]

[observability]
enabled = true
```

Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "Bundler",
    "types": ["@cloudflare/workers-types"],
    "strict": true,
    "skipLibCheck": true
  }
}
```

## 3. Implement the Stateless Adapter
Create `src/worker.ts`. Because we do not use the SDK's built-in transports, we must manually parse the JSON-RPC request and route it. To serve `tools/list` and `tools/call`, extract the tools from `server._registeredTools`.

Use this exact boilerplate:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { z } from 'zod';

export interface Env {
  // Bind Cloudflare Secrets here
  API_KEY?: string;
  API_KEYS?: string; // Comma-separated list for round-robin
}

const CORS_HEADERS = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'POST, GET, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization, x-api-key',
};

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // CORS Preflight
    if (request.method === 'OPTIONS') {
      return new Response(null, { status: 204, headers: CORS_HEADERS });
    }

    // Health Check
    if (request.method === 'GET' && (url.pathname === '/' || url.pathname === '/mcp')) {
      return new Response('Stateless MCP Server Running', { headers: CORS_HEADERS });
    }

    // Main MCP Endpoint (JSON-RPC)
    if (url.pathname === '/mcp' && request.method === 'POST') {
      const server = new McpServer({ name: 'cf-mcp-server', version: '1.0.0' });
      
      // REGISTER TOOLS HERE
      server.tool("hello", "Say hello", { name: z.string() }, async ({ name }) => {
        return { content: [{ type: "text", text: `Hello ${name} from Cloudflare!` }] };
      });

      let body: any;
      try {
         body = await request.json();
      } catch {
         return Response.json({ jsonrpc: "2.0", id: null, error: { code: -32700, message: "Parse error" }}, { headers: CORS_HEADERS });
      }

      const processMessage = async (msg: any) => {
        const { jsonrpc, id, method, params } = msg;
        
        if (method === "initialize") {
           return { jsonrpc: "2.0", id, result: { protocolVersion: "2024-11-05", capabilities: {}, serverInfo: { name: "cf-mcp", version: "1.0.0" } } };
        }
        
        if (method === "notifications/initialized") return null;
        
        if (method === "tools/list") {
           const registeredTools = (server as any)._registeredTools;
           const tools = [];
           if (registeredTools instanceof Map) {
             for (const [name, tool] of registeredTools) tools.push({ name, description: tool.description, inputSchema: tool.inputSchema });
           } else if (registeredTools) {
             for (const name of Object.keys(registeredTools)) tools.push({ name, description: registeredTools[name].description, inputSchema: registeredTools[name].inputSchema });
           }
           return { jsonrpc: "2.0", id, result: { tools } };
        }
        
        if (method === "tools/call") {
           const toolName = params?.name;
           const args = params?.arguments || {};
           
           const registeredTools = (server as any)._registeredTools;
           const tool = registeredTools instanceof Map ? registeredTools.get(toolName) : registeredTools?.[toolName];
           if (!tool) return { jsonrpc: "2.0", id, error: { code: -32601, message: `Tool not found` } };
           
           const handler = tool.handler || tool.execute || tool.callback;
           try {
             const result = await handler(args, {});
             return { jsonrpc: "2.0", id, result };
           } catch (err: any) {
             return { jsonrpc: "2.0", id, result: { content: [{ type: "text", text: `Error: ${err.message}` }], isError: true } };
           }
        }
        return { jsonrpc: "2.0", id, error: { code: -32601, message: "Method not found" } };
      };

      // Handle Batch Requests (Array) or Single Requests (Object)
      if (Array.isArray(body)) {
         const responses = [];
         for (const msg of body) {
            const res = await processMessage(msg);
            if (res) responses.push(res);
         }
         return Response.json(responses, { headers: CORS_HEADERS });
      } else {
         const res = await processMessage(body);
         if (!res) return new Response(null, { status: 204, headers: CORS_HEADERS });
         return Response.json(res, { headers: CORS_HEADERS });
      }
    }
    return new Response("Not Found", { status: 404 });
  }
};
```

## 4. Resolve Incompatible Node.js Dependencies
Cloudflare Workers do not support modules that rely heavily on Node.js native `net`, `tls`, or `fs` (like Axios). 
If a tool uses `axios`, use `esbuild` to alias it to a no-op or native `fetch`.
In `package.json`:
```json
"scripts": {
  "build": "esbuild src/worker.ts --bundle --outfile=dist/worker.js --alias:axios=./src/agnost-stub.ts"
}
```

## 5. API Key Rotation (Round-Robin)
For rate-limiting protection, read a comma-separated list of keys from `env.API_KEYS` and cycle through them on 429 responses. 
```bash
npx wrangler secret put API_KEYS
```

## 6. Deployment
Deploy to Cloudflare edge:
```bash
npx wrangler deploy
```
Provide the user with the deployed URL (`https://<worker-name>.<username>.workers.dev/mcp`) for ChatGPT integration.
