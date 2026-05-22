# MCP Builder

> A Claude Code skill for building production MCP servers with **dual transport** (stdio + Streamable HTTP) and **OAuth 2.1** for remote use with [Claude Co-Work](https://claude.ai).

`mcp-builder` codifies the patterns I use to ship MCP servers that work with the Claude Code CLI (stdio), with Claude Co-Work (HTTP + OAuth), and with any MCP-compatible client — from a single shared server factory.

---

## Install

Clone this repo, then drop the skill into your `~/.claude/skills/` directory:

```bash
git clone https://github.com/addiplus/mcp-builder.git
cp -r mcp-builder ~/.claude/skills/
```

Once installed, Claude Code auto-activates the skill on prompts like "build an MCP server", "add OAuth to my MCP", "deploy MCP to Vercel".

---

## What's covered

The skill walks 5 phases of MCP server construction:

1. **Design** — what your server exposes, tool naming, project structure
2. **Core** — transport-agnostic server factory, Zod-validated tool definitions, shared types
3. **HTTP transport** — Streamable HTTP via Hono + Vercel serverless
4. **OAuth 2.1** — full ceremony with PKCE, stateless HMAC auth codes, 5 endpoints (`/authorize`, `/token`, `/register`, `/.well-known/oauth-authorization-server`, `/.well-known/oauth-protected-resource`)
5. **Testing** — vitest suite, mock target server, claim verification

---

## Companion reference

[`addiplus/mcp-cookie-auth-reference`](https://github.com/addiplus/mcp-cookie-auth-reference) is the canonical public reference implementing every pattern in this skill. It targets cookie-authenticated web systems that don't expose a public API — a common enterprise pattern that's hard to do well.

- 1,896 LOC TypeScript
- 24/24 vitest tests passing
- Hono mock target
- MIT licensed

Read the skill for patterns; clone the reference for implementation.

---

## Key insights baked in

- **Transport agnosticism** — the server factory is created once. Tools are defined once. stdio and HTTP each get a fresh server instance.
- **OAuth resource field needs the `/mcp` path** — common gotcha that breaks Claude Co-Work auth registration.
- **Bearer must be capitalized** in the `Authorization` header — non-capitalized `bearer` fails in some clients.
- **Stateless HMAC auth codes** scale better than database-backed codes for serverless Vercel deployments.
- **Vercel `.well-known` routing** requires a `vercel.json` rewrite — the framework default puts it under `/api/`.

---

## When to use this skill vs. the reference repo

| You want to... | Use |
|----------------|-----|
| Understand the pattern before writing code | this skill |
| See exact TypeScript implementations | [`addiplus/mcp-cookie-auth-reference`](https://github.com/addiplus/mcp-cookie-auth-reference) |
| Both | this skill loads first; reference repo on demand |

---

## License

MIT. See [LICENSE](LICENSE).

## Author

Built by [Joe Smith](https://github.com/addiplus). Companion repos:

- [`addiplus/mcp-cookie-auth-reference`](https://github.com/addiplus/mcp-cookie-auth-reference) — TypeScript MCP server reference
- [`addiplus/claude-skill-pack`](https://github.com/addiplus/claude-skill-pack) — Bun-native Claude Code dev loop
- [`addiplus/prd-engine`](https://github.com/addiplus/prd-engine) — Claude Code plugin for generating agent-executable PRDs
