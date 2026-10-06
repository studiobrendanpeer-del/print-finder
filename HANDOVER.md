# Print Finder — handover

A static site (GitHub Pages) that searches open-access museum collections and only shows public-domain images with enough pixels to print at a chosen paper size/DPI (default A2 @ 200 dpi).

## Files
- `index.html` — the whole app (HTML/CSS/JS, no build step). Calls museum APIs directly from the browser.
- `nga-index.json` — pre-built index of ~59k National Gallery of Art (Washington) open-access images (long side ≥ 2,480 px), built from github.com/NationalGalleryOfArt/opendata (`published_images.csv` + `objects.csv`). Row format: `[uuid, width, height, title, artist, displaydate, beginyear, endyear, classCode(p/d/r/f/o), medium, keywords, objectid]`.

## Sources (adapters in `index.html`, object `adapters`)
| key | Source | Notes |
|---|---|---|
| rijks | Rijksmuseum | data.rijksmuseum.nl Linked Art: search → object → VisualItem → DigitalObject → iiif.micr.io info.json (4 requests/item). Titles mostly Dutch: `NL` dictionary maps English themes to Dutch title words. CORS not yet verified live. |
| nga | National Gallery of Art | Local search over `nga-index.json`. IIIF at api.nga.gov/iiif (full size downloadable). |
| cma | Cleveland Museum of Art | openaccess-api.clevelandart.org, `images.full` width/height. CORS not verified. |
| aic | Art Institute of Chicago | api.artic.edu `params` JSON query; download may be capped below master size ("Check downloadable size" button reads info.json). |
| smk | SMK Denmark | api.smk.dk, `image_width/height`. |
| eu | Europeana | Needs wskey (default public `api2demo`, rate-limited; user key goes in Settings). Search pre-filters IMAGE_SIZE:extra_large + public-domain rights; per-record call for ebucoreWidth/Height. |
| wm | Wikimedia Commons | Licence tags unreliable — only PD/CC0-labelled kept, flagged "licence varies". |

## UI behaviour
- Results show 12 at a time ("Load 12 more"); each source pages independently and stops once enough results are buffered.
- Post-search filters: clickable source chips (multi-select), sort (print quality / pixels / earliest / latest / title), shape (portrait / landscape / square).
- Print maths in `fit()`: "contain" (whole image with margins) dpi = max(long/paperLong, short/paperShort); "cover" (fill sheet) = min(...).
- Shortlist stored in localStorage, CSV export.

## Known gaps / ideas
- Live API CORS for Rijksmuseum, Cleveland and Europeana was never tested from a real browser — check status chips first if anything fails.
- Candidates not yet added: Met (no dimensions in API), Yale Center for British Art, Getty (Linked Art), Smithsonian (needs api.data.gov key).

## Owner preferences
Brendan is a designer, not a developer. Keep replies short and plain; lead with problems first. Commit straight to `main` (GitHub Pages deploys from `main` / root).
