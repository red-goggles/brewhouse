# Brewhouse: handoff for the Claude working in this repo

> Maintained by the Claude working in the sister repo **brewcontrol** until this project starts.
> Last updated: 2026-11-22. **Status: not started.** Only `docs/DESIGN.md` and this file exist.

## What Brewhouse is

A self-hosted recipe, batch and brew-notes app for one all-electric home brewery. It replaces what the owner
uses Brewfather and BeerSmith for (recipes, scaling, water, inventory) and adds what neither does:
- It sends a recipe straight to the brewery controller (**Brew it**).
- It receives every log, event and measurement back.
- It keeps a journal of notes on what went well and what to change next time.

It runs beside **BrewControl**, the brewery controller, in its own Proxmox LXC: CT 127 `brewhouse` at
10.1.32.10 on proxmox-12, Debian 13, VLAN 132. The controller is CT 126 at 10.1.32.11. The two are separate services
in separate repos, but they look like one app: one hostname, one login, a shared navigation bar and design.

The owner's priorities, in their words: communicate recipes to the controller, receive back all the logs,
and "be a place that I can store notes about what went well and what needs to change".

## Read these first

| Doc | What it holds |
|---|---|
| [docs/DESIGN.md](docs/DESIGN.md) | Data model, notes/retrospective/carry-forward design, feature scope by phase, open questions |
| `../brewcontrol/docs/INTEGRATION.md` ([GitHub](https://github.com/red-goggles/brewcontrol/blob/main/docs/INTEGRATION.md)) | **The contract:** routing, shared login, shared UI kit, both API directions, the Brew it flow. The controller side is owned there |
| `../brewcontrol/docs/PLAN.md` | The controller's plan and phases. Brewhouse is its Phase 5 |
| `../brewcontrol/docs/legacy/` | How the old BruControl setup worked, the owner's hardware and scripts. Background only |

Local checkouts: this repo and the controller (`../brewcontrol`) sit side by side in the same folder.
Remotes: `https://github.com/red-goggles/brewhouse` and `https://github.com/red-goggles/brewcontrol`.
Push over SSH with the `github-redgoggles` host alias: `git@github-redgoggles:red-goggles/brewhouse.git`
(the `gh` CLI's account can't see red-goggles repos).

## Hard rules

1. **Never talk to brewery hardware.** Everything physical goes through the controller's API. Brewhouse
   submits plans; the controller validates them, and a person starts them on the controller's screen.
   Brewhouse cannot start, arm-and-run, or override anything by itself.
2. **The controller must never depend on Brewhouse.** Don't design anything the controller has to wait
   for. Brewhouse catches up from the controller's event stream (by sequence number) when it comes back.
3. **Equipment is defined once, in the controller.** Read vessel sizes, losses, heater power and the
   manual vs. automated routing from `GET /api/controller/v1/equipment/profile`. Never keep a second copy.
4. **The contract is versioned.** Commit your OpenAPI to `contracts/openapi.json`. Generate the
   controller client from brewcontrol's spec pinned to a tag. Breaking changes bump `/v1` → `/v2`.
   Changes to the controller side get agreed and made in brewcontrol.
5. **Users and auth belong to the controller.** Caddy `forward_auth` (on the brewcontrol LXC) sets a user
   header; trust it and don't build your own login. Because the two apps are on separate hosts, the
   backend must accept requests only from 10.1.32.11. Otherwise anyone on the VLAN could forge that header.
6. Commit or push only when the owner asks. Keep the owner's personal details out of the repo and out of
   requests to external services.

## Stack and conventions (match brewcontrol)

- Python 3.12, FastAPI, asyncio, SQLite (WAL), `uv`. A systemd unit runs as its own service user;
  data lives in `/var/lib/brewhouse`.
- React, TypeScript, Vite, Tailwind. The UI is served under `/brewhouse/`; the API under `/api/brewhouse/v1/`.
- Shared UI kit (design tokens, top-nav shell, charts, cards, unit formatting) comes from
  `brewcontrol/web/ui-kit` as an npm git dependency pinned to a tag. **It doesn't exist yet**; it'll be
  created in brewcontrol's Phase 4 design pass. Until then, keep components simple and swappable.
- Dark theme by default; mobile-first PWA (used from a phone on brew day).
- **Units:** store SI; display the owner's units (US gallons, °F, lb/oz) with a toggle.
- Calculations use standard published formulas: Tinseth IBU by default (Rager optional), Morey SRM,
  standard ABV. Every formula should live in a pure, unit-tested function.

## The owner and their brewery (context for recipe math and UI)

- Experienced homebrewer. Uses Brewfather most recently, BeerSmith before, and switches between them.
- **Three-vessel, all-electric:**
  - **HLT:** heating element, coil.
  - **Mash tun:** RIMS heater, coil.
  - **Boil kettle:** two elements, steam condenser.
  - Pressure-transducer volume sensing in all three vessels, in gallons.
  - Pumps.
  - Water is RO, filled overnight.
- Hose routing between vessels is **manual today** (electric valves may come). Manual steps become
  "do X, then continue" prompts in the plan.
- **Breaker limit:** only 2 heating elements at once. The controller enforces it; the plan compiler should
  avoid planning violations.
- **Cellar:** two fermenters (FV1, FV2) cooled from a self-contained glycol tank (its own thermostat), one
  glycol pump per fermenter plus one for the chill plate (not wired in yet). Each fermenter has a Tilt hydrometer
  (Black → FV1, Red → FV2) and temperature probes.
- **Old mash script ceiling:** at most 8 steps. Brewhouse should allow any number.

## Recipe source facts (for importers)

**Brewfather API v2** (owner needs Premium):
- **Connection:**
  - Base URL: `https://api.brewfather.app/v2`.
  - Auth: Basic `userid:apikey`; keys have per-scope permissions.
  - Rate limit: **500 calls/hour per key**. On 429, honour `Retry-After`.
  - Paging: `limit` (max 50) with `start_after`; `complete=true` returns full objects.
  - All values are metric.
- **Recipes:** `GET/POST/PATCH/DELETE /recipes`.
- **Batches:**
  - `GET /batches/:id` with `include=recipe.mash,recipe.fermentation`.
  - `PATCH` allows only `status` and the `measured*` fields.
  - `GET /batches/:id/readings`.
- **Inventory:** `GET/POST/PATCH/DELETE /inventory/{fermentables|hops|miscs|yeasts}`.
- **Fields:**
  - Recipe: `boilTime`, `batchSize`, `boilSize`.
  - Mash steps: `stepTemp`, `stepTime`, `rampTime`.
  - Hops: `time`, `use`, `amount`, `timeUnit`.
  - Water: `data.mashWaterAmount`, `data.spargeWaterAmount`, `data.strikeTemp`, `data.hltWaterAmount`.
  - Fermentation steps: `stepTemp`, `stepTime` (days).
  - Some step field names came from a doc summary; verify against live responses.
- **Custom Stream** (optional mirror of fermentation readings):
  - `POST https://log.brewfather.net/stream?id=<id>`, JSON with `name` plus values.
  - Max 1 reading per device per 15 min.

**BeerXML 1.0 / BeerJSON:** both Brewfather and BeerSmith can export at least BeerXML. BeerSmith's native
`.bsmx` is undocumented; support it only if BeerXML loses something the owner needs. BeerSmith has no public API.

## First tasks (Phase 5a; no hardware dependency)

1. Repo skeleton matching brewcontrol: `backend/`, `web/`, `contracts/`, `docs/`, plus CI, lint and tests.
2. Data model and migrations (see DESIGN.md): Recipe → RecipeVersion → Batch, journal, measurements,
   retrospective, tasting notes.
3. Calculation core (OG/FG/ABV/IBU/SRM, volumes), with tests against known Brewfather numbers for the
   owner's own recipes.
4. Importers: BeerXML/BeerJSON first, then the Brewfather API (recipes, batches, readings).
5. Recipe editor with live stats, batches list and detail, notes journal, retrospective with carry-forward.
6. Equipment profile: stub it from a local YAML until the controller's `/equipment/profile` exists, in the
   same shape as the controller's (coordinate before inventing fields).

Phase 5b (Brew it) waits for brewcontrol's Phase 3 session runner. Phase 5c (fermentation) waits for its
glycol controller.

## Controller API availability (updated by the brewcontrol Claude)

**Controller progress (2026-11-22): Phase 0 software done** (protocol, simulator, importer, proxy).
Phase 1 (read-only monitoring) is live on CT 126 port 8000: `status`, `points`, `history`, `config` and a `ws` stream under `/api/controller/v1`. The brewery's
hardware points now have stable ids in `../brewcontrol/config/brewery.yaml`, e.g. `hlt_temp`,
`boil_kettle_volume`, `fermenter_1_temp`, `hlt_pump`,
`hydrometer_fv1`. The equipment profile and events will refer to these, so use the same ids in any
stub. Vessel and equipment objects, which are what Brewhouse will actually read, come in Phase 2.

| Endpoint | Status |
|---|---|
| `/auth/verify` (forward_auth) | not built. The controller has PIN login with a session cookie (`brewcontrol_session`, `/api/controller/v1/auth/*`); forward_auth comes with Caddy |
| `GET /equipment/profile` | not built. Heater equipment exists as `GET /api/controller/v1/heaters` (ids `bk_heater_1`, `bk_heater_2`, `hlt_heater`, `rims_heater`; config includes `watts`, `boil_f`, `max_temp_f`, sensors); the profile will build on it |
| `POST /plans`, `/validate`, `/arm` | not built. The engine they'll feed exists: programs made of steps (routines + person confirmations), one session at a time, `GET /api/controller/v1/programs` (each with a JSON-schema of its params) and `GET /session`. A Brewhouse plan will compile to a program's params. People start sessions at the controller; heating is attended-only, so Brewhouse must never start one |
| `GET /events`, WebSocket | WebSocket `/api/controller/v1/ws` streams live point/device changes (Phase 1); brew-session events and `GET /events?after=` not built |
| `GET /readings` | built as `GET /api/controller/v1/history?series=a,b&from=&to=&max_points=` (Phase 1) |
| `POST /fermenters/{id}/profile` | not built |
| `web/ui-kit` package | not built |

## Open questions for the owner

Copied from DESIGN.md; ask when the project starts:
- After migrating, switch off Brewfather entirely, or keep syncing with it for a while?
- Ingredient and style data source: the owner's own Brewfather/BeerSmith libraries (simplest), open
  BeerJSON data, or BJCP 2021? Check licensing before bundling.
- How much history to import: every past batch with readings, or recipes only?

## Changelog of this file

- 2026-11-22: hosting split into two LXCs (brewhouse CT 127 at 10.1.32.10, brewcontrol CT 126 at 10.1.32.11), Debian 13; added the rule to accept the header only from brewcontrol.
- 2026-11-22: created. Project named, repo created, design split out of brewcontrol into `docs/DESIGN.md`.
- 2026-11-22: both CTs fully updated (Debian 13.8), replicated to proxmox-11 every 15 min (jobs 126-0, 127-0), baseline snapshot `base-2026-11-22` taken. Snapshot before risky changes; roll back with `pct rollback 127 <snap>` on proxmox-12.
- 2026-11-22: brewcontrol Phase 0 software done. Hardware point ids are fixed in `brewcontrol/config/brewery.yaml`; no controller API yet.
- 2026-11-22: brewcontrol Phase 1 running; read-only API live (see the availability table). Firmware framing found: messages end with `;`.
- 2026-11-22: cellar hardware changed. Glycol is now a self-controlled tank with one pump per fermenter and one for the chill plate (not wired yet); old glycol valves and the CIP pump are retired; mash tun volume sensor removed.
- 2026-11-22: brewcontrol Phase 2 step 1 live: PIN login, controllable outputs (RO valve and booster), interlocks, audit log at `/api/controller/v1/events`.
- 2026-11-23: brewcontrol Phase 3 engine built (sessions, programs "heat_hold" and "boil"); brew-day routines come after the owner's walkthrough. The plan contract will be shaped by the engine's step/params model.
- 2026-11-22: brewcontrol heater layer built (simulator-tested; live test pending). Stable heater ids `bk_heater_1`, `bk_heater_2`, `hlt_heater`, `rims_heater`; modes duty / pid / boil; `GET /api/controller/v1/heaters`. Each heater's output and target are logged to history as series `<heater_id>.output` and `<heater_id>.target`. Heating is attended-only until a hardware watchdog relay is fitted.
- 2026-11-23: brewcontrol made the mash tun, HLT, BK and cooling water pumps, `steam_conversion_valve` and `cooling_water_valve` controllable. Draft BrewPlan fields (what Brewhouse must supply) are in brewcontrol `docs/brew-day.md` section 5; not a contract yet.
- 2026-11-23: brewcontrol sessions gained checklists (built, not yet deployed): step kind `stage`, `session.checklist`, `session.measurements` (entered values such as gravities), `POST /session/checklist/{item}`; `/session/confirm` takes `step` and `attended`.
