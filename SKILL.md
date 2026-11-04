---
name: compliance-pages
description: Generate a small website's privacy, terms and cookie pages plus a self-hosted, dependency-free consent banner from one site-profile JSON of stated facts. Use when an agency or freelancer wants the local half of CookieYes ($10–55 per domain per month) or iubenda ($4.99–79.99/site/mo plus pageview overages) — the documents and the banner as files the site owns, no per-domain rent, no hosted consent service. The output is templates, never legal advice and never a compliance guarantee; facts the profile does not state ship as loud [TO FILL: ...] placeholders, never as invented text. Exit codes are 0 (ok), 1 (attention — placeholders or contradictions remain) and 2 (clean error).
---

# compliance-pages — agent skill

## Purpose

Every small website needs a privacy page, a terms page, a
cookie page and a consent banner. Platforms rent those back
per domain, per month, forever. This tool generates them
locally from one profile of **stated facts**: three
self-contained HTML pages, a `consent.js` + `banner.html`
pair, and a `manifest.txt` that audits exactly which profile
values the documents were built from. Pure Python standard
library; local only; the profile and the `--out` folder are
the only things it reads or writes.

**The honesty core of this tool:** the documents may only
ever say what the profile says. A fact the profile does not
state becomes a loud `[TO FILL: ...]` placeholder in the
output — listed in the manifest, counted, and reflected in
the exit code. There is no field, flag or supported edit
that fills a gap with plausible-sounding text. If the
profile is missing a fact, ask the site owner for the fact
and put it in the profile; never patch the rendered output.

## When to use / not use

Use when:
- A small static site (agency client work, a trades or
  clinic site) needs privacy/terms/cookie pages and a
  consent banner as files it owns.
- An agent drafts a site profile from the owner's stated
  facts and generates the pages for the owner to review.
- Auditing an existing profile for gaps and contradictions
  before a site launches (`check`).

Do NOT use when:
- The user wants "GDPR/CCPA compliance" or any legal
  guarantee. Refuse that framing plainly: this tool produces
  starting-point templates that carry their own
  not-legal-advice notice; a qualified professional reviews
  them. Never describe the output as compliant, certified,
  or legally sufficient — in the pages, in the manifest, or
  in your own summary to the user.
- The user needs a server-side consent log, IAB TCF
  strings, geo-targeted banners, or automatic script
  blocking. All are hosted-service features, out of scope by
  design; the banner records a choice in the visitor's
  browser and blocks nothing by itself.
- The user asks you to "just put something reasonable" for
  a missing fact (what data the site collects, how long it
  is kept, the governing law, how consent gating is wired).
  Those are facts only the owner can supply. Surface the
  gap; do not fill it.

## Commands

```bash
# Generate everything into a folder (validates structure
# first; a structurally broken profile writes NOTHING)
python3 src/compliance_pages.py make --profile site.json \
    --out FOLDER [--as-of YYYY-MM-DD] [--json]

# Gaps + contradictions for a profile; writes nothing
python3 src/compliance_pages.py check --profile site.json \
    [--as-of YYYY-MM-DD] [--json]
```

## The profile (JSON)

Root object. Unknown fields are ignored. Every field is a
fact stated by the site owner; absent means "not stated".

| Field | Rules |
|---|---|
| `business` | object of optional strings: `name`, `legal_form`, `country`, `province`, `address`, `contact_email`, `contact_name`. `country` selects the variant: `"Canada"` (any case) → `canada`; anything else → `generic` |
| `site` | object of optional strings: `name`, `url`. The URL prints as text |
| `as_of` | `YYYY-MM-DD` — the generation / "last updated" stamp. May be overridden by `--as-of`. One of the two is **required** for `make`; the date is never read from the clock |
| `data_collected` | list of non-empty strings — personal information the site collects, owner's words. Absent or empty = not stated |
| `data_use` | list of non-empty strings — what it is used for |
| `services` | object: `hosting`, `analytics`, `payments` (string or `null` = not used), `embeds` (list of strings). Whole block absent = not stated (placeholder), which is different from "declared, all null" |
| `cookies` | object with **all three** booleans required when present: `essential`, `analytics`, `marketing`. Absent block = not stated — never read as "no cookies" |
| `consent_note` | string — how analytics/marketing scripts are gated on the stored choice, owner's words, for wiring that exists |
| `retention` | string — how long collected information is kept |
| `governing_law` | string — the terms page's governing-law line |
| `quotes_are_estimates` | boolean, default `false`; `true` adds a "Quotes and estimates" section to the terms page |
| `pages` | non-empty subset of `privacy`, `terms`, `cookies`; default all three. `consent.js`, `banner.html`, `manifest.txt` are always generated |

Structural problems (wrong types, unknown page name,
impossible date, a `cookies` block missing a key) are
collected into one error naming each, exit 2, nothing
written.

## What generates, and when

- **Sections follow the facts.** The privacy page's
  third-party table names only declared services; the
  cookies page's category subsections (Essential /
  Analytics / Marketing) appear only for categories marked
  `true`; the terms page's "Quotes and estimates" section
  exists only when `quotes_are_estimates` is `true`; the
  Canada variant names the Office of the Privacy
  Commissioner of Canada in the privacy page, the generic
  variant names "your local data-protection authority".
- **Placeholders.** One canonical `[TO FILL: ...]` marker
  per unstated fact, repeated at each place a document uses
  that fact (the contact email's marker appears on all three
  pages). Markers are counted mechanically in the generated
  documents; `manifest.txt` lists them and is not counted
  itself (it would double-count).
- **The banner.** `banner.html` is a fragment: paste before
  `</body>`, then load `consent.js`. Category checkboxes
  appear only for declared optional categories; with none,
  the banner is an essential-cookies notice with one OK
  button. It links to `cookies.html` only when that page is
  generated. `consent.js` is dependency-free vanilla JS: no
  network requests, no beacons; it stores the choice in the
  visitor's `localStorage` under `cp-consent-v1`, publishes
  `window.cpConsent.get()` and a `cp:consent` event (and
  `window.cpConsent.reset()` clears the stored choice and
  re-shows the banner, for a "cookie settings" link), and
  blocks nothing by itself.
- **Deterministic.** Same profile + same date →
  byte-identical files, manifest included.

## Gaps and contradictions (`check`, and `make`'s manifest)

Gaps are the unstated facts above, reported in schema order
with the placeholder each will ship as. An unset generation
date is also reported (`as_of` gap; `make` refuses without
a date, exit 2).

Contradictions — facts that fight each other:

1. `cookies` is in `pages` while the `cookies` block says
   the site sets no cookies (all three `false`).
2. `services.analytics` names a provider while
   `cookies.analytics` is `false`.
3. `cookies.analytics` or `cookies.marketing` is `true`
   with no `consent_note` — the cookies page then carries
   the consent-gating placeholder where the note belongs.

## Output JSON schema (`--json`)

Common fields: `tool` (`"compliance-pages"`), `version`,
`command`, `profile` (file name), `variant`.

- `make`: `as_of`, `out_dir`, `files` (canonical order:
  requested pages, `consent.js`, `banner.html`,
  `manifest.txt`), `sections` — `{file: [section names]}`
  for the three pages (cookies page category subsections
  are listed as `category: Essential` etc.),
  `placeholders` — `{total, items: [{text, count, files}]}`,
  `gaps`, `contradictions`, `attention` (true when exit
  code is 1).
- `check`: `as_of` (or null), `pages`, `gaps` —
  `[{field, placeholder}]` (`placeholder` is null for the
  `as_of` gap), `contradictions`, `complete`.

On error, stdout is a single object: `{"error": "<message>"}`
with exit code 2 — including argparse usage errors in
`--json` mode. Never plain text, never a traceback.

## Exit codes

- `0` — success and complete: `make` wrote files with no
  placeholders and no contradictions; `check` found a
  complete, consistent profile.
- `1` — ATTENTION: `make` wrote the files, but placeholders
  remain and/or contradictions were flagged (both listed in
  `manifest.txt`); or `check` found gaps/contradictions.
  Do not present the generated pages as finished while the
  exit code is 1.
- `2` — ERROR: profile missing/corrupt/structurally
  invalid (all problems named), no generation date,
  `--out` not a folder, invalid flag values. Nothing was
  written; a corrupt profile is left byte-identical.

## Limits & honesty rules the agent must respect

- **Never invent a fact.** Not a data item, not a retention
  period, not a governing law, not a consent-gating claim.
  If it is not in the profile, it does not go in the pages —
  the placeholder is the honest output.
- **Never call it compliance.** The pages are templates
  with a not-legal-advice notice baked in. Relay them as
  starting points for professional review, and keep the
  notice intact when you quote or copy the output.
- **Placeholders stay loud.** Do not strip, soften or
  "temporarily" replace `[TO FILL: ...]` markers to make a
  page look finished. A page with markers is an unfinished
  page; report the count and the fields.
- **The banner blocks nothing.** When you summarise an
  install, state the remaining wiring: include
  `banner.html`, load `consent.js`, and gate optional
  scripts on `window.cpConsent` / the `cp:consent` event.
  Never claim consent is "handled" by the fragment alone,
  and never claim a consent record exists — storage is the
  visitor's browser only.
- **The profile is the owner's statement.** `check` catches
  the profile contradicting itself, not the profile
  contradicting the real website. If the owner says the
  site sets no cookies, the pages say so; flagging a
  mismatch with the actual site is a human's job.
- Local only, sends nothing: the tool reads/writes only the
  profile and the `--out` folder, and the generated files
  make no external requests (the one URL that may appear is
  the site's own, printed as text from the profile).
