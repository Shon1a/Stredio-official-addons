# Changelog

All notable changes to the Stredio official add-on collection are documented here.
Bump `version` in both `index.json` and `addons.json` on every change; keep
`schema: 1` for additive changes and reserve `schema: 2` for a breaking redesign.

## 2026.07.01 — schema 1 (initial)

- Initial externalization of STREDIO's built-in official add-on cards from the
  inline `const ADDONS` in `index.html`.
- Two-file layout mirroring `stredio-translations`: `index.json` (manifest) +
  `addons.json` (collection payload).
- Four official, metadata/discovery-only entries: `upcoming`, `studios`,
  `catalog`, `providers`. Display fields byte-identical to the inline source, so
  the CDN merge is a no-op for the four.
- Canonical fields (`version`, `types`, `resources`, `flags:{official,protected}`,
  `kind:"discovery"`) added alongside the UI-flat fields for a future
  `Stredio-Heart` core.
- Install flag exposed as `defaultInstalled` (honored only for new ids).
- Schemas (`schema/manifest.schema.json`, `schema/addon-collection.schema.json`)
  and CI (`.github/workflows/validate.yml`) enforce the neutral-conduit stance:
  no `transportUrl`, no `resources:["stream"]`, and the four protected ids must
  always be present.
