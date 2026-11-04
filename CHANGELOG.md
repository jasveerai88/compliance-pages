# Changelog

## v0.1.0 — 2026-11-04

First release.

- **`make` — one site profile becomes the pages.** A single
  JSON profile of stated facts renders `privacy.html`,
  `terms.html` and `cookies.html` (a subset may be requested
  with `pages`), plus a self-hosted consent banner
  (`banner.html` + `consent.js`) and a `manifest.txt`, into
  the `--out` folder. Each page is one self-contained HTML
  file: inline CSS, no JavaScript of its own, no external
  requests; the only URL that can appear is the site's own,
  printed as text from the profile. Every profile string is
  HTML-escaped, so markup in a business name renders as
  text.
- **Templates, stamped as templates.** Every generated page
  carries a visible notice at the top — a starting point,
  not legal advice — and a footer stamp naming the tool,
  the generation date and the profile it came from. The
  tool never describes its output as compliant with any
  law. Two variants, chosen by `business.country`: generic,
  and Canada-flavoured (which names the Office of the
  Privacy Commissioner of Canada in the privacy page).
- **Facts only from the profile — gaps stay loud.** A fact
  the documents need but the profile does not state becomes
  a canonical `[TO FILL: ...]` placeholder in the output,
  highlighted in the page, listed and counted in the
  manifest, and `make` exits 1 while any remain. Sections
  appear only when the facts call for them: no analytics in
  the profile, no analytics section; `quotes_are_estimates`
  off, no quotes section. An absent `cookies` block means
  "not stated" (placeholder) — never silently read as "no
  cookies".
- **The manifest is the audit trail.** It lists the files
  written, the sections generated in each, every profile
  value used (or "(not stated)"), the exact placeholder
  count and where each marker stands, any contradictions,
  and the owner wiring still to do. Generation is
  deterministic: same profile + same date → byte-identical
  files, manifest included. The stamp date comes from the
  profile's `as_of` or `--as-of` — never from the clock;
  with neither, `make` is a clean exit-2 error.
- **`check` — audit a profile without writing anything.**
  Reports the gaps (unstated facts, each with the
  placeholder it will ship as, plus an unset generation
  date) and the contradictions: a cookies page requested
  while the profile says the site sets no cookies; an
  analytics service declared while analytics cookies are
  marked off; optional cookies declared with no
  `consent_note` describing how those scripts are gated on
  consent. Exit 0 complete, 1 gaps or contradictions.
- **A banner that records a choice — and blocks nothing.**
  `consent.js` is dependency-free vanilla JavaScript: it
  makes no network requests, stores the visitor's choice in
  the visitor's own browser local storage
  (`cp-consent-v1`), and publishes it as
  `window.cpConsent.get()` and a `cp:consent` event so the
  owner's scripts can honour it (with
  `window.cpConsent.reset()` to clear the stored choice and
  re-show the banner). No server-side consent
  log, no IAB TCF. The category checkboxes in `banner.html`
  exist only for optional categories the profile declares.
- **Structural validation reports everything at once.** A
  profile with wrong types, an unknown page name, an
  impossible date or a half-given `cookies` block fails
  with every problem named, exit 2, and nothing written. A
  corrupt profile is a clean error and is never modified.
- **Agent-ready.** Both commands take `--json` (one JSON
  object; errors as `{"error": ...}`, argparse errors
  included), and SKILL.md documents the profile schema, the
  JSON shapes, the exit codes (0 ok / 1 attention / 2
  error) and the honesty rules.
