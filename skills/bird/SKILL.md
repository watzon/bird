---
name: bird
description: Use when working on the bird X/Twitter CLI — tweeting, replying, reading, searching, credential resolution, engine switching, testing, and binary compilation.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [x, twitter, cli, typescript, graphql, sweetistics]
    related_skills: [github-pr-workflow, codebase-best-practices]
---

# bird 🐦 — Fast X CLI

## Overview

`bird` is a TypeScript CLI for interacting with X/Twitter — posting tweets, replying, reading, searching, checking mentions, and reading **X Articles** (long-form posts). It supports two transport engines:

- **GraphQL** — talks directly to X's internal GraphQL API using browser cookies (Chrome or Firefox on macOS)
- **Sweetistics** — uses a SaaS API key for posting without local cookies

The project lives at `~/Projects/personal/bird/`. It's a personal tool, not a published package.

## When to Use

- You're implementing a new command or feature (like, retweet, bookmark, follow)
- You're fixing bugs in credential resolution, tweet parsing, or error handling
- You're refreshing GraphQL query IDs (they rotate frequently)
- You're running tests or adding test coverage
- You're building the binary with bun

## Project Structure

```
bird/
├── src/
│   ├── index.ts                    # CLI entry point (commander)
│   └── lib/
│       ├── twitter-client.ts       # GraphQL API client (tweet, reply, read, search, etc.)
│       ├── sweetistics-client.ts   # Sweetistics REST API client
│       ├── cookies.ts              # Credential resolution (Chrome/Firefox/env/flags)
│       ├── extract-tweet-id.ts     # Parse tweet ID from URL or raw ID string
│       └── query-ids.json          # Rotating GraphQL query IDs (refreshed via script)
├── tests/
│   ├── cli.test.ts                 # CLI integration tests
│   ├── cookies.test.ts             # Credential resolution tests
│   ├── sweetistics-client.test.ts  # Sweetistics client tests
│   └── twitter-client.test.ts      # GraphQL client tests
├── scripts/
│   └── update-query-ids.ts         # Refresh GraphQL query IDs from X
├── docs/
│   └── releasing.md                # Release instructions
├── README.md
├── package.json
├── tsconfig.json
├── vitest.config.ts
└── biome.json
```

## Commands Reference

| Command | Args | Flags | Description |
|---------|------|-------|-------------|
| `tweet` | `<text>` | `--media`, `--alt` | Post a tweet |
| `reply` | `<tweet-id-or-url> <text>` | `--media`, `--alt` | Reply to a tweet |
| `read` | `<tweet-id-or-url>` | `--json`, `--article` | Read a tweet (use `--article` for full X Article content) |
| `replies` | `<tweet-id-or-url>` | `--json` | List replies |
| `thread` | `<tweet-id-or-url>` | `--json` | Show conversation thread |
| `search` | `<query>` | `-n <count>`, `--json` | Search tweets |
| `mentions` | — | `-n <count>`, `--json` | Mentions of @clawdbot |
| `whoami` | — | — | Show logged-in account |
| `check` | — | — | Verify credentials |

### Global Options

| Option | Env Var | Description |
|--------|---------|-------------|
| `--auth-token <token>` | `AUTH_TOKEN`, `TWITTER_AUTH_TOKEN` | Twitter auth_token cookie |
| `--ct0 <token>` | `CT0`, `TWITTER_CT0` | Twitter ct0 cookie |
| `--chrome-profile <name>` | — | Chrome profile for cookie extraction |
| `--firefox-profile <name>` | — | Firefox profile for cookie extraction |
| `--engine <graphql\|sweetistics\|auto>` | `BIRD_ENGINE` | Transport engine |
| `--sweetistics-api-key <key>` | `SWEETISTICS_API_KEY` | Sweetistics API key |
| `--sweetistics-base-url <url>` | `SWEETISTICS_BASE_URL` | Sweetistics base URL |
| `--media <path>` (repeatable) | — | Attach media (up to 4 images or 1 video) |
| `--alt <text>` (repeatable) | — | Alt text for media |

### Config Precedence

CLI flags > environment variables > `.birdrc.json5` (project) > `~/.config/bird/config.json5` (global)

## Development Workflow

### Setup

```bash
cd ~/Projects/personal/bird
pnpm install
```

### Build & Run

```bash
pnpm build              # tsc compile to dist/
pnpm dev tweet "test"   # Run via tsx (no build needed)
pnpm bird tweet "test"  # Build + run
pnpm run binary         # Compile standalone binary with bun
```

### Test

```bash
pnpm test               # vitest run
pnpm test:watch         # vitest watch mode
pnpm test -- --coverage # with coverage
```

### Lint & Format

```bash
pnpm lint               # biome check
pnpm lint:fix           # biome check --write
pnpm format             # biome format --write
```

### Refresh GraphQL Query IDs

X rotates their GraphQL query IDs periodically. When you get 404s or "Unknown query" errors:

```bash
pnpm run graphql:update   # Runs scripts/update-query-ids.ts
```

This writes fresh IDs to `src/lib/query-ids.json`.

## Engine Selection

The engine determines how `bird` communicates with X:

1. **graphql** (default) — Uses X's internal GraphQL API. Requires browser cookies (`auth_token` + `ct0`) from a logged-in X session. Works on macOS with Chrome/Firefox cookie extraction. Rate-limited aggressively (expect 429s).

2. **sweetistics** — Uses the Sweetistics SaaS API. Requires `SWEETISTICS_API_KEY`. No cookies needed. Supports media uploads. 15s timeout on all calls. Conversation calls (`thread`, `replies`) use `force=true` to bypass cache.

3. **auto** — Sweetistics if API key is available, otherwise GraphQL.

### Cookie Resolution Order (GraphQL mode)

1. `--auth-token` / `--ct0` CLI flags
2. `AUTH_TOKEN` / `CT0` env vars (also `TWITTER_AUTH_TOKEN` / `TWITTER_CT0`)
3. Firefox cookies (`~/Library/Application Support/Firefox/Profiles/<profile>/cookies.sqlite`)
4. Chrome cookies (`~/Library/Application Support/Google/Chrome/<Profile>/Cookies` + WAL/SHM)

Cookie sources can be disabled via config: `allowChrome: false`, `allowFirefox: false`.

## Architecture Notes

### X Articles (long-form posts)

X Articles are rendered using the `TweetResultByRestId` GraphQL query with `fieldToggles: { withArticleRichContentState: true }`. The article content uses Draft.js format:

- **`content_state.blocks[]`** — text blocks with `type`: `unstyled`, `header-one`, `header-two`, `blockquote`, `unordered-list-item`, `atomic` (media)
- **`content_state.entityMap[]`** — entities: `MARKDOWN` (code blocks), `MEDIA` (images), `LINK` (URLs)
- **`summary_text`** — AI-generated summary
- **`cover_media`** — cover image

Use `bird read --article <id>` to fetch full article content. Without `--article`, `bird read` shows the title, cover, and preview text.

The `TweetResultByRestId` query puts user data (`screen_name`, `name`) under `user_results.result.core` instead of `user_results.result.legacy` (like `TweetDetail`). The `mapTweetResult` function handles both response formats.

**Query ID:** `TweetResultByRestId` in `query-ids.json` and `FALLBACK_QUERY_IDS` — rotate via `pnpm run graphql:update` if it becomes stale.

### Sweetistics Fallback Pattern

Every GraphQL command has a Sweetistics fallback: if the GraphQL call fails and a Sweetistics API key is available, it automatically retries via Sweetistics before reporting failure.

### Media Handling

- Media uploads **require** the Sweetistics engine
- Supported: jpg, jpeg, png, webp, gif (images); mp4, mov (video)
- Max 4 images OR 1 video (not both)
- `--media` and `--alt` flags are repeated, aligned by order

### TweetData Shape

```typescript
interface TweetData {
  id: string;
  text: string;
  author: { username: string; name: string };
  createdAt?: string;
  replyCount?: number;
  retweetCount?: number;
  likeCount?: number;
  conversationId?: string;
  inReplyToStatusId?: string;
}
```

## Common Pitfalls

1. **GraphQL 404 / "Unknown query"** — X rotates query IDs. Run `pnpm run graphql:update` to refresh `query-ids.json`.

2. **GraphQL 429 (rate limited)** — X aggressively rate-limits GraphQL reads/writes. Use Sweetistics for high-volume posting or switch to `auto` mode.

3. **Chrome cookie extraction fails via SSH** — The macOS keychain `security` tool blocks over SSH. Use environment variables instead (`AUTH_TOKEN`, `CT0`).

4. **Media without Sweetistics** — `--media` requires Sweetistics engine. GraphQL mode will reject with a clear error message.

5. **Config not being picked up** — Check precedence: CLI > env > `.birdrc.json5` > `~/.config/bird/config.json5`. Config files use JSON5 syntax (trailing commas, comments allowed).

6. **Test failures due to missing env** — Some tests may need `SWEETISTICS_API_KEY` or valid cookies to pass. Check `tests/` for mock coverage.

7. **Binary fails after dependency changes** — `pnpm run binary` uses bun's `--compile` flag which bundles deps. If you add a new dependency, ensure it's compatible with bun's bundler or stick with `node dist/index.js`.

## Verification Checklist

- [ ] `pnpm build` passes with no errors
- [ ] `pnpm test` passes all tests
- [ ] `pnpm lint` passes (no biome violations)
- [ ] New commands follow the same Sweetistics fallback pattern
- [ ] `query-ids.json` is up to date if touching GraphQL endpoints
- [ ] Config file precedence is respected
