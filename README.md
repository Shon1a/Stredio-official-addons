# Stredio-official-addons

The curated **official add-on list** for [STREDIO](https://github.com/Shon1a/Stredio) — static JSON, served over jsDelivr and read straight by the browser. The STREDIO server is never involved, the same way [`stredio-translations`](https://github.com/Shon1a/stredio-translations) holds the UI text.

Entries are **metadata / discovery only**. `transportUrl` and `resources: ["stream"]` are rejected by the schema, by CI, and by the app loader — streaming add-ons are added by users themselves, in-app.

## Files

| File | Purpose |
|------|---------|
| `index.json` | Manifest — schema version + which payload file(s) to load + the protected ids. |
| `addons.json` | The official add-on entries. |
| `schema/*.json` | JSON Schemas for both files. |
| `.github/workflows/validate.yml` | CI: schema + no-stream + protected-ids checks. |

## How it loads

```
https://cdn.jsdelivr.net/gh/Shon1a/Stredio-official-addons@master/index.json
```

The browser reads `index.json`, then the `addons.json` it points to, and merges the entries over STREDIO's built-in defaults. If the fetch fails or `schema` ≠ `1`, the built-in list stands — the CDN only ever refines existing cards or adds new ones, never removes them.

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
- Metadata/discovery only — no `transportUrl`, streams, or debrid config.
- Bump `version` in both files on every change; keep `schema: 1` for additive edits.

CI blocks anything that breaks these rules.

## License

[MIT](./LICENSE) — data and metadata only; no media, no stream sources.
