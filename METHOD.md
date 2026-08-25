# How the data on this page was gathered

Target: accommodation within ~20 minutes' drive of Ortisei (Val Gardena), for
2 adults, 5–10 September 2026, sorted cheapest first.

## Why not scrape each hotel website

The area has ~1,500 licensed properties on dozens of different booking engines,
most behind Cloudflare. Going site by site is neither cheap nor reliable. Every
number here comes instead from a **registry or a tourist-board search engine**,
each of which answers for hundreds of properties in a single request.

## The four sources

**1. Open Data Hub Südtirol** — `https://tourism.opendatahub.com/v1/Accommodation`
Official provincial registry, no API key. Filter by municipality with
`locfilter=mun<MUNICIPALITY_ID>` (ids from `/v1/Municipality`). Returns name,
address, type, `AccoProperties.HasApartment`, website, and `Mapping.lts.rid` —
the LTS id that joins to the tourist-board systems below.
This gave the master list: **1,502 licensed properties in the six municipalities.**

`/v1/AccommodationAvailable` exists and aggregates both booking channels, but
returns `401 User not allowed for availabilitysearch` — it needs a partner account.

**2. Booking Südtirol / HGV widget API** — `https://api.widgets.bookingsuedtirol.com/v6`
Open, no key. Per property id:
- `/properties/{id}?lang=en` — name, address, boards, `isBookable`
- `/properties/{id}/rooms?lang=en&sourceId=98` — units with `features_view`
  (kitchen shows up as "Eat-in kitchen" / "Separate kitchen")
- `/properties/{id}/offers?from=&to=&guests=[[18,18]]&lang=en&sourceId=98`
  — `rates[]` with `room_id`, `service` (1 none, 2 B&B, 3 half board, 4 full board)
  and `price_total` in EUR for the whole stay
- `/properties/{id}/availabilities?from=&to=&guests=[[18,18]]` — arrival→departure
  matrix, used as an independent check on the offers endpoint

Property ids are dense in **9800–14599**; enumerating that range yields the full
directory (**2,835 properties**, 440 of them inside the target ring). The endpoint
rate-limits and returns an HTML error page rather than an error status, so any
enumeration must re-fetch anything that did not parse as JSON — otherwise a large
share of the directory silently goes missing.

**3. LTS hotelfinder widget** — `https://widget.lts.it/hotelfinderv2/HotelFinderRender`
Used by the Alpe di Siusi tourist board (`widgetId=ext-seiseralmmarketing`).
POST JSON to the same URL the page loads:
```json
{"command":"search","component":"ListView","view":"list",
 "value":"{\"from\":\"2026-09-05\",\"to\":\"2026-09-10\"}",
 "slideId":"lts-hotelfinder","sessionId":"<any>","referer":"https://widget.lts.it/"}
```
Then `{"command":"pager", ... "value":"{\"page\": N}"}` for pages, and
`{"command":"detail","component":"DetailView","value":"{\"id\":\"<LTS_ID>\",\"tab\":\"rooms\"}"}`
for the unit list. The detail panel states the stay explicitly
("Verfügbare Zimmer vom 05.09.2026 - 10.09.2026 (5 nights)", "Prices for 2 Person/s"),
so prices are unambiguous totals for the stay.

This is where the cheap farm apartments live — they are largely absent from the
HGV channel.

**4. Val Gardena tourist board** — `https://www.val-gardena.com/en/accommodations-val-gardena-italy/`
Server-rendered, so plain `curl` works:
`?page=N&a=list&enable-stay=1&stay[from]=05.09.2026&stay[until]=10.09.2026&stayD[rooms][0]=1&stayD[persons][0]=2`
Result cards carry `data-lts-id`, joining back to the registry. Prices here are
property-level "from … total", not per room.

Pager links on this site are ROT13-encoded (`cntr` = `page`).

## Joining it together

`Mapping.lts.rid` in the registry is the same key as `data-lts-id` on the board
sites, so registry ↔ board join exactly. The HGV directory carries no LTS id and
is matched on normalised name plus postcode/municipality.

Watch out when bucketing by municipality name: "Rasun **Anterselva**" and
"**Selva** dei Molini" both match a naive `selva`. Filter on postcode first
(39046 Ortisei, 39047 Santa Cristina, 39048 Selva, 39040 shared) and only then
on the municipality string.

## What is not covered

Laion and Ponte Gardena have no open dated search of their own, so they appear
only through the HGV engine (33 of 98 registry entries). IDM's provincial portal
exposes an Algolia proxy at `idm-all-apim-prod-001.azure-api.net` with a public
key embedded in the page, but it serves catalogue content, not dated availability.

## The photos

All 1,137 gallery photos are stored in this repo under `img/`, in two WebP
sizes: `img/s` at 360px for the card thumbnails and `img/l` at 820px for the
lightbox (66 MB in total). They used to be hotlinked from the three tourist
board CDNs, which meant a fresh TLS handshake per host and roughly 1.5 s before
a photo appeared; from this origin the same photo opens in under 0.5 s.

Two things are worth keeping in mind when regenerating:

- The build order is `images.py` → `roomfirst.py` → `make_html2.py`.
  `roomfirst.py` (which calls `imgfetch.py`) rewrites `gallery_c` from CDN tokens
  into local ids, so running `images.py` again undoes that and it has to be re-run.
- Room photos come first in every gallery. The HGV API attaches `pictures` to each
  room record and the tourist board puts its room list after
  `lts-search-rooms__container`, which covers 78 of the 150 properties. Val Gardena
  labels nothing, so the remaining galleries are ordered by how much sky, greenery
  and snow is in the frame, which puts indoor shots first.
- The lightbox arrows are SVG, not the `‹` `›` characters. Those characters
  carry the Unicode `Bidi_Mirrored` property, so in an RTL page the browser
  flips them and they end up pointing inward no matter which way round they are
  written.
