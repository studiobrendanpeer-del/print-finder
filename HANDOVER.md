# Print Finder — handover

A static site (GitHub Pages) that searches open-access museum collections and shows every public-domain image that prints at A4 or larger at the chosen DPI (default 200). Each card shows the largest A size (A4 … A0, 2A0, 4A0) the image can print, directly under the thumbnail.

## Files
- `index.html` — the whole app (HTML/CSS/JS, no build step). Calls museum APIs directly from the browser.
- `nga-index.json` — pre-built index of ~59k National Gallery of Art (Washington) open-access images (long side ≥ 2,480 px), built from github.com/NationalGalleryOfArt/opendata (`published_images.csv` + `objects.csv`). Row format: `[uuid, width, height, title, artist, displaydate, beginyear, endyear, classCode(p/d/r/f/o), medium, keywords, objectid]`. 15 MB (≈5 MB gzipped); the keywords column is about a third of it, so trimming it is the easy win if load time matters.

## Sources (`adapters` in `index.html`; each returns `{items, more}`, `st` = that source's paging state)
Every source except Wikimedia (`vetted:false`) re-checks each result's licence on our side; never rely on a search filter alone.

| key | Source | Notes |
|---|---|---|
| rijks | Rijksmuseum | data.rijksmuseum.nl Linked Art: search → object → VisualItem → DigitalObject → iiif.micr.io info.json (4 requests/item). Items without a rights statement are dropped. Titles mostly Dutch: `NL` maps English theme words to Dutch title words. IIIF v3: crops must use size `w,` (not `full`). |
| nga | National Gallery of Art | Local search over `nga-index.json` (whole words, plurals allowed: `iris` → irises, not Irish), handed out 200 at a time (`NGA_PAGE`). IIIF at api.nga.gov/iiif. |
| cma | Cleveland Museum of Art | openaccess-api.clevelandart.org, `images.full` width/height. Full file is a TIFF. |
| aic | Art Institute of Chicago | api.artic.edu `params` JSON query. Size pre-filter uses `AIC_MIN` (A4 at the lowest dpi option) so changing dpi after a search can't miss images. Download may be capped below master size ("Check downloadable size"). Images are behind a Cloudflare bot check (blocked from cloud servers; fine in home browsers). |
| met | The Met | collectionapi.metmuseum.org **v1.1** search (v1 retired 2026-10-01); `isPublicDomain` param is ignored, so checked per object. No sizes in the API: `jpegSize()` reads the JPEG header (streams ≤256 KB and cancels, even if the server ignores Range). Most originals ≈4,000 px. |
| getty | Getty Museum | SPARQL at data.getty.edu (title keyword match, medium by AAT type, object types → `medium`) → one fetch per image record for CC0 rights, pixel size and IIIF. Object-level CC0 covers metadata only. Title-only search, so "by <artist>" finds nothing here. |
| si | Smithsonian | api.si.edu Open Access, `media_usage:CC0`; size in the search result; CC0 re-checked per image. Key in Settings: DEMO_KEY ≈10 req/hour per IP; free key at api.data.gov/signup. |
| loc | Library of Congress | www.loc.gov/photos/ (or /maps/ when the query says "map") `fo=json`. Keeps "No known restrictions…"/public domain; untagged Geography & Map items allowed per LoC policy. Size from tile.loc.gov IIIF `master:…u/info.json`. Photo-master IIIF can't scale, so thumbs use `r.jpg`/`v.jpg`. Rate limit ≈20 searches/10 s — overuse blocks for ~1 h. |
| mia | Minneapolis Institute of Art | search.artsmia.org Lucene query in the URL path. **Search text goes through `plainWords()`** (no symbols, AND/OR/NOT lower-cased) — "flowers NOT" used to cancel the public-domain clause and return copyrighted work. `rights_type==="Public Domain"` re-checked. Images img.artsmia.org `_400/_800/_full.jpg` (no CORS: no in-browser size check). |
| wellcome | Wellcome Collection | `/catalogue/v2/images` with `locations.license=pdm,cc-0` (re-checked); IIIF info.json for size + parent work for date/format. From cloud servers `iiif.wellcomecollection.org/image/*` blocks normal browser user agents; home browsers fine. |
| ycba / yuag | Yale Center for British Art, Yale University Art Gallery | `yale()` + `YALE` config (LUX set names per medium) → LUX record (date, artist, object number) → IIIF manifest (rights, canvas size, IIIF). `medium` = classification + materials, never the department ("Paintings and Sculpture" would trip the sculpture filter). Yale's CDN blocks non-browser user agents. |
| smk | SMK Denmark | api.smk.dk; `public_domain` re-checked; licence from `rights`. |
| eu | Europeana | wskey in Settings (default `api2demo`). Pre-filters IMAGE_SIZE:extra_large + `RIGHTS:*publicdomain*` (not `reusability=open`, which admits CC BY); rights re-checked; per-record call for pixel size. |
| wm | Wikimedia Commons | Licence tags unreliable — only PD/CC0-tagged kept, flagged "licence varies"; hidden by "Public-domain sources only". |

## How the app works
- **Search state:** `run()` stores the submitted search in `searched`. Rendering and fetching use `cur()` = that search + the live dpi, layout and licence settings. Typing in the form changes nothing until Search is pressed. Each search is a history entry (Back/Forward re-run it).
- **Fetching:** each source pages independently (`sourceLoop`, up to 4 rounds per top-up). Infinite scroll: 48 cards at a time; `maybeMore()` runs after every render and when `#more` comes within `NEAR` px. It only fetches again if the previous batch added visible results (`lastFetchAt`), otherwise it shows a "Search deeper" button — heavy filters can't make it fetch forever.
- **Ordering:** default **Best match** = titles containing every keyword first, then each source's own relevance order (`srank`, so sources interleave). Cards the viewer has scrolled past are pinned (`view.order`) so nothing jumps while scrolling. `resetView()` (any sort/filter/setting change) starts again from the top.
- **Print maths:** `fit()` grades an item against every A size ("contain" dpi = max(long/paperLong, short/paperShort), "cover" = min). `fitOf()` memoises it per dpi/layout in a WeakMap — results are re-filtered and sorted on every redraw (10k results ≈50 ms).
- **Filters after searching:** source pills, shape, A3+/A2+/A1+/A0+, **Artworks only** (`JUNK` regex on each museum's object type / medium text, never titles — "Vase of Flowers" is usually a picture), licence setting. Status counts are after all filters except the source pills.
- **Cards** are cached by id in `cardCache` (cleared on dpi/layout change) and the grid is appended to rather than rebuilt while scrolling, so images don't reload and keyboard focus stays put. Board stars are re-synced on every render.
- **Licence badges** come from each item's own licence (`licLabel`): "CC0" only when it really is CC0, otherwise "Public domain" / "No known restrictions".
- **Boards:** `pf_boards` in localStorage = [{id,name,items:[snapshot]}] (old `pf_shortlist` migrates into "Shortlist"). ☆ Save on a card / "Save to board…" in the detail view opens `boardPicker()`. Boards dialog: switch, create, rename (inline), delete, CSV (UTF-8 BOM, formula-safe), Backup/Import (validated and cleaned through `snap()` before anything merges; links must be http(s)). Other tabs pick up changes via the `storage` event. Browser-only — no sync between devices.
- **Similar** (metadata only, no image matching): card button = longest title word + same medium; detail view chips = each title word (same medium, ±30 yrs), "by <surname>", "any <medium> <years>".
- **Safety:** museum text is only ever inserted escaped (`esc`) or parsed inertly (`strip` uses DOMParser); links pass `safeUrl` (http/https only); pixel sizes are coerced to numbers on arrival.

## Known gaps / ideas
- **Settings → Test all sources** runs one small live search per source from the viewer's browser. Use it first when anything breaks.
- Not verifiable from the cloud sandbox: AIC images, Wellcome full-size images, Wikimedia (intermittent 429), Smithsonian (DEMO_KEY quota). Sandbox allowlist needs `*.collections.yale.edu`, `*.loc.gov`, `*.wellcomecollection.org`, `*.artsmia.org` plus the museum API hosts.
- Not added: NYPL (API sends no CORS headers — unusable from a browser). Candidates needing a free key: Harvard Art Museums, Paris Musées, Te Papa.

## Owner preferences
Brendan is a designer, not a developer. Keep replies short and plain; lead with problems first. Commit straight to `main` (GitHub Pages deploys from `main` / root).
