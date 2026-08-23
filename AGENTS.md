# AGENTS.md

## Purpose

This repo is a single Cloudflare Worker that exposes a UniFi-compatible Dynamic DNS
endpoint. UniFi OS devices (UDM-Pro, USG) can't use Cloudflare as a DDNS provider
natively, so the Worker accepts the DDNS update requests those devices already know how
to send (dyndns/duckdns/ydns query-string flavors, HTTP Basic auth) and translates them
into Cloudflare API v4 calls that update the matching `A`/`AAAA` DNS records. End users
deploy their own copy of the Worker and point their router at its `*.workers.dev` route.

## Tech stack

- JavaScript (ES modules), no TypeScript, no build step, no dependencies.
- Cloudflare Workers runtime (`export default { fetch }` module worker), `compatibility_date = "2023-04-23"`.
- Wrangler CLI for local dev and deploy; GitHub Actions (`cloudflare/wrangler-action@v3`) for CI deploy.
- No `package.json`, no test framework, no linter/formatter config in the repo.

## Layout

- `src/index.js` — the entire Worker. Entry point is the default-exported `fetch` handler
  at the bottom of the file, which delegates to `handleRequest` and converts thrown
  errors into plain-text responses. Other pieces: `Cloudflare` (API client:
  `findZone`, `findRecord`, `updateRecord`), `requireHttps`, `parseBasicAuth`,
  `informAPI` (hostname → zone resolution and record update loop), and the
  `BadRequestException` / `CloudflareApiException` error classes that carry
  `status`/`statusText`.
- `wrangler.toml` — Worker name, `main`, compatibility date. No bindings, vars, or secrets;
  the Cloudflare API token is supplied per-request by the router, never stored in the Worker.
- `.github/workflows/deploy.yml` — deploys on push to `main` and on `workflow_dispatch`,
  using the `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` repo secrets.
- `.devcontainer/devcontainer.json` — Node devcontainer that installs Wrangler globally.
- `README.md` — user-facing setup for Cloudflare, UniFi OS, and on-device debugging
  (`inadyn` on UDM-Pro, `ddclient` on USG). Keep it in sync when request handling changes.

## Build / run / test

There is no build and there are no automated tests. Install Wrangler
(`npm i -g wrangler` or `yarn global add wrangler`), then:

- `wrangler dev` — run the Worker locally.
- `wrangler deploy` — deploy to your Cloudflare account.

Verify changes by hitting the endpoint directly, e.g.:

```
curl -u '<zone-name>:<cloudflare-api-token>' \
  'https://<worker>.<subdomain>.workers.dev/update?hostname=sub.example.com&ip=203.0.113.1'
```

A successful update returns `200` with the body `good`. Errors return `400`/`404`/`500`
with a plain-text message. Real end-to-end checks require a Cloudflare zone, a
**Zone:DNS:Edit** scoped API token, and a pre-existing `A`/`AAAA` record — the Worker
updates records but never creates them.

## Conventions

- Tabs for indentation, double-quoted strings, semicolons — match `src/index.js`.
- Keep the Worker dependency-free and single-file; don't introduce `package.json`,
  a bundler, or npm packages without a strong reason.
- Signal error responses by throwing `BadRequestException` or `CloudflareApiException`;
  the top-level `fetch` catch turns them into responses. Don't build error `Response`
  objects deep in the call stack.
- Every response that carries data sets `Cache-Control: no-store`.
- Requests must be HTTPS (`requireHttps` checks both the URL protocol and
  `x-forwarded-proto`); unauthenticated requests are answered with `404`, not `401`,
  to avoid advertising the endpoint. Preserve that behavior.
- Query-parameter parsing is deliberately permissive across DDNS dialects
  (`hostname`/`host`/`domains`, `ips`/`ip`/`myip`, `token`, falling back to the
  `Cf-Connecting-Ip` header). Add new aliases rather than renaming existing ones —
  routers in the field depend on them.
- The API token arrives as the Basic-auth password (or `?token=`) and the zone name as the
  username. Never log, persist, or echo credentials; the existing code only
  `console.error`s error objects.
- `.gitignore` covers `node_modules`, `.mf/cert`, and `secrets.txt`; never commit tokens.
- Pushing to `main` deploys automatically, so treat `main` as releasable.
