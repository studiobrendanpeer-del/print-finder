# Print Finder — handover

A static site (GitHub Pages) that searches open-access museum collections and shows every public-domain image that prints at A4 or larger at the chosen DPI (default 200). Each card shows the largest A size (A4 … A0, 2A0, 4A0) the image can print, directly under the thumbnail.

## Files
- `index.html` — the whole app (HTML/CSS/JS, no build step). Calls museum APIs directly from the browser.
- `nga-index.json` — pre-built index of ~59k National Gallery of Art (Washington) open-access images (long side ≥ 2,480 px), built from github.com/NationalGalleryOfArt/opendata (`published_images.csv` + `objects.csv`). Row format: `[uuid, width, height, title, artist, displaydate, beginyear, endyear, classCode(p/d/r/f/o), medium, keywords, objectid]`.

## Sources (adapters in `index.html`, object `adapters`)
| key | Source | Notes |
|---|---|---|
| rijks | Rijksmuseum | data.rijksmuseum.nl Linked Art: search → object → VisualItem → DigitalObject → iiif.micr.io info.json (4 requests/item). Titles mostly Dutch: `NL` dictionary maps English themes to Dutch title words. |
| nga | National Gallery of Art | Local search over `nga-index.json`. IIIF at api.nga.gov/iiif (full size downloadable). |
| cma | Cleveland Museum of Art | openaccess-api.clevelandart.org, `images.full` width/height. |
| met | The Met | collectionapi.metmuseum.org **v1.1** search (v1 search retired 2026-10-01). `isPublicDomain` param is ignored, so checked per object. No sizes in the API: `jpegSize()` reads the JPEG header with a Range request (64 KB). Most originals are ~4,000 px, so few pass at A2/200. |
| getty | Getty Museum | SPARQL at data.getty.edu (label keyword match, medium by AAT type) → one fetch per image record (`data.getty.edu/media/image/…`) for CC0 rights, pixel size and media.getty.edu IIIF. Object-level CC0 is metadata only — image rights live on the image record. |
| si | Smithsonian | api.si.edu Open Access, `media_usage:CC0`; pixel size is in the search result. Key in Settings: DEMO_KEY ≈10 req/hour per IP; free personal key at api.data.gov/signup. |
| loc | Library of Congress | www.loc.gov/photos/ (or /maps/ when the query says "map") `fo=json`. Rights from `rights_advisory` (keep "No known restrictions…"/public domain; untagged Geography & Map items allowed per LoC policy). Pixel size from tile.loc.gov IIIF `master:…u/info.json` (name guessed from the service JPEG, falls back to item JSON). Photo-master IIIF can't scale (`full/400,` → 500), so thumbs use the `r.jpg`/`v.jpg` service files. Rate limit ~20 searches/10 s — overuse blocks for ~1 h. |
| mia | Minneapolis Institute of Art | search.artsmia.org Lucene query in the URL path (`rights_type:"Public Domain" image:valid public_access:1`); hits carry true master pixel size, one request per 100 results. Images from img.artsmia.org `…_400/_800/_full.jpg` (no CORS there, so no in-browser size check). Avoid `*.api.artsmia.org/full` — serves older, smaller files. |
| wellcome | Wellcome Collection | api.wellcomecollection.org `/catalogue/v2/images` with `locations.license=pdm,cc-0`; IIIF info.json for pixel size + parent work for date/format. From the cloud sandbox, `iiif.wellcomecollection.org/image/*` 403s normal browser user agents (bot rule on datacenter IPs); home browsers expected fine — check with Test all sources. |
| ycba / yuag | Yale Center for British Art, Yale University Art Gallery | Shared `yale()` helper + `YALE` config (LUX set names per medium). LUX search (lux.collections.yale.edu, `memberOf` its two art collections + `hasDigitalImage`) → LUX record (date, artist, YCBA object number from thumbnail URL) → IIIF manifest at manifests.collections.yale.edu/{ycba|yuag}/obj/N (CC0 `rights`, canvas pixel size, images.collections.yale.edu IIIF). Yale's CDN blocks non-browser user agents (curl gets 403); browsers are fine. |
| aic | Art Institute of Chicago | api.artic.edu `params` JSON query; download may be capped below master size ("Check downloadable size" button reads info.json). |
| smk | SMK Denmark | api.smk.dk, `image_width/height`. |
| eu | Europeana | Needs wskey (default public `api2demo`, rate-limited; user key goes in Settings). Search pre-filters IMAGE_SIZE:extra_large + `RIGHTS:*publicdomain*` (not `reusability=open`, which admits CC BY and emptied whole pages); per-record call for ebucoreWidth/Height. |
| wm | Wikimedia Commons | Licence tags unreliable — only PD/CC0-labelled kept, flagged "licence varies". |

## UI behaviour
- Infinite scroll: 48 cards at a time; an IntersectionObserver on `#more` plus `maybeMore()` (re-checked after every render) adds cards and asks sources for more when the bottom is within 1,200 px. `lastFetchAt` stops auto-fetching after a round that found nothing new (then `#more` offers a click to search deeper). Cards are cached in `cardCache` (keyed by id+dpi+mode) so images don't reload on re-render.
- Post-search filters: clickable source chips (multi-select), sort (print quality / pixels / earliest / latest / title), shape (portrait / landscape / square).
- Print maths in `fit()`: for every A size in `PAPER`, "contain" (whole image with margins) dpi = max(long/paperLong, short/paperShort); "cover" (fill sheet) = min(...). `maxA` = largest size still ≥ chosen dpi; anything below A4 is dropped. There's no paper selector any more. Post-search "A3+ / A2+ / A1+ / A0+" chips filter by `maxA`, and "Largest print size" sort uses it too.
- Default sort **Best match**: results whose title contains every keyword first, then each source's own relevance order (`it.srank`, so sources interleave), then max A size.
- **Artworks only** chip (on by default) hides items whose *medium / object-type text* matches `JUNK` (ceramics, jade, metalwork, sculpture, furniture, costume, coins…). Titles are deliberately not checked ("Vase of Flowers" is usually a painting). Getty's medium is filled from its SPARQL object types (e.g. "Decorative Arts", "Photographs") for this. Europeana and Wikimedia carry no object type, so nothing is filtered there.
- **Boards** (replaced the shortlist): `pf_boards` in localStorage = [{id,name,items:[snapshot]}]; an old `pf_shortlist` is migrated into a board called "Shortlist". ☆ on a card / "Save to board…" in the detail view opens `boardPicker()` (tick boards, or type a new name). Boards dialog: switch board, rename, delete, CSV per board, Backup (JSON download) / Import (merges by board name). Browser-only — no sync between devices.
- NGA local search matches whole words with optional plural (`iris` → irises, not Irish).

## Known gaps / ideas
- **Settings → Test all sources** runs one small live search per source from the viewer's browser and reports Works / Failed / thumbnail / size-check status. Use it first when anything breaks.
- All 10 sources verified working from headless Chromium on 2026-10-07. Not verifiable from the cloud sandbox: AIC images (Cloudflare bot check on datacenter IPs), Wikimedia (intermittent 429 on shared IPs), Smithsonian thumbnails (ids.si.edu not allowlisted).
- Sandbox allowlist needs `*.collections.yale.edu`, `*.loc.gov`, `*.wellcomecollection.org`, `*.artsmia.org` for testing.
- Not added: NYPL (API sends no CORS headers — unusable from a browser). Candidates needing a free key: Harvard Art Museums, Paris Musées, Te Papa.

## Owner preferences
Brendan is a designer, not a developer. Keep replies short and plain; lead with problems first. Commit straight to `main` (GitHub Pages deploys from `main` / root).
