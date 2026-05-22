---
name: mcp-builder
description: Build production MCP servers with dual transport (stdio + HTTP) and OAuth 2.1 for Claude Co-Work. Covers server design, tool definitions, Vercel deployment, and the full OAuth ceremony. Use when building a new MCP server, adding HTTP transport, adding OAuth, or deploying to Vercel. Triggers on "MCP server", "build MCP", "create MCP", "Co-Work", "mcp-builder", "remote MCP".
metadata:
  priority: 80
  filePattern:
    - "**/api/mcp.ts"
    - "**/src/server-factory.ts"
    - "**/src/tools.ts"
    - "**/.well-known/**"
    - "**/oauth/**"
  bashPattern:
    - "mcp"
  promptSignals:
    phrases:
      - "MCP server"
      - "build MCP"
      - "create MCP"
      - "Co-Work"
      - "remote MCP"
      - "MCP OAuth"
      - "Streamable HTTP"
    allOf: []
    anyOf: []
    noneOf: []
    minScore: 6
---

# MCP Server Builder

Build production MCP servers that work with **Claude Code CLI** (stdio), **Claude Co-Work** (HTTP + OAuth 2.1), and any MCP-compatible client.

## Reference Implementation

The companion repo [`addiplus/mcp-cookie-auth-reference`](https://github.com/addiplus/mcp-cookie-auth-reference) is the canonical public reference for every pattern in this skill:

- **Source**: full TypeScript MCP server with cookie-authenticated session wrapping
- **Stack**: Hono mock target, Zod-validated tools, vitest suite (24/24 passing)
- **Blog walk-through**: [mcp-cookie-auth-blog.vercel.app](https://mcp-cookie-auth-blog.vercel.app)
- **Architecture**: Dual transport (stdio + Streamable HTTP), stateless OAuth 2.1, Vercel serverless

Read the reference files when you need exact implementation details. This skill teaches the *patterns*; the reference repo teaches the *implementation*.

## Architecture Overview

```
                    +-----------------+
                    | Data Source API  |
                    | (Loom, Notion,  |
                    |  GitHub, etc.)  |
                    +--------+--------+
                             |
                    +--------+--------+
                    |  Server Factory  |  <-- creates fresh server per request
                    |  (tools, config) |
                    +---+----------+--+
                        |          |
              +---------+--+  +----+----------+
              | stdio      |  | HTTP handler  |
              | (index.ts) |  | (api/mcp.ts)  |
              +-----+------+  +----+----------+
                    |              |
            +-------+--+    +-----+--------+
            |Claude Code|    | OAuth 2.1    |
            |  CLI      |    | (5 endpoints)|
            +----------+    +-----+--------+
                                   |
                           +-------+-------+
                           | Claude Co-Work|
                           | (claude.ai)   |
                           +---------------+
```

**Key principle**: The server factory is transport-agnostic. Tools are defined ONCE, shared across stdio and HTTP. Each transport entry point creates a fresh server instance and connects it.

## Phase 1: Design Your MCP Server

Before writing code, answer these questions:

### 1.1 What does this server expose?
- What **data source** are you wrapping? (API, database, file system, SaaS tool)
- What **tools** (actions) should it provide? (list, search, get, create, update, delete)
- What **auth** does the data source need? (API key, cookie, OAuth token, none)

### 1.2 Name your tools well
MCP tools should be:
- **Verb-noun**: `list_recordings`, `search_docs`, `get_transcript`
- **Descriptive**: The description is what the LLM reads to decide when to use the tool
- **Bounded**: Include `limit` params with sensible defaults (25) and max caps (100)

### 1.3 Project structure

```
my-mcp-server/
├── src/
│   ├── index.ts              # stdio entry point
│   ├── server-factory.ts     # transport-agnostic factory
│   ├── tools.ts              # tool definitions (Zod schemas)
│   ├── client.ts             # data source API client
│   ├── config.ts             # environment validation
│   ├── types.ts              # shared types
│   ├── schemas.ts            # Zod response validation (optional)
│   ├── formatters.ts         # response formatting helpers
│   ├── auth-logic.ts         # bearer token validation (Phase 4)
│   ├── oauth.ts              # OAuth utilities (Phase 4)
│   └── crypto-utils.ts       # HMAC, secure compare (Phase 4)
├── api/                      # Vercel serverless functions (Phase 3)
│   ├── mcp.ts                # Streamable HTTP handler
│   ├── health.ts             # GET /health
│   ├── oauth/                # OAuth 2.1 endpoints (Phase 4)
│   │   ├── authorize.ts
│   │   ├── token.ts
│   │   └── register.ts
│   └── well-known/           # RFC metadata (Phase 4)
│       ├── oauth-protected-resource.ts
│       └── oauth-authorization-server.ts
├── vercel.json               # routes + config (Phase 3)
├── tsconfig.json
├── package.json
└── vitest.config.ts
```

## Phase 2: Build Core Server

### 2.1 Dependencies

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.27.1",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.4"
  }
}
```

Only TWO runtime dependencies. The MCP SDK and Zod (for input schemas).

### 2.2 Server factory pattern

```typescript
// src/server-factory.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { YourClient } from "./client.js";
import { registerTools } from "./tools.js";

export interface ServerConfig {
  apiKey: string;        // whatever your data source needs
  // ... other config
}

export function createServer(config: ServerConfig) {
  const server = new McpServer({
    name: "your-server-name",
    version: "1.0.0",
  });

  const client = new YourClient(config.apiKey);
  registerTools(server, client);

  return { server, client };
}
```

**CRITICAL**: Each call to `createServer()` MUST produce FRESH instances.
For HTTP transport, each request creates its own server (stateless).

### 2.3 Tool definitions with Zod schemas

```typescript
// src/tools.ts
import { z } from "zod";

export function registerTools(server: any, client: any): void {
  // Tool with no input parameters
  server.tool(
    "tool_name",
    "Clear description of what this tool does and when to use it.",
    async () => {
      try {
        const result = await client.doSomething();
        return { content: [{ type: "text", text: JSON.stringify(result) }] };
      } catch (err: any) {
        return { content: [{ type: "text", text: `Error: ${err.message}` }], isError: true };
      }
    },
  );

  // Tool with input parameters (Zod schema)
  server.tool(
    "search_items",
    "Search for items by query string. Returns matching items with metadata.",
    {
      query: z.string().min(1).describe("Search query"),
      limit: z.number().int().min(1).max(100).default(25).describe("Max results"),
    },
    async ({ query, limit }: { query: string; limit: number }) => {
      try {
        const results = await client.search(query, { limit });
        return { content: [{ type: "text", text: formatResults(results) }] };
      } catch (err: any) {
        return { content: [{ type: "text", text: `Error: ${err.message}` }], isError: true };
      }
    },
  );
}
```

**Tool patterns**:
- ALWAYS wrap handlers in try/catch
- NEVER leak auth credentials in error messages
- Use `isError: true` for error responses
- Add pagination with `limit` + `offset` for list tools
- Use `z.string().describe()` -- the description helps the LLM understand the parameter

### 2.4 stdio entry point

```typescript
// src/index.ts
#!/usr/bin/env node
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { loadConfig } from "./config.js";
import { createServer } from "./server-factory.js";

const config = loadConfig(); // throws on missing env vars
const { server } = createServer(config);

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Server running on stdio"); // stderr, not stdout!
}

main().catch((error) => {
  console.error("Fatal:", error);
  process.exit(1);
});
```

**CRITICAL**: Use `console.error()` for logging, NEVER `console.log()`.
stdio MCP uses stdout for JSON-RPC. Any non-JSON on stdout breaks the protocol.

### 2.5 Connect to Claude Code

Add to your project's `.mcp.json` (or the user's `~/.claude.json`):

```json
{
  "mcpServers": {
    "your-server": {
      "command": "node",
      "args": ["path/to/build/index.js"],
      "env": {
        "YOUR_API_KEY": "..."
      }
    }
  }
}
```

**At this point, your MCP server works with Claude Code and any stdio-compatible client.**

## Phase 3: Add HTTP Transport (Vercel)

### 3.1 The Vercel handler

```typescript
// api/mcp.ts
import type { IncomingMessage, ServerResponse } from "node:http";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { createServer } from "../src/server-factory.js";

export default async function handler(req: IncomingMessage, res: ServerResponse) {
  // Security headers
  res.setHeader("X-Content-Type-Options", "nosniff");
  res.setHeader("X-Frame-Options", "DENY");
  res.setHeader("Cache-Control", "no-store");

  // CORS preflight
  if (req.method === "OPTIONS") {
    res.statusCode = 204;
    res.end();
    return;
  }

  // Method gate -- Streamable HTTP only supports POST
  // GET is for SSE (we don't support it), return proper 405
  if (req.method !== "POST") {
    res.statusCode = 405;
    res.setHeader("Allow", "POST, OPTIONS");
    res.setHeader("Content-Type", "text/plain");
    res.end("Method Not Allowed. Use POST.");
    return;
  }

  // Auth -- see Phase 4 for OAuth, or use simple bearer token:
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith("Bearer ") || authHeader.slice(7) !== process.env.MCP_AUTH_TOKEN) {
    res.statusCode = 401;
    res.setHeader("Content-Type", "application/json");
    res.end(JSON.stringify({ error: "Unauthorized" }));
    return;
  }

  // Read body
  const chunks: Buffer[] = [];
  let size = 0;
  const MAX_BODY = 1_048_576; // 1MB
  for await (const chunk of req) {
    size += chunk.length;
    if (size > MAX_BODY) {
      res.statusCode = 413;
      res.end("Request body too large");
      return;
    }
    chunks.push(chunk);
  }
  const body = Buffer.concat(chunks).toString("utf-8");
  const parsedBody = JSON.parse(body);

  // Create fresh server + transport per request (stateless)
  const config = loadConfigFromEnv();
  const { server } = createServer(config);

  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined, // stateless -- no sessions
    enableJsonResponse: true,      // JSON responses, not SSE
  });

  await server.connect(transport);
  await transport.handleRequest(req, res, parsedBody);
  await server.close();
}
```

**Key decisions**:
- `sessionIdGenerator: undefined` = stateless (no session affinity needed on serverless)
- `enableJsonResponse: true` = JSON responses instead of SSE streams
- Fresh server per request = safe for concurrent serverless invocations
- 1MB body limit = prevents memory exhaustion attacks

### 3.2 Health endpoint

```typescript
// api/health.ts
export default function handler(req: IncomingMessage, res: ServerResponse) {
  res.statusCode = 200;
  res.setHeader("Content-Type", "application/json");
  res.end(JSON.stringify({ status: "ok", version: "1.0.0" }));
}
```

### 3.3 vercel.json

```json
{
  "version": 2,
  "regions": ["iad1"],
  "functions": {
    "api/**/*.ts": {
      "memory": 256,
      "maxDuration": 30
    }
  },
  "rewrites": [
    { "source": "/mcp", "destination": "/api/mcp" },
    { "source": "/health", "destination": "/api/health" }
  ]
}
```

### 3.4 Environment variables

```bash
# Set via Vercel dashboard or CLI
MCP_AUTH_TOKEN=<generate-a-strong-random-token>
YOUR_API_KEY=<your-data-source-credential>
```

**At this point, your MCP server works over HTTP with bearer token auth.**

## Phase 4: Add OAuth 2.1 for Claude Co-Work

This is the full OAuth 2.1 ceremony that Claude Co-Work requires to connect to remote MCP servers. It implements:
- **RFC 9728**: Protected Resource Metadata
- **RFC 8414**: Authorization Server Metadata
- **RFC 7591**: Dynamic Client Registration
- **RFC 7636**: PKCE with S256
- **RFC 8707**: Resource Indicators (required by MCP 6/18 spec)

### 4.1 The spec compliance checklist

Claude Co-Work implements the **MCP 6/18 spec** (June 18, 2025). These are MANDATORY:

| Requirement | Implementation |
|---|---|
| 401 with `WWW-Authenticate: Bearer resource_metadata="<url>"` | auth-logic.ts |
| `GET /.well-known/oauth-protected-resource` returns resource metadata | well-known/ |
| `resource` field = canonical MCP endpoint URL (with `/mcp` path) | **CRITICAL** |
| `authorization_servers` array in resource metadata | well-known/ |
| `GET /.well-known/oauth-authorization-server` returns auth server metadata | well-known/ |
| `POST /oauth/register` for dynamic client registration | oauth/register.ts |
| `GET /oauth/authorize` with PKCE (code_challenge + S256) | oauth/authorize.ts |
| `POST /oauth/token` exchanges code + code_verifier for access_token | oauth/token.ts |
| `token_type` should be `"Bearer"` (capital B) — case-insensitive per RFC 6749 §4.2.2/§5.1, but Co-Work's parser is strict in practice | **CRITICAL** |
| 405 for GET /mcp returns plain text + `Allow` header (not JSON-RPC) | api/mcp.ts |

### 4.2 Stateless HMAC-signed auth codes

**This is the key architectural insight**: you don't need Redis/KV to implement OAuth on serverless. Authorization codes are HMAC-signed payloads:

```typescript
// src/oauth.ts
import { createHmac, randomBytes } from "crypto";

export function getIssuer(): string {
  return (process.env.VERCEL_PROJECT_PRODUCTION_URL
    ? `https://${process.env.VERCEL_PROJECT_PRODUCTION_URL}`
    : process.env.MCP_SERVER_URL || "http://localhost:3000"
  );
}

// Sign an auth code payload with HMAC-SHA256
export function createSignedAuthCode(
  payload: { code_challenge: string; redirect_uri: string; client_id: string },
  secret: string,
  ttlSeconds = 300,
): string {
  const nonce = randomBytes(16).toString("hex");
  const expiry = Math.floor(Date.now() / 1000) + ttlSeconds;
  const data = JSON.stringify({ ...payload, expiry, nonce });
  const sig = createHmac("sha256", secret).update(data).digest("hex");
  const raw = JSON.stringify({ data, sig });
  return Buffer.from(raw).toString("base64url");
}

// Verify and decode a signed auth code
export function verifySignedAuthCode(
  code: string,
  secret: string,
): { code_challenge: string; redirect_uri: string; client_id: string } | null {
  try {
    const raw = JSON.parse(Buffer.from(code, "base64url").toString("utf-8"));
    const expected = createHmac("sha256", secret).update(raw.data).digest("hex");
    // Use timing-safe comparison to prevent signature-timing attacks
    // (add `timingSafeEqual` to the crypto import at the top of this file)
    const expectedBuf = Buffer.from(expected, "hex");
    const sigBuf = Buffer.from(raw.sig, "hex");
    if (expectedBuf.length !== sigBuf.length || !timingSafeEqual(expectedBuf, sigBuf)) return null; // tampered

    const payload = JSON.parse(raw.data);
    if (payload.expiry < Math.floor(Date.now() / 1000)) return null; // expired

    return payload;
  } catch {
    return null;
  }
}
```

**Why this works**: `MCP_AUTH_TOKEN` serves as both the HMAC signing key AND the final access token. The authorize endpoint signs the code, the token endpoint verifies it, and then returns `MCP_AUTH_TOKEN` as the access_token. Zero shared state.

### 4.3 Protected Resource Metadata (RFC 9728)

```typescript
// api/well-known/oauth-protected-resource.ts
export default function handler(req, res) {
  const baseUrl = getIssuer();
  res.statusCode = 200;
  res.setHeader("Content-Type", "application/json");
  res.end(JSON.stringify({
    resource: `${baseUrl}/mcp`,           // MUST include /mcp path!
    authorization_servers: [baseUrl],
    bearer_methods_supported: ["header"],
    scopes_supported: ["mcp:tools"],
  }));
}
```

**CRITICAL**: The `resource` field MUST be the canonical URL of the MCP endpoint (`/mcp` path included). If it's just the base URL without `/mcp`, Co-Work may reject it.

### 4.4 Authorization Server Metadata (RFC 8414)

```typescript
// api/well-known/oauth-authorization-server.ts
export default function handler(req, res) {
  const baseUrl = getIssuer();
  res.statusCode = 200;
  res.setHeader("Content-Type", "application/json");
  res.end(JSON.stringify({
    issuer: baseUrl,
    authorization_endpoint: `${baseUrl}/oauth/authorize`,
    token_endpoint: `${baseUrl}/oauth/token`,
    registration_endpoint: `${baseUrl}/oauth/register`,
    response_types_supported: ["code"],
    grant_types_supported: ["authorization_code"],
    code_challenge_methods_supported: ["S256"],
    token_endpoint_auth_methods_supported: ["none"],
    scopes_supported: ["mcp:tools"],
  }));
}
```

### 4.5 Dynamic Client Registration (RFC 7591)

```typescript
// api/oauth/register.ts
import { randomUUID } from "crypto";

export default async function handler(req, res) {
  if (req.method !== "POST") { res.statusCode = 405; res.end(); return; }

  const body = await readBody(req);
  const params = JSON.parse(body);

  const clientId = randomUUID();
  res.statusCode = 201;
  sendJson(res, {
    client_id: clientId,
    client_name: params.client_name || "MCP Client",
    redirect_uris: params.redirect_uris || [],
    grant_types: ["authorization_code"],
    response_types: ["code"],
    token_endpoint_auth_method: "none",
  });
}
```

### 4.6 Authorize Endpoint (PKCE)

```typescript
// api/oauth/authorize.ts
export default async function handler(req, res) {
  // Only GET
  if (req.method !== "GET") { res.statusCode = 405; res.end(); return; }

  const url = new URL(req.url, `https://${req.headers.host}`);
  const responseType = url.searchParams.get("response_type");
  const clientId = url.searchParams.get("client_id");
  const redirectUri = url.searchParams.get("redirect_uri");
  const codeChallenge = url.searchParams.get("code_challenge");
  const codeChallengeMethod = url.searchParams.get("code_challenge_method");
  const state = url.searchParams.get("state");

  // Validate required params
  if (responseType !== "code" || !clientId || !redirectUri || !codeChallenge
      || codeChallengeMethod !== "S256") {
    res.statusCode = 400;
    sendJson(res, { error: "invalid_request" });
    return;
  }

  // Auto-approve (single-tenant) -- create signed auth code
  const code = createSignedAuthCode(
    { code_challenge: codeChallenge, redirect_uri: redirectUri, client_id: clientId },
    process.env.MCP_AUTH_TOKEN!,
  );

  // 302 redirect back to client with code + state
  const redirect = new URL(redirectUri);
  redirect.searchParams.set("code", code);
  if (state) redirect.searchParams.set("state", state);

  res.statusCode = 302;
  res.setHeader("Location", redirect.toString());
  res.end();
}
```

**Auto-approve pattern**: For single-tenant servers (you control who has MCP_AUTH_TOKEN), the authorize endpoint immediately redirects with the code. No login page needed.

### 4.7 Token Endpoint

```typescript
// api/oauth/token.ts
export default async function handler(req, res) {
  if (req.method !== "POST") { res.statusCode = 405; res.end(); return; }

  const body = await readBody(req);
  const params = new URLSearchParams(body);
  const grantType = params.get("grant_type");
  const code = params.get("code");
  const codeVerifier = params.get("code_verifier");
  const redirectUri = params.get("redirect_uri");

  if (grantType !== "authorization_code" || !code || !codeVerifier) {
    res.statusCode = 400;
    sendJson(res, { error: "invalid_request" });
    return;
  }

  // Verify HMAC-signed auth code
  const payload = verifySignedAuthCode(code, process.env.MCP_AUTH_TOKEN!);
  if (!payload) {
    res.statusCode = 400;
    sendJson(res, { error: "invalid_grant", error_description: "Invalid or expired code" });
    return;
  }

  // Verify PKCE: SHA256(code_verifier) must equal stored code_challenge
  const hash = createHash("sha256").update(codeVerifier).digest("base64url");
  if (hash !== payload.code_challenge) {
    res.statusCode = 400;
    sendJson(res, { error: "invalid_grant", error_description: "PKCE verification failed" });
    return;
  }

  // Verify redirect_uri matches
  if (redirectUri && redirectUri !== payload.redirect_uri) {
    res.statusCode = 400;
    sendJson(res, { error: "invalid_grant" });
    return;
  }

  // Return MCP_AUTH_TOKEN as the access_token
  sendJson(res, 200, {
    access_token: process.env.MCP_AUTH_TOKEN,
    token_type: "Bearer",     // case-insensitive per RFC 6749, but Co-Work requires capital B
    expires_in: 86400 * 30,   // 30 days
    scope: "mcp:tools",
  });
}
```

### 4.8 Bearer token validation with WWW-Authenticate

```typescript
// src/auth-logic.ts
import { secureCompare } from "./crypto-utils.js";
import { getIssuer } from "./oauth.js";

export function validateBearerToken(
  authHeader: string | undefined,
  expectedToken: string | undefined,
) {
  const metadataUrl = `${getIssuer()}/.well-known/oauth-protected-resource`;

  if (!authHeader) {
    return {
      valid: false,
      status: 401,
      wwwAuthenticate: `Bearer resource_metadata="${metadataUrl}"`,
      errorBody: { jsonrpc: "2.0", error: { code: -32600, message: "Authentication required" }, id: null },
    };
  }

  if (!authHeader.startsWith("Bearer ")) {
    return { valid: false, status: 401, /* ... */ };
  }

  const token = authHeader.slice(7);
  if (!expectedToken || !secureCompare(token, expectedToken)) {
    return { valid: false, status: 403, /* ... */ };
  }

  return { valid: true };
}
```

**CRITICAL**: The `WWW-Authenticate` header MUST include `resource_metadata="<url>"` per RFC 9728. This is how Co-Work discovers the OAuth flow.

### 4.9 Vercel routes for .well-known

Vercel ignores dot-directories (`.well-known/`). Put handlers in `api/well-known/` and use route rewrites:

```json
{
  "rewrites": [
    { "source": "/.well-known/oauth-protected-resource", "destination": "/api/well-known/oauth-protected-resource" },
    { "source": "/.well-known/oauth-authorization-server", "destination": "/api/well-known/oauth-authorization-server" },
    { "source": "/oauth/authorize", "destination": "/api/oauth/authorize" },
    { "source": "/oauth/token", "destination": "/api/oauth/token" },
    { "source": "/oauth/register", "destination": "/api/oauth/register" },
    { "source": "/mcp", "destination": "/api/mcp" },
    { "source": "/health", "destination": "/api/health" }
  ]
}
```

### 4.10 Connect to Claude Co-Work

1. Deploy to Vercel (git push triggers auto-deploy)
2. Set `MCP_AUTH_TOKEN` in Vercel environment variables
3. In Claude Co-Work settings, add connector with URL: `https://your-server.vercel.app/mcp`
4. Complete the OAuth flow in the browser popup
5. The connector should show as connected

## Phase 5: Testing

### 5.1 What to test

| Layer | Test file | What to verify |
|---|---|---|
| Tools | `src/tools.test.ts` | Each tool handles success + error + null data |
| Server factory | `src/server-factory.test.ts` | Fresh instances, tool registration |
| HTTP handler | `api/mcp.test.ts` | Auth, CORS, method gate, body parsing, MCP protocol |
| OAuth | `src/oauth.test.ts` | HMAC signing, verification, expiry, PKCE |
| Metadata | via HTTP tests | Correct JSON structure, all required fields |

### 5.2 Testing OAuth flow

```typescript
describe("OAuth flow", () => {
  it("should create and verify signed auth codes", () => {
    const code = createSignedAuthCode(
      { code_challenge: "abc", redirect_uri: "https://example.com", client_id: "test" },
      "secret",
    );
    const payload = verifySignedAuthCode(code, "secret");
    expect(payload?.code_challenge).toBe("abc");
  });

  it("should reject expired codes", async () => {
    const code = createSignedAuthCode(payload, "secret", 0); // 0 TTL
    await new Promise(r => setTimeout(r, 1100));
    expect(verifySignedAuthCode(code, "secret")).toBeNull();
  });

  it("should reject tampered codes", () => {
    const code = createSignedAuthCode(payload, "secret");
    expect(verifySignedAuthCode(code, "wrong-secret")).toBeNull();
  });
});
```

## Debugging Playbook

### Check Vercel runtime logs FIRST

```
Use mcp__claude_ai_Vercel__get_runtime_logs to see:
1. Is the OAuth flow reaching your server? (look for the 6-step sequence)
2. Are all endpoints returning success? (200/201/302)
3. Is POST /mcp succeeding after OAuth? (200/202)
```

The expected OAuth flow in logs (in chronological order):
```
GET  /.well-known/oauth-protected-resource        | 200
GET  /.well-known/oauth-authorization-server       | 200
POST /oauth/register                               | 201
GET  /oauth/authorize                              | 302
GET  /.well-known/oauth-authorization-server       | 200  (re-fetch)
POST /oauth/token                                  | 200
POST /mcp                                          | 200  (first MCP call)
```

### Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Co-Work shows `ofid_*` error | `resource` field doesn't include `/mcp` path | Set `resource: "${baseUrl}/mcp"` |
| Browser opens then error | `token_type` is `"bearer"` (lowercase) | Use `"Bearer"` (capital B) |
| GET /mcp confuses client | JSON-RPC error body on 405 | Return plain text + `Allow` header |
| 401 but no OAuth flow starts | Missing `resource_metadata` in WWW-Authenticate | Add `Bearer resource_metadata="<url>"` |
| OAuth flow never reaches server | `.well-known` handlers in dot-directory | Move to `well-known/` + vercel.json rewrites |
| Token endpoint returns 400 | Body parsed as JSON, not form-encoded | Use `new URLSearchParams(body)` |
| PKCE verification fails | Wrong hash algorithm | Must be SHA-256 with base64url encoding |

### What "Authorization with the MCP server failed" means

If the Vercel logs show ALL endpoints returning success but Co-Work still shows this error:
1. The failure is CLIENT-SIDE (Co-Work internal)
2. Remove the connector from Co-Work settings
3. Re-add it fresh
4. If it persists, contact Anthropic support with the `ofid_*` reference

## Spec References

| Spec | What it covers | Required by |
|---|---|---|
| RFC 9728 | Protected Resource Metadata | MCP 6/18 spec |
| RFC 8414 | Authorization Server Metadata | MCP 6/18 spec |
| RFC 7591 | Dynamic Client Registration | MCP 6/18 spec |
| RFC 7636 | PKCE (Proof Key for Code Exchange) | OAuth 2.1 |
| RFC 8707 | Resource Indicators | MCP 6/18 spec |
| MCP 3/26 | Base MCP authorization spec | Claude Co-Work |
| MCP 6/18 | Updated spec with RFC 9728 requirement | Claude Co-Work (current) |

The MCP authorization spec is at:
`https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization`
