---
generated: '2026-09-19'
method: generated
name: Find local producers near a Norwegian place
description: Resolve a place name to coordinates, find producers by category or free text within a radius, then fetch one producer's full details — without inventing distances the data does not support.
api: openapi/rettfrabonden-com-openapi.yml
operations: [geocodePlace, discoverProducers, searchFood, getProducerInfo]
mcp_tools: [lokal_geocode, lokal_discover, lokal_search, lokal_info]
source: >-
  operationIds verified in openapi/rettfrabonden-com-openapi.yml; MCP tool names verified in
  mcp/rettfrabonden-com-mcp-tools.json (live tools/list 2026-09-19); rules from conventions/, errors/,
  rate-limits/ and authentication/.
---

# Find local producers near a Norwegian place

Discover farms, farm shops, REKO rings and markets around a place in Norway and return honest, verifiable
detail. Works identically over REST (base `https://rettfrabonden.com/api/marketplace`) and MCP
(`https://rettfrabonden.com/mcp`); the MCP tool in brackets is the same capability.

## Auth
- None. Every operation here is open. Do not send a key unless you hold a voluntary consumer key
  (`X-API-Key`), which only raises the general rate limit. See `authentication/rettfrabonden-com-authentication.yml`.

## Rate limits
- Search and discover share a static bucket of **150 requests per 15 minutes per IP** (MCP `tools/call`
  counts here). Read `RateLimit-Remaining` / `RateLimit-Reset` on every REST response; back off for the
  window when Remaining is 0. See `rate-limits/rettfrabonden-com-rate-limits.yml`.

## Steps
1. **Geocode the place** — `geocodePlace` (`GET /api/marketplace/geocode?place=Florø`) [`lokal_geocode`].
   Capture `result.lat`, `result.lng` and `result.radiusKm` (a suggested radius). A `404` means the place is
   unknown — ask the user for a nearby town rather than guessing coordinates. Skip this step when the user
   gave a free-text query with a place in it: `searchFood` geocodes automatically.
2. **Discover by structured filters** — `discoverProducers` (`POST /api/marketplace/discover`)
   [`lokal_discover`] with `categories` (e.g. `["honey"]`), optional `tags` (`organic`, `seasonal`), the
   `lat`/`lng` from step 1 and `maxDistanceKm` (default 50; the MCP tool notes 15 in a city, 50–100 in rural
   Norway). A `400` with `error: "Ugyldig søk"` means a field has the wrong type (`categories` must be an
   array).
   **Or search by free text** — `searchFood` (`GET /api/marketplace/search?q=økologisk ost nær Bergen`)
   [`lokal_search`], optionally with `lat`, `lng`, `radius`. Keep accents (`økologisk`, not `okologisk`).
   Leave `start_conversation` unset: searching is read-only and must not message producers unless the user
   asked for contact.
3. **Read the result honestly** — for each `results[].agent`, use `location.distanceLabel` verbatim.
   `location.distanceKm` is present **only** when `geoPrecision` is `address`; for `city`/`kommune`
   centroids it is deliberately omitted, and you must not compute a kilometre figure from the centroid
   coordinates. The top-level `distanceKm` is deprecated. If `needs_location` is true, re-issue with
   `lat`/`lng`; if `relaxed_filters` contains `geo`, tell the user the search was widened to all of Norway.
4. **Fetch one producer's detail** — `getProducerInfo` (`GET /api/marketplace/agents/{agentId}/info`)
   [`lokal_info`] with the `agent.id` from step 2. Present `knowledge.address`, `openingHours`,
   `products[]`, `certifications[]`, `paymentMethods[]`, `deliveryOptions[]`. Prices appear only when the
   producer wrote one into the listing — treat a price as *available*, never *guaranteed*.

## Errors
- Envelope is `{ "success": false, "error": "...", "details": [...] }`; an unknown route returns HTML
  ("Cannot GET …"), so check `Content-Type` before parsing. See `errors/rettfrabonden-com-problem-types.yml`.

## Notes
- The Terms (`/terms`) ask integrators to verify opening hours and prices with the producer before
  travelling or ordering, and forbid bulk scraping to republish the dataset.
- Coverage is Norway only; inputs and messages may be Norwegian or English.
