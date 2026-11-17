---
name: polarsqueezy
description: Migrate a Lemon Squeezy store's products, prices, files, license-key config, discount codes and customers to Polar (automated) or Gumroad (guided). Plan first, run safely, report at the end.
---

# PolarSqueezy

Move a merchant's catalog from **Lemon Squeezy (LS)** to **Polar** (automated) or **Gumroad** (guided). Voice: friendly, plain English, honest about what cannot move. Never promise more than the destination's API allows.

## Ground rules (never break these)

1. **Copy, never cut.** The LS side is read-only. Make only GET requests to LS. Never delete or edit anything in the source store.
2. **Plan before anything moves.** Show the plan, get an explicit "go", then execute.
3. **Sandbox first for Polar.** Offer a run against Polar's sandbox before production. Production only after the user confirms.
4. **Idempotent and re-runnable.** Keep a mapping file so a second run creates nothing twice and deletes nothing.
5. **Secrets stay in memory.** Tokens go in environment variables for the session only. Never write them to files, logs, the report or chat. Tell the user to revoke the tokens when done.
6. **Never email the merchant's customers.** Create customer records only. No receipts, no invites, no marketing opt-ins unless LS says the customer opted in.
7. **Subscriptions do not move.** Saved cards cannot be carried over by a third party. Detect subscription products and subscribers, flag them "Needs a look" or "Can't move", and give the handover checklist. Never claim they moved.
8. State plainly the first time: this is an independent tool, not affiliated with Lemon Squeezy, Polar or Gumroad.

## Step 0: tell them about the official option, then preflight

Polar publishes an open-source importer (`npx polar-migrate`, needs an LS API key). Per Polar's docs it moves products and variants, license keys, files, discount codes and customers, but not active subscriptions. Mention it once, honestly, and say what this skill adds: a reviewable plan, sandbox run, re-runs, a report, and a Gumroad route. Let the user choose.

Preflight network access. The workspace shell may block these hosts:

```bash
for u in https://api.lemonsqueezy.com/v1/users/me https://api.polar.sh/v1/organizations/ https://sandbox-api.polar.sh/v1/organizations/ https://api.gumroad.com/v2/user; do
  printf '%s -> ' "$u"; curl -sS -o /dev/null -w '%{http_code}\n' --max-time 10 "$u" || echo blocked
done
```

A 401 means reachable (just unauthenticated). If a host is blocked, do not retry with other commands. Instead, if a shell on the user's own computer is available (device bash), run the same scripts there. Otherwise write the scripts into a folder for the user to run locally and explain how.

## Step 1: ask for what is needed (one message)

- Destination: **Polar** or **Gumroad**.
- LS API key (read-only is enough; test-mode keys only see test data, so say which they gave).
- Which LS store (list with `GET /v1/stores` if more than one).
- Polar: an **organization access token** with scopes for products, benefits, files, discounts and customers; say sandbox or production. Gumroad: they must create an OAuth application and token with the `edit_products` scope; walk them through it.
- Set tokens as env vars (`LS_API_KEY`, `POLAR_TOKEN`, `GUMROAD_TOKEN`). Never echo them.

## Step 2: scan LS (read-only)

Base `https://api.lemonsqueezy.com/v1`, headers `Authorization: Bearer $LS_API_KEY`, `Accept: application/vnd.api+json`. JSON:API; paginate with `page[number]` and `page[size]` (max 100) and follow `links.next`. Be gentle with rate limits: honour `Retry-After` and back off on 429.

Read: `/stores`, `/products?filter[store_id]=ID&include=variants`, `/variants`, `/prices`, `/files`, `/discounts`, `/customers`, `/license-keys`, `/subscriptions` (count only, to flag). Save a normalized snapshot to `snapshot.json` (no secrets). Normalize into: products (name, description, status, image URLs, variants with price/currency/interval/pay-what-you-want/has-files/license settings), files (name, size, download URL), discounts (name, code, percent or fixed, max redemptions, start/end, duration, product restriction), customers (email, name, country, marketing opt-in).

File download URLs may be signed and expire. Download and upload in the same step; do not keep customer files after transfer. Confirm field names against a live response rather than guessing.

## Step 3: build the plan and show it

Give every item exactly one status:

- **Ready**: maps cleanly.
- **Needs a look**: copies with a lossy mapping or needs a manual step. Say why.
- **Can't move**: skipped and listed in the report. Say why.

| Situation | Polar | Gumroad |
|---|---|---|
| One-time product, fixed price | Ready | Needs a look (empty product must exist first) |
| Several variants | Needs a look: default one Polar product per variant, named `Product, Variant`; offer to merge into one product with several prices | Variant category and variants on the existing product |
| Pay-what-you-want | Ready if supported for the product type (verify), else Needs a look | Needs a look: minimum as price |
| Subscription product or subscribers | Can't move (offer catalog-only copy if the user asks, never subscribers) | Can't move |
| Files | Ready | Needs a look (verify file attach via API) |
| License-key settings | Ready (license-key benefit) | Needs a look |
| Already-issued license keys | Needs a look unless Polar can import key strings (verify) | Can't move |
| Discount, all products | Ready | Needs a look: Gumroad codes are per product, fan out per product |
| Discount with an unsupported rule (variant-specific, currency mismatch) | Needs a look | Needs a look |
| Customers | Ready (records only) | Can't move; offer a CSV |
| Currency the destination does not support | Can't move | Can't move |

Present: counts per type, then a table of every item with its status and a one-line reason. Then ask: "Want me to run this against the Polar sandbox first?" Do not proceed without a clear yes.

## Step 4a: execute for Polar

Base `https://api.polar.sh/v1` (production) or `https://sandbox-api.polar.sh/v1` (sandbox; confirm the host before use). `Authorization: Bearer $POLAR_TOKEN`. Before writing, fetch Polar's current OpenAPI (https://polar.sh/docs/api-reference, or https://polar.sh/docs/llms.txt) and confirm field names; do not trust the sketch below blindly.

Order: **benefits, then products, then attach benefits, then discounts, then customers.**

- Files: create a file record, upload (multipart), mark uploaded, create a `downloadables` benefit.
- License-key settings: `license_keys` benefit with prefix, activation limit, expiry as in LS.
- Product: create with name, description, `recurring_interval: null`, fixed price (minor units + currency), then attach benefits.
- Discount: name, code, percent or fixed, redemption cap, start/end, product restriction.
- Customer: email, name, billing country only.
- Put `polarsqueezy_source_id` in each object's `metadata` where supported, so a crash can be recovered by lookup.

Idempotency: keep `entity_map.json` of `{kind, ls_id, polar_id, hash, status}`. Before every write, check the map (and metadata lookup). Skip when already present. A re-run never duplicates and never deletes. Write progress after each item. On 429 or 5xx retry with exponential backoff and jitter, at most 5 times, then mark the item failed and continue.

## Step 4b: guided run for Gumroad

Gumroad's API does not reliably create products (sources say product creation is dashboard-only; one third-party spec disagrees). Test once with the user's token: `POST https://api.gumroad.com/v2/products`. If it works, treat Gumroad like Polar. If not:

1. Give the user a copy-paste sheet per product (name, price, description, files to upload) and say "create these empty products in Gumroad, then tell me when done".
2. `GET /v2/products` and match to LS products by name. Ask the user to confirm any ambiguous match.
3. Fill in via API what is supported on those products: updates, variant categories and variants (`/products/:id/variant_categories`, `/variants`), offer codes (`POST /products/:id/offer_codes`, percent or fixed), custom fields.
4. Customers, sales and subscribers cannot be created through the API. Offer a CSV of customers for their email tool. Do not say customers moved.

## Step 5: report

Write `polarsqueezy-report.md` and send it. Contents: what moved (counts), every "Needs a look" item with the manual step, every "Can't move" item with the reason, failures to retry, and an unpacking checklist: click through the new store, check flagged items, update links and checkout URLs on the site and emails, handle subscribers (ask them to re-subscribe at their next billing date, or plan a handover), pause the old store only when happy. End with: revoke the LS, Polar and Gumroad tokens you shared; delete `snapshot.json` if it holds customer emails you no longer need.

## Failure handling

- Bad key: say which one and how to make a new one. Do not guess.
- Network blocked: see Step 0. Do not hunt for workarounds.
- Partial failure: finish the rest, list failures, offer a re-run that retries only those.
- Anything unexpected in a response: stop, show the user, do not improvise a write.

## Checklist before you say "done"

- [ ] No write ever went to LS.
- [ ] Report counts match the plan and the mapping file.
- [ ] No secret appears in any file, log or message.
- [ ] Subscriptions were flagged, not claimed moved.
- [ ] The user was told what to do by hand.
