# Contributing

Contributions are welcome — bug reports, fixes and small focused features.

## Ground rules

- **Standard library only.** compliance-pages' promise is
  "clone and run". Pull requests that add a third-party dependency
  will not be merged.
- **Local only, and it sends nothing.** No network calls, no
  telemetry, no uploads of user files — and the generated files
  stay network-free too (dependency-free CSS/JS, no external
  requests; the only URL that may appear is the site's own,
  printed as text from the profile). A pull request that adds a
  hosted consent log, analytics or a remote banner changes what
  this tool *is* and will not be merged.
- **Facts come from the profile, or they don't appear.** The
  generator may only print what the profile states. A change
  that lets the tool invent, default or "helpfully" fill a fact
  — a data item, a retention period, a governing law, a
  consent-gating claim — will not be merged. Unstated facts
  ship as `[TO FILL: ...]` placeholders: loud, counted in the
  manifest, and reflected in the exit code. Keep them loud.
- **It is never "compliance".** The output is a template with a
  not-legal-advice notice baked into every page. Don't add
  wording — in code, docs or generated text — that calls the
  output compliant, certified or legally sufficient, and don't
  remove or soften the template notice or the honesty wording
  in the manifest.
- **The banner records; it does not block.** consent.js stores
  the visitor's choice in their own browser and publishes it
  (`window.cpConsent`, the `cp:consent` event). Don't add
  claims that it enforces consent, and keep it dependency-free
  vanilla JS with no network strings (the tests grep for them).
- **Rendering stays deterministic and self-contained.** The
  same profile + same date must render byte-identical files
  (the stamp date is an input, never the clock), and the HTML
  stays a single file with inline CSS and every profile string
  HTML-escaped. Tests pin both properties; keep them passing
  when you touch a template.
- **Validation reports everything at once.** Structural
  problems are collected and named together so the author fixes
  the profile once. Don't regress to fail-on-first-problem, and
  don't let a broken profile write partial output — nothing is
  written until the profile is structurally clean. (Missing
  *facts* are not structural: they generate, as placeholders,
  by design.)
- **Both directions tested.** A behaviour change needs a test
  where a complete profile succeeds cleanly and an incomplete
  or contradictory one is flagged — not just one side.
  Placeholder counts in tests must be hand-derived from the
  fixture (see the placement map in the test docstring), not
  copied from the tool's own output.
