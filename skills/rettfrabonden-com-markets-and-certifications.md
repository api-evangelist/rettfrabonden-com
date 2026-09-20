---
generated: '2026-09-19'
method: generated
name: Answer market-day and certification questions
description: Use the umbrella-organisation tools to answer "which farmers sell at Bondens marked?", "is this farm Debio-certified?" and "when is the next market in Oslo?" from the registry instead of guessing.
api: mcp/rettfrabonden-com-mcp.yml
operations: []
mcp_tools: [lokal_list_umbrellas, lokal_get_umbrella_members, lokal_get_producer_affiliations, lokal_bm_next_markets]
source: >-
  MCP-only capability (no REST operation in either published OpenAPI — mcp/rettfrabonden-com-tool-crosswalk.yml
  mcp_only[]). Tool names, enums and required inputs verified in mcp/rettfrabonden-com-mcp-tools.json (live
  tools/list 2026-09-19). Umbrella descriptions from llms/rettfrabonden-com-llms.txt.
---

# Answer market-day and certification questions

Norwegian producers belong to *umbrellas*: market networks (**Bondens marked**, REKO rings), venues
(**Mathallen Oslo**), industry organisations (**Hanen**), certifiers (**Debio** — the official organic
control body, Ø-merket) and cooperatives. These tools read that membership graph. All are read-only,
idempotent and need no credential; call them over MCP at `https://rettfrabonden.com/mcp` after `initialize`
(keep the `Mcp-Session-Id`).

## Steps
1. **Find the umbrella** — `lokal_list_umbrellas`, optionally with `umbrellaType` in
   `market_network | venue | industry_org | certification | cooperative` and a `limit`. Capture the
   umbrella's id (a UUID) and name. Use this when the user asks "what is Bondens marked?" or "which
   certifications matter for local food in Norway?".
2. **List its members** — `lokal_get_umbrella_members` with `umbrellaId` (and `limit`). Returns producer
   names, cities and profile links — e.g. every Debio-certified producer, or every Mathallen tenant.
3. **Check one producer's affiliations** — `lokal_get_producer_affiliations` with `producerId` (the
   `agent.id` from `lokal_search` / `lokal_discover`). Returns the umbrellas the producer belongs to with
   their types — this is how to answer "is Farm Y Debio-certified?" or "which markets does Producer Z
   attend?" from data rather than from the producer's own description text.
4. **Next market days** — `lokal_bm_next_markets` with a `region` substring (e.g. `Oslo`, `Vestfold`) or a
   `lokallag_slug` (e.g. `bondens-marked-agder`), and `days` (default 30, max 90). Data is refreshed daily
   from bondensmarked.no; say so when quoting a date, and suggest confirming close to the day.

## Rules
- Membership is what the registry records, not a legal certificate. The provenance page
  (`/proveniens`) says organic claims are cross-checked against Debio's register (finnoko.debio.no) and
  that AI inferences never count as evidence — quote the affiliation, do not upgrade it to "certified by
  law".
- Only "Verifisert av eier" (verified by owner) is shown as a badge on the site; cross-checking alone adds
  no badge. Do not present a producer as "verified" unless the data says `isVerified: true`.
- Rate limit: MCP `tools/call` shares the 150-per-15-minutes search bucket (`rate-limits/`).
- Errors: `errors/rettfrabonden-com-problem-types.yml`; conventions: `conventions/rettfrabonden-com-conventions.yml`.
