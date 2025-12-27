# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an MCP (Model Context Protocol) server that provides OAuth2-authenticated access to Oura Ring health data. It's designed for integration with MCP-compatible clients like Poke.

## Common Commands

```bash
# Development
npm run dev          # Start with auto-reload (tsx watch)
npm run build        # Compile TypeScript to dist/
npm start            # Run compiled server

# Quality checks
npm run typecheck    # Type check without emitting
npm run lint         # ESLint check
npm test             # Run Jest tests
npm run test:integration  # Integration tests only
npm run test:oauth        # OAuth-specific tests
```

## Architecture

### Request Flow

1. **Entry Point** (`src/index.ts`): Express server with middleware chain (helmet, CORS, rate limiting)
2. **Authentication** (`src/middleware/auth.ts`): Bearer token validation via `AUTH_TOKEN` env var
3. **MCP Protocol** (`src/mcp/server.ts`): Handles both SSE (streaming) and Streamable HTTP transports
4. **Tool Execution** (`src/mcp/tools.ts`): 9 MCP tools that call the Oura client
5. **Oura API** (`src/oura/client.ts`): Authenticated axios client with rate limit tracking

### OAuth2 Flow

The server implements OAuth2 with PKCE for secure token exchange:
- `src/oauth/handler.ts`: Handles `/oauth/authorize` and `/oauth/callback`, manages PKCE state
- `src/oauth/tokens.ts`: Token persistence with AES-256-GCM encryption to `tokens.json`
- Tokens are refreshed automatically when expiring (5-minute buffer)

### MCP Transport Modes

The `/sse` endpoint supports two modes:
1. **GET request**: Establishes SSE connection, returns session ID, tool responses stream via SSE events
2. **POST with JSON-RPC body**: Streamable HTTP mode - direct request/response (used by Poke)

Tool calls come via `/message` (SSE mode) or `/sse` POST (Streamable HTTP).

### Key Patterns

- **Caching**: `src/utils/cache.ts` provides TTL-based caching for API responses (5-min default, 1-hour for personal info)
- **Validation**: Joi schemas in `src/utils/validation.ts` validate all tool parameters
- **Types**: `src/oura/types.ts` contains both internal types and Oura API response types

## Environment Variables

Required:
- `AUTH_TOKEN`: MCP client authentication
- `OURA_CLIENT_ID`, `OURA_CLIENT_SECRET`, `OURA_REDIRECT_URI`: OAuth app credentials
- `TOKEN_ENCRYPTION_KEY`: AES-256 key for token encryption

Optional:
- `PORT`: Server port (default: 3001)
- `NODE_ENV`: development/production
- `CORS_ORIGIN`: Allowed origins (default: *)
- `LOG_LEVEL`: error/warn/info/debug

## MCP Tools

All tools are defined in `src/mcp/tools.ts` with handlers that:
1. Validate input with Joi schemas
2. Check cache for existing data
3. Call Oura API via client functions
4. Format response as JSON text

Tools: `get_personal_info`, `get_sleep_summary`, `get_readiness_score`, `get_activity_summary`, `get_heart_rate`, `get_workouts`, `get_sleep_detailed`, `get_tags`, `get_health_insights`

## Testing with Real Oura Account

Before submitting PRs, test with a real Oura account:
1. Set up OAuth app at https://cloud.ouraring.com/oauth/applications
2. Configure environment variables
3. Visit `/oauth/authorize` to authenticate
4. Test tool calls via MCP client
