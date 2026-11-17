# PolarSqueezy

An open-source [Claude skill](https://docs.claude.com) that moves a **Lemon Squeezy** store's catalog to **Polar** (automated) or **Gumroad** (guided).

It plans first, shows you exactly what will and won't move, runs only after you say go, and finishes with a report.

## What it moves

| | Polar | Gumroad |
|---|---|---|
| Products and prices | Automated | Guided (you create empty products, it fills them in) |
| Files | Automated | Needs a look |
| License-key settings | Automated | Needs a look |
| Discount codes | Automated | Per product, via API |
| Customers | Records only | Not possible via API, CSV offered |
| Active subscriptions | Not moved | Not moved |

Saved cards cannot be carried over by a third party, so active subscriptions are flagged with a handover checklist, never claimed as moved.

## Safety

- The Lemon Squeezy side is read-only: GET requests only.
- Plan, then an explicit go, then run. Polar sandbox offered first.
- Re-runs are idempotent: a mapping file means nothing is created twice and nothing is deleted.
- Tokens live in session environment variables only. Never written to files or logs. Revoke them when you are done.
- It never emails your customers.

## Already on Polar's side?

Polar publishes its own open-source importer, `npx polar-migrate`, for Lemon Squeezy to Polar (products and variants, license keys, files, discount codes, customers; not active subscriptions). Use whichever you prefer. This skill adds a reviewable plan, sandbox run, re-runs, a report and a Gumroad route.

## Install

Copy `SKILL.md` into a folder named `polarsqueezy` in your skills directory (for Claude Code: `~/.claude/skills/polarsqueezy/SKILL.md`), or add it as a custom skill in Claude. Then ask: *"Migrate my Lemon Squeezy store to Polar."*

## Status

Early. Some API behaviour is marked "verify" in the skill (Polar pay-what-you-want support, importing existing license-key strings, the sandbox API host, whether Gumroad's product-create endpoint works). Treat the first run as a sandbox run and check the report.

## Disclaimer

PolarSqueezy is an independent project. It is not affiliated with or endorsed by Lemon Squeezy, Polar or Gumroad.

## Licence

MIT. See [LICENSE](LICENSE).
