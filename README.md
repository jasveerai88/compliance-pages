# compliance-pages

**Consent platforms charge you per domain, forever, for
pages that are documents. Documents are files.**

CookieYes bills $10/$25/$55 **per domain per month** — an
agency managing ten client sites on the Pro plan pays about
$250/month, and there is no multi-domain bundle: every domain
requires its own subscription. iubenda charges
$4.99–$79.99 per site per month, plus pageview overages.
Every one of those client sites needs the same things: a
privacy page, a terms page, a cookie page, and a consent
banner. `compliance-pages` generates them locally from one
site-profile JSON — static files that belong to the site, a
self-hosted banner with no third party in the loop, no
subscription, no view metering, no per-domain rent.

- Pure Python 3 standard library — no dependencies, no install step
- Local only — the profile never leaves your machine, and the
  generated pages make no external requests
- Facts come only from the profile — anything unstated ships
  as a loud `[TO FILL: ...]` placeholder, never as an
  invented sentence
- Script-friendly exit codes: `0` = ok, `1` = attention
  (files written, but placeholders or contradictions remain),
  `2` = clean error, nothing written

## Read this first — what these pages are, and are not

These are **templates**, and every generated page says so at
the top, in a visible notice that ships with the document:

> Template — this page was generated from a site profile by
> compliance-pages. It is a starting point, not legal advice:
> have a qualified professional review it before relying on
> it.

- **Not legal advice, not a guarantee.** The tool never calls
  its output "compliant" with any law, and neither should
  you. Laws differ by jurisdiction; the profile drives a
  modest generic variant, or a Canada-flavoured one when
  `business.country` is Canada. That is labelling, not
  coverage.
- **The tool never invents a fact.** A fact the documents
  need but the profile does not state becomes a loud
  `[TO FILL: ...]` placeholder in the output, is listed in
  the manifest, and makes `make` exit 1. Sections appear
  only when the profile's facts call for them: no analytics
  in the profile, no analytics section claiming any.
- **The banner records a choice; it blocks nothing.** The
  visitor's choice is stored in the visitor's own browser
  (local storage) — nothing is sent to a server, there is no
  consent log, and there is no IAB TCF support. The site
  owner must still wire analytics and marketing scripts to
  honour the stored choice; `consent.js` publishes it as
  `window.cpConsent` and a `cp:consent` event for exactly
  that, and the manifest spells the wiring out.

## The profile

One JSON file per site — every field is a fact **stated by
the site owner**. Absent means "not stated", never "guess":

```json
{
  "business": {"name": "Okanagan Valley Plumbing & Heating Ltd.",
               "legal_form": "corporation",
               "country": "Canada",
               "province": "British Columbia",
               "address": "1420 Industrial Court, Kelowna, BC V1X 3B7",
               "contact_email": "privacy@ovph.example",
               "contact_name": "Office Manager"},
  "site":     {"name": "OV Plumbing & Heating",
               "url": "https://ovph.example"},
  "as_of": "2026-11-04",
  "data_collected": ["Your name, phone number and email address when you send the quote form"],
  "data_use": ["To reply to quote requests and schedule work"],
  "services": {"hosting": "ExampleHost Canada",
               "analytics": "Example Metrics",
               "payments": "Stripe (for card payments on invoices)",
               "embeds": ["A map embed on the contact page"]},
  "cookies":  {"essential": true, "analytics": true, "marketing": false},
  "consent_note": "The analytics script is loaded only after a visitor accepts analytics cookies in the banner…",
  "retention": "Quote requests and job records are kept for seven years…",
  "governing_law": "the laws of the Province of British Columbia and the federal laws of Canada that apply there",
  "quotes_are_estimates": true,
  "pages": ["privacy", "terms", "cookies"]
}
```

| Field | Meaning |
|---|---|
| `business.name` | who operates the site |
| `business.legal_form` | e.g. corporation, sole proprietorship, partnership — in the owner's words |
| `business.country` | drives the variant: `Canada` (any case) → Canada-flavoured labels; anything else → generic |
| `business.province` | optional; printed where the business is described |
| `business.address`, `business.contact_email`, `business.contact_name` | printed in the pages' who-we-are / contact sections |
| `site.name`, `site.url` | the website itself; the URL prints as text, never as a link the renderer adds |
| `as_of` | generation / "last updated" date, `YYYY-MM-DD` — or pass `--as-of` (which wins). One of the two is **required**: the stamp date is never read from the clock |
| `data_collected` | list of strings — the personal information the site collects, in the owner's words |
| `data_use` | list of strings — what it is used for |
| `services.hosting` / `.analytics` / `.payments` | provider names, or `null` for "not used" |
| `services.embeds` | list of strings — embedded third-party content |
| `cookies` | `{essential, analytics, marketing}` — all three booleans required when the block is present; an **absent** block means "not stated" (placeholder), **not** "no cookies" |
| `consent_note` | how analytics/marketing scripts are gated on the stored choice, in the owner's words — for wiring that actually exists |
| `retention` | how long collected information is kept |
| `governing_law` | the governing-law line for the terms page |
| `quotes_are_estimates` | when `true`, the terms page gains a "Quotes and estimates" section |
| `pages` | subset of `privacy`, `terms`, `cookies`; default all three. `consent.js`, `banner.html` and `manifest.txt` are always generated |

Structural problems (wrong types, an unknown page name, an
impossible date, a half-given `cookies` block) are collected
and reported together, and a broken profile writes
**nothing**. Missing *facts* are different: they generate,
loudly (below).

## Commands

```bash
# Generate the pages, banner and manifest
python3 src/compliance_pages.py make \
    --profile examples/sample-site.json --out /tmp/cp1

# Audit a profile first — gaps and contradictions, writes nothing
python3 src/compliance_pages.py check \
    --profile examples/sample-site.json
```

`make` writes `privacy.html`, `terms.html`, `cookies.html`
(as requested), `consent.js`, `banner.html`, and
`manifest.txt` — the manifest lists the files, the sections
in each, **every profile value the documents were built
from**, every placeholder still standing, every
contradiction flagged, and the owner wiring still to do.

`check` reports two things:

- **Gaps** — facts the documents need that the profile does
  not state. Each is named with the placeholder it will ship
  as.
- **Contradictions** — facts that fight each other:
  a cookies page requested while the profile says the site
  sets no cookies; an analytics service declared while
  analytics cookies are marked off; optional (analytics /
  marketing) cookies declared with no `consent_note`
  describing how those scripts are gated on consent.

## Installing the banner

Paste `banner.html` into each page just before `</body>`,
then load `consent.js` after it:

```html
<script src="consent.js" defer></script>
```

Then the wiring the owner still owes (also printed in every
manifest): gate your analytics and marketing scripts on the
stored choice. `consent.js` publishes the choice as
`window.cpConsent.get()` and fires a `cp:consent` event on
`window`, both on page load (when a choice already exists)
and the moment a new choice is saved. A "cookie settings"
link can call `window.cpConsent.reset()` to clear the
stored choice and show the banner again, so visitors can
change their mind. The banner records a choice and blocks
nothing by itself; a choice is stored only in the visitor's
browser, under the local-storage key `cp-consent-v1`, and
clearing the browser's site data asks again.

## Usage with AI agents

Every command takes `--json`: exactly one JSON object on
stdout, with the same exit codes as the text view (errors
come back as `{"error": ...}`, argparse usage errors
included). Agents: read [SKILL.md](SKILL.md) first — the
profile schema, the JSON output shapes, exit codes, and the
honesty rules (never invent a fact, never call the output
compliant, placeholders stay loud).

## Real runs (verbatim)

A complete profile — exit 0, no placeholders:

```text
$ python3 src/compliance_pages.py make --profile examples/sample-site.json --out /tmp/cp1
Wrote 6 file(s) into /tmp/cp1: privacy.html, terms.html, cookies.html, consent.js, banner.html, manifest.txt
  Placeholders remaining: 0 — the profile stated every fact the documents use
  compliance-pages generated templates only — not legal advice; have them reviewed before relying on them.
(exit code: 0)
```

Its manifest, in full:

```text
compliance-pages v0.1.0 — generation manifest
Profile: sample-site.json
Stamped date: 2026-11-04 (from the profile's as_of)
Variant: canada

Files written:
  privacy.html
  terms.html
  cookies.html
  consent.js
  banner.html
  manifest.txt

Sections generated:
  privacy.html: Who we are; Information we collect; How we use information; Third-party services; Cookies; How long we keep information; Your choices and how to contact us
  terms.html: Who operates this site; Using this site; Quotes and estimates; Site content and intellectual property; Limits of liability; Governing law; Changes to these terms; Contact
  cookies.html: What cookies are; Cookies this site sets; The consent banner and your stored choice; Managing cookies in your browser; Contact; category: Essential; category: Analytics

Profile values used:
  business.name: Okanagan Valley Plumbing & Heating Ltd.
  business.legal_form: corporation
  business.country: Canada
  business.province: British Columbia
  business.address: 1420 Industrial Court, Kelowna, BC V1X 3B7
  business.contact_email: privacy@ovph.example
  business.contact_name: Office Manager
  site.name: OV Plumbing & Heating
  site.url: https://ovph.example
  as_of: 2026-11-04
  data_collected: 3 item(s) stated
  data_use: 3 item(s) stated
  services.hosting: ExampleHost Canada
  services.analytics: Example Metrics
  services.payments: Stripe (for card payments on invoices)
  services.embeds: 1 item(s) stated
  cookies.essential: true
  cookies.analytics: true
  cookies.marketing: false
  consent_note: The analytics script is loaded only after a visitor accepts analytics cookies in the banner. Our pages read the stored choice through window.cpConsent before the analytics script tag is added, and the script is never added for a visitor who declined or has not chosen yet.
  retention: Quote requests and job records are kept for seven years for tax and warranty records. Contact-form messages that do not lead to a job are deleted after 12 months.
  governing_law: the laws of the Province of British Columbia and the federal laws of Canada that apply there
  quotes_are_estimates: true
  pages: privacy, terms, cookies

Placeholders in the generated documents: 0
  (counted in the pages, banner.html and consent.js; this manifest
   lists the markers, so it is not counted itself)

Contradictions flagged: 0

Owner wiring still to do:
  - Paste banner.html into every page before </body> and load
    consent.js after it.
  - Gate analytics and marketing scripts on the stored choice:
    consent.js publishes window.cpConsent.get() and fires a
    "cp:consent" event; the banner records a choice and blocks
    nothing by itself.
  - Consent is stored only in the visitor's browser (local
    storage); there is no server-side consent log and no IAB
    TCF support.
  - Have a qualified professional review the generated pages
    before relying on them. These are templates, not legal
    advice; laws differ by jurisdiction.
```

An incomplete profile — it still generates, and every
unstated fact stands in the output as a loud placeholder.
Exit 1:

```text
$ python3 src/compliance_pages.py make --profile examples/incomplete-site.json --out /tmp/cp2
Wrote 6 file(s) into /tmp/cp2: privacy.html, terms.html, cookies.html, consent.js, banner.html, manifest.txt
  Placeholders remaining: 15 — the profile did not state every fact the documents use (listed in manifest.txt)
  compliance-pages generated templates only — not legal advice; have them reviewed before relying on them.
(exit code: 1)
```

Its manifest's placeholder audit (the count is exact —
markers are counted in the generated documents; the
manifest lists them, so it is not counted itself):

```text
Placeholders in the generated documents: 15
  (counted in the pages, banner.html and consent.js; this manifest
   lists the markers, so it is not counted itself)
  [TO FILL: website address (URL)] x2 — privacy.html, terms.html
  [TO FILL: legal form of the business — e.g. corporation, sole proprietorship, partnership] x2 — privacy.html, terms.html
  [TO FILL: business mailing address] x1 — privacy.html
  [TO FILL: the personal information this site collects] x1 — privacy.html
  [TO FILL: what the collected information is used for] x1 — privacy.html
  [TO FILL: third-party services this site uses — hosting, analytics, payments, embeds — or remove this section if there are none] x1 — privacy.html
  [TO FILL: which cookies this site sets — essential, analytics, marketing] x2 — privacy.html, cookies.html
  [TO FILL: how long collected information is kept] x1 — privacy.html
  [TO FILL: contact email for privacy and terms questions] x3 — privacy.html, terms.html, cookies.html
  [TO FILL: the governing law for these terms] x1 — terms.html
```

And `check` on both profiles:

```text
$ python3 src/compliance_pages.py check --profile examples/sample-site.json
compliance-pages v0.1.0 — check · profile sample-site.json
Variant: canada · pages make would generate: privacy, terms, cookies (+ consent.js, banner.html, manifest.txt always)
Generation date: 2026-11-04

GAPS: none — the profile states every fact the documents use.

Verdict: COMPLETE — make will generate with no placeholders.
(exit code: 0)

$ python3 src/compliance_pages.py check --profile examples/incomplete-site.json
compliance-pages v0.1.0 — check · profile incomplete-site.json
Variant: canada · pages make would generate: privacy, terms, cookies (+ consent.js, banner.html, manifest.txt always)
Generation date: 2026-11-04

GAPS — facts the documents need that the profile does not state (10):
  - business.legal_form → [TO FILL: legal form of the business — e.g. corporation, sole proprietorship, partnership]
  - business.address → [TO FILL: business mailing address]
  - business.contact_email → [TO FILL: contact email for privacy and terms questions]
  - site.url → [TO FILL: website address (URL)]
  - data_collected → [TO FILL: the personal information this site collects]
  - data_use → [TO FILL: what the collected information is used for]
  - services → [TO FILL: third-party services this site uses — hosting, analytics, payments, embeds — or remove this section if there are none]
  - cookies → [TO FILL: which cookies this site sets — essential, analytics, marketing]
  - retention → [TO FILL: how long collected information is kept]
  - governing_law → [TO FILL: the governing law for these terms]

Verdict: INCOMPLETE — fill the gaps above in the profile; anything left unstated ships as a loud [TO FILL: ...] placeholder, never as an invented fact.
(exit code: 1)
```

## Tests

```bash
python3 -m unittest discover -s tests -v
# Ran 34 tests — OK
```

The suite pins, in both directions: the complete sample
exits 0 with zero placeholders and the hand-derived section
lists for all three pages; the incomplete sample exits 1
with exactly 15 placeholders (privacy 9, terms 4, cookies 2,
banner 0, consent.js 0 — counted by hand from the placement
map in the test docstring) and `check` names its 10 gaps;
the three contradiction rules fire and stay silent when the
profile is consistent; section presence follows the facts
(analytics, payments, quotes-are-estimates, page subsets);
the generated pages contain no URL but the site's own, as
text; `consent.js` contains no network, beacon or script
strings and does use local storage; a business name
containing `<script>` renders as escaped text; rendering is
byte-identical across runs; `--as-of` overrides the profile
date, and no date anywhere is a clean exit-2 error; a
corrupt profile errors with the file left byte-identical.

## Who this is for

Agencies and freelancers who ship small static sites —
trades, clinics, local services — and currently either
hand-write these pages per client or pass a per-domain
consent subscription through forever. Ten client sites on a
per-domain plan is roughly $250/month for documents that
never change; here they are six files per site, generated
once, owned outright, and re-generated when the facts
change.

## Internal use

This tool does a real job for its author: **Focus
Brilliant**, the author's website-building service for local
trades and clinic clients, ships these pages and the banner
as a named deliverable on every client site instead of
renting a consent platform per domain. The bundled sample is
the shape of a real Focus Brilliant client profile (a
Kelowna plumbing and heating company; names, addresses and
contacts are fictional).

## Known limits

- **Templates, not law.** The clause text is generic
  starting-point wording. It is not jurisdiction-specific
  drafting, it does not track changes in any law, and it is
  not a substitute for a qualified professional's review.
- **No consent log, no TCF, no blocking.** The banner
  records the visitor's choice in their own browser only.
  Regimes or clients that need server-side consent records,
  IAB TCF strings, or automatic script blocking need more
  than this tool — it says so in its own output.
- **Two variants only.** `generic` and Canada-flavoured
  labels. The variant changes names and pointers (e.g. the
  Privacy Commissioner of Canada), not legal substance.
- **The profile is the truth.** If the profile says the
  site sets no cookies, the pages say so — the tool cannot
  audit the real site. `check` catches the profile
  contradicting *itself*; it cannot catch the profile
  contradicting the website.
- **Placeholders are counted in the documents**, not in
  `manifest.txt` (which lists them, so counting it would
  double-count). A quick `grep -r "TO FILL" out/` over a
  fresh generation will show the manifest's listings too —
  that is expected.

## Privacy

What it touches: only the profile file you point it at and
the `--out` folder. What it sends: nothing — there is no
network code in this tool, and none in the files it
generates.

## Support

If this replaces one month of per-domain consent rent,
consider sponsoring: **https://github.com/sponsors/jasveerai88**

## Licence

MIT — see [LICENSE](LICENSE).
