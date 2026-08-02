# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-05-15

### Added

- Initial public release.
- Primary SKILL.md (~780 lines) covering 5 phases of MCP server construction:
  1. **Design**: what your server exposes, tool naming, project structure
  2. **Core**: transport-agnostic server factory, Zod-validated tools, shared types
  3. **HTTP transport**: Streamable HTTP via Hono on Vercel serverless
  4. **OAuth 2.1**: full ceremony with PKCE, stateless HMAC auth codes, 5 endpoints
  5. **Testing**: vitest suite, mock target server, claim verification
- Reference patterns drawn from production work on the Loom MCP Server.
- Key insights baked into the skill: transport agnosticism, OAuth resource field `/mcp` path requirement, Bearer capitalization, stateless HMAC scaling for serverless, Vercel `.well-known` routing gotcha.
- Companion implementation: [`addiplus/mcp-cookie-auth-reference`](https://github.com/addiplus/mcp-cookie-auth-reference): full TypeScript MCP server with cookie-authenticated session wrapping, Hono mock target, 24/24 vitest tests passing, MIT-licensed.
