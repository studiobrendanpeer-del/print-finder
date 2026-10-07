# Print Finder — handover

A static site (GitHub Pages) that searches open-access museum collections and only shows public-domain images with enough pixels to print at a chosen paper size/DPI (default A2 @ 200 dpi).

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
| aic | Art Institute of Chicago | api.artic.edu `params` JSON query; download may be capped below master size ("Check downloadable size" button reads info.json). |
| smk | SMK Denmark | api.smk.dk, `image_width/height`. |
| eu | Europeana | Needs wskey (default public `api2demo`, rate-limited; user key goes in Settings). Search pre-filters IMAGE_SIZE:extra_large + `RIGHTS:*publicdomain*` (not `reusability=open`, which admits CC BY and emptied whole pages); per-record call for ebucoreWidth/Height. |
| wm | Wikimedia Commons | Licence tags unreliable — only PD/CC0-labelled kept, flagged "licence varies". |

## UI behaviour
- Results show 12 at a time ("Load 12 more"); each source pages independently and stops once enough results are buffered.
- Post-search filters: clickable source chips (multi-select), sort (print quality / pixels / earliest / latest / title), shape (portrait / landscape / square).
- Print maths in `fit()`: "contain" (whole image with margins) dpi = max(long/paperLong, short/paperShort); "cover" (fill sheet) = min(...).
- Shortlist stored in localStorage, CSV export.

## Known gaps / ideas
- **Settings → Test all sources** runs one small live search per source from the viewer's browser and reports Works / Failed / thumbnail / size-check status. Use it first when anything breaks.
- All 10 sources verified working from headless Chromium on 2026-10-07. Not verifiable from the cloud sandbox: AIC images (Cloudflare bot check on datacenter IPs), Wikimedia (intermittent 429 on shared IPs), Smithsonian thumbnails (ids.si.edu not allowlisted).
- Not yet added: Yale Center for British Art (lives on lux.collections.yale.edu; needs that host allowlisted to build/test).

## Owner preferences
Brendan is a designer, not a developer. Keep replies short and plain; lead with problems first. Commit straight to `main` (GitHub Pages deploys from `main` / root).
