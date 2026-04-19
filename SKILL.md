---
name: sp-auth0
description: Use for reliable Auth0 provisioning and configuration through Stripe Projects in this repo.
---

# Auth0 via Stripe Projects (Strict Mode)

You are responsible for correctly provisioning, configuring, and validating Auth0 resources using Stripe Projects. Your goal is not just to create resources, but to ensure they are fully functional, correctly configured, and verifiably working for the target application.

---

## Core Rules

- Prefix every command with: `DEV_MODE=true`
- Use `stripe projects` CLI as the **primary control plane**
- Reference the [Auth0 Management API OpenAPI spec](https://auth0.com/docs/api/management/openapi.json) to inform valid properties for your configuration payloads.
- Never guess config keys or capabilities; use the OpenAPI spec schema.
- Always inspect the catalog before acting to ensure the resource type is supported.
- Always verify after changes
- Prefer complete configuration over minimal configuration
- Do not declare success without validation
- If Stripe Projects cannot perform a required action, fall back to:
  `stripe projects open auth0` and explicitly describe the remaining manual steps

---

## Required Workflow (Do Not Skip Steps)

### 1) Discover Current State

Run:

```bash
DEV_MODE=true stripe projects status
DEV_MODE=true stripe projects services list
DEV_MODE=true stripe projects catalog auth0 --json
DEV_MODE=true stripe projects catalog auth0/client --json
```

Use this data alongside the **Auth0 OpenAPI spec** to determine:
- available Auth0 resource types within Stripe Projects
- supported config fields (mapped to the Auth0 API schema)
- required/optional parameters
- limitations of Stripe Projects

If `auth0/client` is not present in the catalog, stop and use only available resource types.

---

### 2) Classify the Application Type

Determine the correct Auth0 application type before creating anything:

- **Regular Web App** → server-rendered apps (e.g. Next.js with backend/session)
- **SPA** → browser-only apps using tokens directly
- **Native** → mobile/desktop apps
- **M2M** → service-to-service authentication

If unclear:
- inspect repo structure
- check dependencies (e.g. `@auth0/nextjs-auth0`)
- check routing/auth patterns

Default to **Regular Web App** for Next.js unless clearly SPA.

Never proceed without explicitly choosing an app type.

---

### 3) Derive Required URLs (Do Not Guess)

Determine:

- Base URL (e.g. `http://localhost:3000`)
- Callback URL (e.g. `/auth/callback`)
- Logout URL
- Web origins

Infer from:
- framework conventions
- existing routes
- env files
- README/config
- dev server config

Rules:
- Do NOT invent production domains
- Always include localhost for dev
- Do NOT omit required URLs
- Do NOT use wildcards unless explicitly justified

---

### 4) Create or Update Auth0 Client (Complete Config Only)

Formulate the `--config` JSON payload using the [Auth0 Management API OpenAPI spec](https://auth0.com/docs/api/management/openapi.json). 
- Creation payloads map directly to the `POST /v2/clients` endpoint.
- Update payloads map directly to the `PATCH /v2/clients/{id}` endpoint.

Create:

```bash
DEV_MODE=true stripe projects add auth0/client --name "<name>" --config '<json>' -y
```

Update:

```bash
DEV_MODE=true stripe projects update <name> auth0/client --config '<json>' -y
```

Requirements:
- You can and should use the CLI to set comprehensive configurations, including `callbacks`, `name`, `description`, `app_type`, `allowed_logout_urls`, `web_origins`, and everything else supported by the spec.
- If `--name` is used, include `"name"` in config
- Always include full URI configuration when supported.
- Do not submit partial configs for full setup requests.

---

### 5) Sync + Verify (Mandatory)

Run:

```bash
DEV_MODE=true stripe projects status
DEV_MODE=true stripe projects env
DEV_MODE=true stripe projects env --pull
```

Confirm:
- resource exists
- config applied
- env variables available locally
- project state is consistent

If verification is incomplete, explicitly state what cannot be confirmed.

---

### 6) Align Code Integration

Match implementation to the framework:

- Next.js → `@auth0/nextjs-auth0`
- Express → Auth0 Express patterns
- SPA → Auth0 SPA SDK

Rules:
- Do not mix auth patterns
- Do not introduce incompatible flows
- Reuse existing SDK if present
- Follow official integration patterns for the detected framework

---

### 7) Dashboard Fallback (When Needed)

If Stripe Projects cannot configure something:

```bash
DEV_MODE=true stripe projects open auth0
```

Then clearly specify:
- exact field names
- exact values to input
- where in the dashboard to set them

---

## Auth0 Configuration Requirements

For any web-based app, ensure ALL are set:

- Allowed Callback URLs
- Allowed Logout URLs
- Allowed Web Origins (if applicable)

Typical local setup (adjust if needed):

- `http://localhost:3000`
- `http://localhost:3000/auth/callback`

Do not claim setup is complete if these are missing.

---

## Output Requirements

Always include:

1. What was discovered from catalog/status
2. Chosen application type + reasoning
3. Exact CLI commands used or recommended
4. Final configuration values (especially URLs, validated against the OpenAPI spec)
5. Verification results
6. Any required manual steps (if applicable)

---

## Failure Handling

If something fails:

1. Re-check catalog output and the Auth0 OpenAPI spec schema
2. Remove unsupported or incorrectly formatted fields only
3. Do NOT guess replacements
4. Validate service availability (`services list`)
5. Use dashboard fallback if needed

---

## Hard Constraints

- Never fabricate config keys (always refer to the Auth0 OpenAPI spec)
- Never assume production URLs
- Never skip verification
- Never leave URI config incomplete
- Never bypass Stripe Projects when it supports the operation

---

## Commands Reference

```bash
DEV_MODE=true stripe projects status
DEV_MODE=true stripe projects services list
DEV_MODE=true stripe projects catalog auth0 --json
DEV_MODE=true stripe projects catalog auth0/client --json
DEV_MODE=true stripe projects add auth0/client --name "<name>" --config '<json>' -y
DEV_MODE=true stripe projects update <name> auth0/client --config '<json>' -y
DEV_MODE=true stripe projects env
DEV_MODE=true stripe projects env --pull
DEV_MODE=true stripe projects open auth0
```
