# latchkey-doordash-agent-proto-skill

Order food on DoorDash from the command line or an AI agent. Uses [Latchkey](https://github.com/imbue-ai/latchkey) for browser-based auth and credential injection, and [curl-impersonate](https://github.com/lexiforest/curl-impersonate) to bypass Cloudflare TLS fingerprinting.

This repo is documentation only — no code to install. A human or agent reads these docs and follows along.

## Prerequisites

- macOS or Linux
- Node.js 18+
- A DoorDash account with a saved delivery address and payment method

## Step 1: Clone Dependencies

### Latchkey

```bash
git clone https://github.com/imbue-ai/latchkey.git
cd latchkey
git checkout ad68247
npm install
npm run build
cd ..
```

### curl_chrome136

Download the `curl_chrome136` binary for your platform from [lexiforest/curl-impersonate v1.5.6 releases](https://github.com/lexiforest/curl-impersonate/releases/tag/v1.5.6). Make it executable:

```bash
chmod +x curl_chrome136
```

DoorDash sits behind Cloudflare with aggressive TLS fingerprinting. Plain curl gets 403. `curl_chrome136` impersonates Chrome's TLS and HTTP/2 fingerprint so requests go through.

### doordash-mcp (reference only)

```bash
git clone https://github.com/ashah360/doordash-mcp.git
cd doordash-mcp
git checkout 405e748
cd ..
```

This is **not** used directly. The `src/api/*.ts` files contain comprehensive GraphQL query definitions for the full DoorDash API — useful as reference material when you need to go beyond what's in the cheatsheet.

## Step 2: Browser Auth into DoorDash

```bash
LATCHKEY_CURL=/path/to/curl_chrome136 npx latchkey auth browser doordash
```

This opens a browser window. Log into your DoorDash account. Latchkey captures the session cookies (`ddweb_token`, `csrf_token`, `ddweb_session_id`) and stores them encrypted in `~/.latchkey`.

Run this from the latchkey directory (where you ran `npm run build`).

## Step 3: Validate Auth

```bash
LATCHKEY_CURL=/path/to/curl_chrome136 npx latchkey curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ consumer { id email firstName } }"}' \
  'https://www.doordash.com/graphql/consumer'
```

You should see your name and email in the response. If you get null fields, re-run Step 2.

Then confirm cart access works:

```bash
LATCHKEY_CURL=/path/to/curl_chrome136 npx latchkey curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ listCarts(input: {cartContextFilter: {experienceCase: MULTI_CART_EXPERIENCE_CONTEXT, multiCartExperienceContext: {}}, cartFilter: {shouldIncludeSubmitted: true}}) { id subtotal restaurant { business { name } } orders { orderItems { quantity item { name } } } } }"}' \
  'https://www.doordash.com/graphql/listCarts?operation=listCarts'
```

## Step 4: Use the Cheatsheet

See [CHEATSHEET.md](CHEATSHEET.md) for all common operations: searching restaurants, browsing menus, adding items to cart, placing orders, and more.

## How It Works

Latchkey injects credentials into curl commands automatically. When you run `npx latchkey curl ...` against a doordash.com URL, Latchkey:

1. Matches the URL to the DoorDash service
2. Injects stored session cookies, CSRF token, and required headers
3. Delegates the actual HTTP call to `curl_chrome136` (via `LATCHKEY_CURL`)

You write standard curl commands. Latchkey handles auth. curl_chrome136 handles TLS fingerprinting.

## Troubleshooting

**403 Forbidden**: You're not using curl_chrome136, or the `LATCHKEY_CURL` env var isn't set. Plain curl will always get 403 from DoorDash's Cloudflare.

**Null fields in consumer query**: Session expired. Re-run `latchkey auth browser doordash`.

**400 Bad Request on GraphQL**: You're probably using `operationName` + `variables` format. DoorDash requires fully inline queries through this path. See the cheatsheet's Sharp Edges section.

**403 on `/graphql/itemPage`**: This specific URL path is blocked by Cloudflare. Route through `/graphql/consumer?operation=itemPage` instead — the GraphQL router doesn't care about the URL path.

## License

MIT
