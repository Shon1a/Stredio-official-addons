# Stredio-official-addons

The curated **official add-on list** for [STREDIO](https://github.com/Shon1a/Stredio) — static JSON, loaded by the browser at runtime and merged over the app's built-in defaults. A content repo alongside [`stredio-translations`](https://github.com/Shon1a/stredio-translations) (which holds the UI text).

Entries are **metadata / discovery only**. Stream sources are rejected by the schema, by CI, and by the app — streaming add-ons are added by users themselves, in-app.

## Files

| File | Purpose |
|------|---------|
| `index.json` | Manifest — schema version + which payload file(s) to load + the protected ids. |
| `addons.json` | The official add-on entries. |
| `schema/*.json` | JSON Schemas for both files. |
| `.github/workflows/validate.yml` | CI: schema + no-stream + protected-ids checks. |

## How it loads

The browser reads `index.json`, then the payload it points to, and merges the entries over STREDIO's built-in defaults. If the fetch fails or the schema version is unknown, the built-in list stands — the list only ever refines existing cards or adds new ones, never removes them.

## Entry shape

```json
{
  "id": "upcoming",
  "section": "official",
  "name": "Upcoming Movies & Series",
  "tags": ["catalog", "metadata"],
  "kind": "discovery",
  "version": "1.3.0",
  "defaultInstalled": true
}
```

Card labels (type / description / tag text) are **not** here — they live in [`stredio-translations`](https://github.com/Shon1a/stredio-translations), keyed by `id` and `tag`.

## Editing

- Never remove or rename the protected ids: `upcoming`, `studios`, `catalog`, `providers`.
- Metadata / discovery only — no stream sources.
- Bump `version` in both files on every change; keep the schema version at `1` for additive edits.

CI blocks anything that breaks these rules.

## License

[MIT](./LICENSE) — data and metadata only; no media, no stream sources.
