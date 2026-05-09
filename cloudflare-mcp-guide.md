---
name: build-cloudflare-mcp-stateless
description: Build and deploy a stateless Model Context Protocol (MCP) server on Cloudflare Workers for ChatGPT Custom Apps. Use this skill when migrating an existing MCP server to Cloudflare, or building a new serverless MCP from scratch.
---

# Skill: Build Cloudflare MCP Server (Stateless)

## 🎯 Purpose and Context
This skill provides the exact methodology to deploy an MCP server to **Cloudflare Workers**. 
Standard MCP implementations rely on stateful transports (`stdio` or Server-Sent Events). However, **ChatGPT Custom Apps require Stateless HTTP (JSON-RPC POST requests)**. 

To achieve this on Cloudflare Workers (which enforce a stateless, ephemeral execution model), you must bypass the standard MCP transport layers and implement a lightweight JSON-RPC router that directly interfaces with the `@modelcontextprotocol/sdk`'s internal registry.

---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Initialize the Environment
If starting from scratch or adding to an existing repo, ensure the following dependencies are installed:
```bash
npm install @modelcontextprotocol/sdk zod
npm install -D wrangler typescript @cloudflare/workers-types esbuild
```

### Step 2: Configure Cloudflare Worker (`wrangler.toml`)
Create a `wrangler.toml` file at the root of the worker directory.
**Crucial:** Enable `nodejs_compat` to allow the MCP SDK to run in the V8 isolate.

```toml
name = "my-mcp-server"
main = "src/worker.ts"
compatibility_date = "2025-05-01"
compatibility_flags = ["nodejs_compat"]

[observability]
enabled = true
```

### Step 3: TypeScript Configuration (`tsconfig.json`)
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

### Step 4: The Stateless Adapter (`src/worker.ts`)
**CRITICAL ARCHITECTURE NOTE:** Because we do not use the SDK's built-in transports, we must manually parse the JSON-RPC request and route it. To serve `tools/list` and `tools/call`, we must extract the tools from `server._registeredTools`.

Use the following exact boilerplate for the Worker entry point:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { z } from 'zod';

export interface Env {
  // Bind your Cloudflare Secrets here
  API_KEY?: string;
}

const CORS_HEADERS = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'POST, GET, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization, x-api-key',
};

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // 1. CORS Preflight
    if (request.method === 'OPTIONS') {
      return new Response(null, { status: 204, headers: CORS_HEADERS });
    }

    // 2. Health Check
    if (request.method === 'GET' && (url.pathname === '/' || url.pathname === '/mcp')) {
      return new Response('Stateless MCP Server Running', { headers: CORS_HEADERS });
    }

    // 3. Main MCP Endpoint (JSON-RPC)
    if (url.pathname === '/mcp' && request.method === 'POST') {
      
      // Initialize Server
      const server = new McpServer({ name: 'cf-mcp-server', version: '1.0.0' });
      
      // ==========================================
      // REGISTER YOUR TOOLS HERE
      // ==========================================
      server.tool("hello", "Say hello", { name: z.string() }, async ({ name }) => {
        return { content: [{ type: "text", text: `Hello ${name} from Cloudflare Edge!` }] };
      });
      // ==========================================

      let body: any;
      try {
         body = await request.json();
      } catch {
         return Response.json({ jsonrpc: "2.0", id: null, error: { code: -32700, message: "Parse error" }}, { headers: CORS_HEADERS });
      }

      // Helper to process single JSON-RPC message
      const processMessage = async (msg: any) => {
        const { jsonrpc, id, method, params } = msg;
        
        if (method === "initialize") {
           return { jsonrpc: "2.0", id, result: { protocolVersion: "2024-11-05", capabilities: {}, serverInfo: { name: "cf-mcp", version: "1.0.0" } } };
        }
        
        if (method === "notifications/initialized") return null;
        
        if (method === "tools/list") {
           // Hack: Extract tools from SDK internal registry
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

      // ChatGPT sends batch requests (Array) or single requests (Object)
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

---

## ⚠️ Known Issues & Solutions (Read Carefully)

### 1. Incompatible Node.js Dependencies (Axios, native modules)
Cloudflare Workers do not support modules that rely heavily on Node.js native `net`, `tls`, or `fs`. 
If a tool uses `axios`, it will fail at runtime.
**Fix:** Use `esbuild` to alias `axios` to a no-op or use `cross-fetch`.
In `package.json`:
```json
"scripts": {
  "build": "esbuild src/worker.ts --bundle --outfile=dist/worker.js --alias:axios=./src/agnost-stub.ts"
}
```

### 2. Batch Request Failures
ChatGPT often wraps JSON-RPC requests in an Array. If your worker only parses objects, it will crash.
**Fix:** The boilerplate above already handles `Array.isArray(body)`. Do not remove this logic.

### 3. API Key Rotation / Load Balancing
If the tool requires rate-limiting protection, use environment variables to pass a comma-separated list of keys, and implement a Round-Robin state per request. Read from `env.EXA_API_KEYS` dynamically inside the `fetch` handler.

---

## 🚀 Deployment & ChatGPT Integration
1. **Deploy:** `npx wrangler deploy`
2. **Secrets:** `npx wrangler secret put API_KEY`
3. **ChatGPT Setup:**
   - Go to GPT Builder -> Actions -> Create new action.
   - Set the URL to your worker's endpoint: `https://<worker-name>.<username>.workers.dev/mcp`
   - You **do not** need an OpenAPI schema if using the MCP standard (ChatGPT detects MCP via JSON-RPC).
