# Stredio-official-addons

Canonical, UI-agnostic **official add-on collection** for [STREDIO](https://github.com/Shon1a) — a Netflix-style multilanguage streaming *catalog* UI. This repo is the external, CDN-served source of truth for STREDIO's built-in **official** add-on cards, exactly the way [`Shon1a/stredio-translations`](https://github.com/Shon1a/stredio-translations) is the external source of truth for UI copy.

It is **100% static data** served over jsDelivr straight from GitHub. STREDIO's browser reads it directly — the STREDIO server (Render) is never touched, so it stays free-tier-safe at any scale.

---

## ⚠️ Neutral-conduit stance (read this first)

STREDIO is a **neutral conduit / personal media library**. It ships **NO stream sources**.

Every entry in this repo is **metadata / discovery ONLY** (`kind: "discovery"`). These cards are internal TMDB-backed discovery features (home rows, marquees) presented as add-on cards — they arrange catalog metadata, they do not host, proxy, or fetch media.

**Forbidden anywhere in this repo** (blocked by schema + CI + the runtime loader):

- `transportUrl`
- `resources` containing `"stream"`
- any bundled source endpoint, debrid key, or stream provider config

Users add their own third-party stream add-ons **by URL, in-app** (the Community section). This repo must never turn STREDIO into a bundled stream provider.

---

## Repository layout

| Path | Role | Analog in `stredio-translations` |
|------|------|----------------------------------|
| `index.json` | **Manifest** — tiny, eager-loaded; lists the collection payload file(s), the schema major, the bump-per-edit `version`, and the protected ids. | `index.json` (lists `languages`) |
| `addons.json` | **Payload** — the official add-on collection (the four entries). | `<code>.json` (per-language strings) |
| `schema/manifest.schema.json` | JSON Schema (draft 2020-12) for `index.json`. | — |
| `schema/addon-collection.schema.json` | JSON Schema (draft 2020-12) for `addons.json`; encodes the neutral-conduit guardrails. | — |
| `.github/workflows/validate.yml` | CI: schema validation + protected-id + no-stream assertions. | — |
| `addons/<id>.config.json` | **Optional / future** lazy per-entry Configure-form payload, referenced by an entry's `configRef`, fetched on demand only. Not used by the four today. | (the lazy `load(code)` split) |
| `CHANGELOG.md` / `LICENSE` | History + license (data/metadata only). | — |

The two-file split (manifest + payload) intentionally mirrors `stredio-translations` so both content repos are learned and loaded the same way.

---

## CDN URL pattern

Serve everything through jsDelivr, pinned to `@master` (same edge-cache behaviour as the translations repo):

```
https://cdn.jsdelivr.net/gh/Shon1a/Stredio-official-addons@master/index.json
https://cdn.jsdelivr.net/gh/Shon1a/Stredio-official-addons@master/addons.json
```

Base const in STREDIO (same literal style as the i18n `CDN` const):

```js
var ADDONS_CDN = 'https://cdn.jsdelivr.net/gh/Shon1a/Stredio-official-addons@master/';
```

> jsDelivr/GitHub resolve the owner/repo case-insensitively. For a deterministic rollout you can pin a tag or commit instead of `@master` (e.g. `@2026.07.01`); jsDelivr caches `@master` at the edge for up to ~12h, so tag-pinning is the way to force an instant refresh.

---

## `index.json` — manifest

```json
{
  "schema": 1,
  "version": "2026.07.01",
  "updated": "2026-07-01T00:00:00Z",
  "repo": "Shon1a/Stredio-official-addons",
  "cdn": "https://cdn.jsdelivr.net/gh/Shon1a/Stredio-official-addons@master/",
  "collections": [
    { "id": "official", "section": "official", "file": "addons.json", "count": 4, "protectedIds": ["upcoming", "studios", "catalog", "providers"] }
  ]
}
```

- `schema` — MUST be `1` for today's UI. Any other value is treated like a failure and the consumer keeps its inline default set (version negotiation).
- `version` — bump on **every** edit. Also update `updated`.
- `collections[]` — each entry names a payload `file` to fetch. Today there is one: the `official` collection in `addons.json`.

## `addons.json` — collection payload

Envelope `{ schema, version, updated, repo, section, addons: Entry[] }`. Each **Entry** carries a canonical core **plus** the UI-flat fields today's single-file UI reads with zero transform:

| Field | Req | Meaning |
|-------|-----|---------|
| `id` | ✅ | Canonical primary key. **Must be byte-identical** to the live keys — it drives `ADDONS.find`, `data-*` attrs, `localStorage['stredio.addons']`, `/api/addon-state`, and the i18n key stems `addon.<id>.type` / `addon.<id>.desc`. |
| `section` | ✅ | `"official"` (only official ships here) or `"community"` (ships empty; users add these in-app). |
| `name` | ✅ | Fallback title. Localized copy is separate (see below). |
| `tags` | ✅ | `string[]`; each maps to translation key `tag.<tag>`. |
| `kind` | ✅ | `"discovery"` — documents metadata-only, no streams. |
| `version` | | Semver **without** leading `v` — canonical for a future core. |
| `ver` | | UI value **with** leading `v` (e.g. `v1.3.0`). If absent the loader derives it as `'v' + version`. |
| `iconCls`, `glyph`, `img` | | UI icon hints. `iconCls` default `puzzle`; it is emitted into a `class` attribute, so the loader only accepts `iconCls` matching `^[a-z0-9 _-]{0,40}$` (anti-XSS) — use plain class tokens. |
| `defaultInstalled` | | Default install state. **Honored only for NEW ids**; ignored for the four (their default is inline, their real state is persisted). Loader also tolerates legacy `installed`. |
| `noConfig`, `preview`, `locked` | | UI behaviour flags. For the four these stay **inline-canonical** — the CDN cannot change them (they gate the Configure/Preview buttons). Used only when defining a brand-new card. |
| `types`, `resources` | | Content types / resources. `resources` **MUST NOT** contain `"stream"`. |
| `flags` | | `{ official, protected }`. `protected: true` marks the behaviour-critical four that merge must never drop. |
| `configRef` | | Optional path to a future `addons/<id>.config.json` lazy payload. |

### Human copy lives in `stredio-translations`, not here

Card type/description/tag **labels** are localized in the translations repo under:

- `addon.<id>.type` — e.g. `addon.upcoming.type`
- `addon.<id>.desc` — e.g. `addon.upcoming.desc`
- `tag.<tag>` — e.g. `tag.catalog`, `tag.metadata`

This repo carries **only** ids/structure. The two repos are joined by the shared, byte-stable `id` and `tag` strings.

---

## How STREDIO loads it (two-step, mirrors the i18n loader)

This self-contained IIFE lives in `index.html` **immediately after the add-ons click-handler block**. Nothing earlier changes: the inline `const ADDONS`, the `loadAddonState` IIFE, the sync layer, `renderAddons()`, and the home-row reconcile all still run **first** against the inline four. The CDN is a pure async **upgrade** layer, and it repaints only the off-screen Add-ons grid — never the home rows, never the server.

```js
(function(){
  var ADDONS_CDN='https://cdn.jsdelivr.net/gh/Shon1a/Stredio-official-addons@master/';
  var DISPLAY=['name','ver','iconCls','glyph','tags','img']; // CDN may refine ONLY these on known ids
  var ICON_OK=/^[a-z0-9 _-]{0,40}$/i;                        // iconCls goes into a class attr; keep CDN values inert
  function safeIcon(v){ return (typeof v==='string'&&ICON_OK.test(v))?v:null; }
  function hasStream(raw){ return raw.transportUrl!=null || (Array.isArray(raw.resources)&&raw.resources.some(function(r){return r==='stream'||(r&&r.name==='stream');})); }
  function verOf(raw){ if(typeof raw.ver==='string') return raw.ver; if(typeof raw.version==='string') return 'v'+raw.version; return undefined; }
  function upsertKnown(cur,raw){                 // refine an existing id (the four): DISPLAY only. returns true iff changed
    var changed=false;
    DISPLAY.forEach(function(f){
      if(f==='tags'){ if(Array.isArray(raw.tags)){ var tg=raw.tags.filter(function(t){return typeof t==='string';}); if(JSON.stringify(tg)!==JSON.stringify(cur.tags)){ cur.tags=tg; changed=true; } } return; }
      if(f==='ver'){ var vv=verOf(raw); if(typeof vv==='string'&&vv!==cur.ver){ cur.ver=vv; changed=true; } return; }
      if(f==='iconCls'){ var ic=safeIcon(raw.iconCls); if(ic!=null&&ic!==cur.iconCls){ cur.iconCls=ic; changed=true; } return; }
      if(!(f in raw)) return;                     // absent -> keep inline value (never wipe)
      var v=raw[f]; if((typeof v==='string'||v===null)&&v!==cur[f]){ cur[f]=v; changed=true; }
    });
    if(!Array.isArray(cur.tags)) cur.tags=[];     // crash-guard for a.tags.map in addonCardHTML
    return changed;
    // id, section, installed, locked, noConfig, preview are NEVER copied onto a known entry.
  }
  function coerceNew(raw){                        // brand-new curated official card
    var v=verOf(raw);
    return { id:raw.id, section:'official',
      name:typeof raw.name==='string'?raw.name:raw.id,
      ver:(typeof v==='string'?v:''),
      iconCls:safeIcon(raw.iconCls)||'puzzle',
      glyph:typeof raw.glyph==='string'?raw.glyph:'',
      tags:Array.isArray(raw.tags)?raw.tags.filter(function(t){return typeof t==='string';}):[],
      installed:(raw.defaultInstalled===true)||(raw.installed===true)||false, // DEFAULT only
      noConfig:!!raw.noConfig, preview:!!raw.preview, locked:!!raw.locked,
      img:typeof raw.img==='string'?raw.img:undefined };
  }
  function merge(list){
    var changed=false;
    list.forEach(function(raw){
      if(!raw||typeof raw.id!=='string'||!raw.id) return;   // shape guard
      if(raw.section!=='official') return;                  // community ships empty; skip non-official
      if(hasStream(raw)) return;                            // neutral-conduit runtime guard
      var cur=ADDONS.find(function(x){return x.id===raw.id;});
      if(cur){ if(upsertKnown(cur,raw)) changed=true; } else { ADDONS.push(coerceNew(raw)); changed=true; }
    });
    return changed;
  }
  function reapplyLocal(){                          // persisted toggle beats CDN default for any NEW id
    try{ var raw=localStorage.getItem(ADDON_KEY); var s=raw?JSON.parse(raw):{};
      ADDONS.forEach(function(a){ if(!a.locked&&typeof s[a.id]==='boolean') a.installed=s[a.id]; }); }catch(e){}
  }
  async function fetchJSON(file){ try{ var r=await fetch(ADDONS_CDN+file,{cache:'force-cache'}); if(r.ok) return await r.json(); }catch(e){} return null; }
  async function loadOfficialAddons(){
    var man=await fetchJSON('index.json');                            // STEP 1: manifest
    if(!man||man.schema!==1||!Array.isArray(man.collections)) return; // fail/unknown-major -> inline four stand
    var off=man.collections.find(function(c){return c&&c.section==='official'&&typeof c.file==='string';});
    if(!off) return;
    var data=await fetchJSON(off.file);                               // STEP 2: payload
    if(!data||data.schema!==1||!Array.isArray(data.addons)) return;
    if(!merge(data.addons)) return;                                   // identical data -> zero re-render/refetch
    reapplyLocal();
    try{ if(typeof renderAddons==='function') renderAddons(); }catch(e){}  // repaint ONLY the Add-ons grid
  }
  loadOfficialAddons();                             // fire-and-forget, like loadManifest()
})();
```

### Fallback (layered, like i18n)

1. **Inline `const ADDONS` is the always-present default** — the app is fully functional with the four cards and correct home rows before/without any CDN response.
2. **Silent try/catch** — on network/parse/404/CORS failure the data stays `null` and the loader returns early; the inline four stand.
3. **Granular degradation** — a known id absent from the CDN keeps its inline metadata (upsert only overwrites *present* fields); a CDN entry missing `tags` is coerced to `[]`; card text still resolves through `stredio-translations`.
4. **`schema !== 1`** is treated as failure → inline four only.

### Zero-regression guarantees

- The CDN may **refine display** of the four (name/ver/icon/glyph/tags) and **add brand-new curated official cards**. It can **never** drop, lock, re-section, flip the on/off default, or change the Configure/Preview behaviour of the four — `id`, `section`, `installed`, `locked`, `noConfig`, and `preview` are never copied onto a known entry, and `defaultInstalled` is honored only for genuinely new ids.
- Ids stay byte-identical, so `localStorage['stredio.addons']` and `/api/addon-state` keep resolving.
- Identical CDN data is a **no-op** (change detection), and any real refinement repaints **only** the Add-ons grid — never `renderHome()`, never a server call.

> Install-state precedence: **inline default (the four) → localStorage → server (`pullAddonState`)**, with the CDN's `defaultInstalled` participating ONLY as the seed for genuinely new ids.

---

## Forward-compat: how a future `Stredio-Heart` core consumes this

STREDIO plans a modular rebuild — a reusable **`Stredio-Heart`** core (a Rust crate holding the logic shared across device/versions) plus thin UI shells. This repo is shaped so that core can read `index.json` → `addons.json` **directly as its official collection, with no shim**:

1. **Canonical fields** — `id`, `name`, `version` (semver), `types`, `resources`, `flags:{official,protected}` — let the core learn each entry's identity/capabilities natively. Today's single-file UI keeps reading the parallel UI-flat fields (`ver`, `iconCls`, `glyph`, `tags`, `defaultInstalled`, ...) with zero transform.
2. **`kind:"discovery"` + the schema/CI-enforced absence of `transportUrl` / `resources:[stream]`** encode the neutral-conduit stance in the data model itself, so the stance survives the rewrite. If an official card ever legitimately becomes a real network add-on, it simply gains an `https` `transportUrl` and the same list holds both kinds; the core dispatches by transport, the UI keeps rendering cards.
3. **Install state is deliberately NOT canonical here** — this repo carries only `defaultInstalled` (a per-user *default hint*). Real per-account state lives in the toggle map (`localStorage['stredio.addons']` + `/api/addon-state`), which the core would own separately. Clean core-data vs per-user-state separation.
4. **Version negotiation** — additive changes stay `schema: 1` (old consumers ignore unknown fields); a breaking redesign bumps to `schema: 2`, at which point any consumer that doesn't understand it falls back to its inline default set rather than mis-rendering.
5. **Two pre-wired extension points, no shape change today**: the manifest+lazy split (`configRef` → `addons/<id>.config.json`, fetched on demand), and the additive nature of the collection.
6. **Human copy stays external** in `stredio-translations`, keyed by `id`, so the core stays locale-agnostic.

> Migration note: the short ids (`upcoming`/`studios`/`catalog`/`providers`) are kept because they are the live `localStorage` + `/api/addon-state` keys and MUST stay byte-identical. A future reverse-DNS id scheme can be introduced via a single id-alias mapping without breaking state or i18n keys.

---

## Editing rules

- **Never delete or rename** the four protected ids: `upcoming`, `studios`, `catalog`, `providers`. (CI blocks it.)
- Entries are **metadata/discovery ONLY** — never add `transportUrl`, streams, or debrid config. (Schema + CI block it.)
- Put human-readable **type/desc/tag labels** in `stredio-translations` under `addon.<id>.type|desc` and `tag.<tag>` — not here. A new curated id/tag needs matching keys there or the card renders raw key strings.
- The **Community** section ships empty — users add third-party add-ons by URL in-app.
- Bump `version` (and `updated`) in both `index.json` and `addons.json` on every edit; keep `schema: 1` for additive changes.
- Keep `collections[].count` and `collections[].protectedIds` in `index.json` in sync with `addons.json` (CI checks count).

---

## License

See [`LICENSE`](./LICENSE). Covers the data/structure/metadata only — no media, no stream sources.
