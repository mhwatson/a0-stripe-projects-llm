---
name: sp-auth0
description: Use for reliable Auth0 provisioning and configuration through Stripe Projects in this repo.
---

# Auth0 via Stripe Projects

## The #1 Rule (Read This First)

**The `--config` flag on `stripe projects add` and `stripe projects update` accepts ANY valid Auth0 Management API field as a JSON passthrough.** This includes `callbacks`, `allowed_logout_urls`, `web_origins`, `app_type`, `grant_types`, and every other field documented in the Auth0 API.

You MUST configure Auth0 clients fully via the CLI. Never tell the user to open the Auth0 dashboard to set callbacks, logout URLs, web origins, or any other field that the Auth0 Management API supports. If you catch yourself about to say "go to the dashboard to configure...", stop — put it in `--config` instead.

The ONLY acceptable reason to use `stripe projects open auth0` is for operations that have no Auth0 Management API equivalent (e.g., visual exploration, enabling social connections that require OAuth app creation on a third-party site). If you invoke the dashboard fallback, you must state which specific Auth0 API endpoint you attempted and what error you received.

## How `--config` Works

The `--config '<json>'` flag passes a JSON object whose keys map directly to the [Auth0 Management API](https://auth0.com/docs/api/management/v2):

- **Creating** a client: fields map to `POST /api/v2/clients`
- **Updating** a client: fields map to `PATCH /api/v2/clients/{id}`

This means you should reference the Auth0 Management API docs (not the catalog output) to determine valid config fields. The catalog tells you which **resource types** exist (e.g., `auth0/client`). It does NOT define the full set of configurable fields — `--config` does that by proxying to the Auth0 API.

### Fields You Must Always Include for Web Apps

These are standard Auth0 API fields, fully supported via `--config`:

| Field | Key | Example |
|---|---|---|
| App type | `app_type` | `"regular_web"`, `"spa"`, `"native"`, `"non_interactive"` |
| Callback URLs | `callbacks` | `["http://localhost:3000/api/auth/callback"]` |
| Logout URLs | `allowed_logout_urls` | `["http://localhost:3000"]` |
| Web Origins | `web_origins` | `["http://localhost:3000"]` |
| Grant Types | `grant_types` | `["authorization_code", "refresh_token"]` |

---

## Workflow

Prefix every command with `DEV_MODE=true`.

### 1. Discover Current State

```bash
DEV_MODE=true stripe projects status
DEV_MODE=true stripe projects services list
DEV_MODE=true stripe projects catalog auth0 --json
```

Use catalog output to confirm `auth0/client` (or other resource types) exist. **Do not use catalog output to decide which `--config` fields are valid** — that is determined by the Auth0 API, not the catalog.

### 2. Classify the Application Type

Determine the Auth0 app type before creating anything:

| Type | When to use | `app_type` value |
|---|---|---|
| Regular Web App | Server-rendered (Next.js w/ backend, Express) | `regular_web` |
| SPA | Browser-only, token-based | `spa` |
| Native | Mobile/desktop | `native` |
| M2M | Service-to-service, no user login | `non_interactive` |

How to determine:
- Check dependencies (`@auth0/nextjs-auth0` = Regular Web, `@auth0/auth0-spa-js` = SPA)
- Check routing patterns, middleware, session handling
- Default to **Regular Web App** for Next.js unless clearly SPA

### 3. Derive Required URLs

Determine from framework conventions, existing routes, env files, and dev server config:

- Base URL (e.g., `http://localhost:3000`)
- Callback URL(s) — framework-specific (e.g., `/auth/callback` for Next.js Auth0 SDK)
- Logout return URL(s)
- Web origins (for CORS)

Rules:
- Always include `localhost` URLs for local dev
- Do not invent production domains
- Do not use wildcards unless explicitly justified
- Do not omit any URL category

### 4. Create or Update the Auth0 Client

**Always include the full configuration in a single `--config` payload.** Do not create first and "configure later in the dashboard."

Create example (Next.js Regular Web App):

```bash
DEV_MODE=true stripe projects add auth0/client --name "my-app" --config '{
  "name": "my-app",
  "app_type": "regular_web",
  "callbacks": ["http://localhost:3000/auth/callback", "http://localhost:3000/api/auth/callback"],
  "allowed_logout_urls": ["http://localhost:3000"],
  "web_origins": ["http://localhost:3000"],
  "grant_types": ["authorization_code", "refresh_token"]
}' -y
```

Update example:

```bash
DEV_MODE=true stripe projects update my-app auth0/client --config '{
  "callbacks": ["http://localhost:3000/api/auth/callback", "http://localhost:3000/auth/callback"],
  "allowed_logout_urls": ["http://localhost:3000"],
  "web_origins": ["http://localhost:3000"]
}' -y
```

If `--config` returns an error for a specific field:
1. Check the Auth0 Management API docs for correct field name/format
2. Fix and retry
3. Only after a confirmed, unresolvable CLI failure for a specific field may you use the dashboard — and you must state the exact error

### 5. Verify (Mandatory)

```bash
DEV_MODE=true stripe projects status
DEV_MODE=true stripe projects env
DEV_MODE=true stripe projects env --pull
```

Confirm:
- Resource exists and shows correct config
- Env variables are populated locally
- URI fields (callbacks, logout URLs, web origins) are set — do not skip this check

### 6. Align Code Integration

Match to the framework's official Auth0 SDK:

- Next.js: `@auth0/nextjs-auth0`
- Express: `express-openid-connect`
- SPA: `@auth0/auth0-spa-js`

Reuse existing SDK if present. Do not mix auth patterns.

### 7. Read Before You Write (Mandatory Post-Install Step)

**After installing any Auth0 SDK (or discovering one already installed), you MUST complete these steps before writing any integration code:**

1. **Check the installed version:**
   ```bash
   npm ls @auth0/nextjs-auth0   # or whichever SDK
   ```

2. **Read the SDK's local README for the installed version:**
   ```bash
   cat node_modules/@auth0/nextjs-auth0/README.md | head -200
   ```
   If the README references a migration guide or breaking changes doc, read that too.

3. **Identify the SDK's initialization pattern** from the README — not from memory or training data. Auth0 SDKs have had breaking changes across major versions (e.g., v4 of `@auth0/nextjs-auth0` changed from `initAuth0()` to `new Auth0Client()`). Your training data may reflect an outdated version.

**Only after completing all three steps may you write integration code.**

If the installed major version differs from what you expected, call it out to the user before proceeding (e.g., "The installed SDK is v4, which uses a different initialization pattern than v3").

This step exists because SDK APIs change across major versions, and build errors from stale patterns are expensive to debug. The local `node_modules` README is the single source of truth for the installed version's API.

---

## Anti-Patterns (Never Do These)

These are the exact behaviors this skill exists to prevent:

| Bad | Good |
|---|---|
| "Go to the Auth0 dashboard and add your callback URL" | Put `callbacks` in `--config` |
| "You'll need to configure Allowed Logout URLs in the dashboard" | Put `allowed_logout_urls` in `--config` |
| "The catalog doesn't list `callbacks` so you'll need to set it manually" | `--config` accepts all Auth0 API fields regardless of catalog output |
| Creating a client with `--config '{"name": "x"}'` then saying "now configure the rest in the dashboard" | Include ALL fields in the initial `--config` payload |
| Skipping URI verification and saying "setup complete" | Always run `stripe projects status` and confirm URIs are set |
| Installing an SDK and immediately writing integration code from memory | Check installed version, read local README, then write code |
| Using `initAuth0()` or any other pattern without verifying it matches the installed version | Read `node_modules/<sdk>/README.md` to confirm the correct initialization pattern |

---

## Commands Reference

```bash
DEV_MODE=true stripe projects status
DEV_MODE=true stripe projects services list
DEV_MODE=true stripe projects catalog auth0 --json
DEV_MODE=true stripe projects add auth0/client --name "<name>" --config '<full-json>' -y
DEV_MODE=true stripe projects update <name> auth0/client --config '<json>' -y
DEV_MODE=true stripe projects env
DEV_MODE=true stripe projects env --pull
DEV_MODE=true stripe projects open auth0   # LAST RESORT — only after CLI failure with specific error
```

---

## Hard Constraints

- `callbacks`, `allowed_logout_urls`, and `web_origins` MUST be set via `--config`, never via the dashboard
- Never use catalog output to determine which `--config` fields are valid
- Never fabricate config keys — reference the Auth0 Management API
- Never assume production URLs
- Never skip verification
- Never declare setup complete if URI config is missing
- Never suggest the dashboard for anything the Auth0 Management API supports
- Never write integration code before checking the installed SDK version and reading its local README
- Never assume SDK initialization patterns from training data — always verify against `node_modules/<sdk>/README.md`
