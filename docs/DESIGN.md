# Brewhouse design

Brewhouse is the recipe, batch and notes app that sits beside the BrewControl controller. How the two
connect (repos, routing, login, the API contract, the Brew it flow) is defined in
[brewcontrol/docs/INTEGRATION.md](https://github.com/red-goggles/brewcontrol/blob/main/docs/INTEGRATION.md)
(local: `../brewcontrol/docs/INTEGRATION.md`).

## Data model

```
Recipe ──< RecipeVersion ──< Batch ──< BrewSession (controller run) ──< Events, Readings
                │               │
                │               ├──< JournalEntry (timestamped: auto from controller + manual)
                │               ├──< Measurement (OG, FG, pH, volumes; from sensors or typed in)
                │               ├──  Retrospective (went well / change next time)
                │               └──< TastingNote
                └── carries forward: open "change next time" items
EquipmentProfile  (owned by the controller; Brewhouse reads it)
Ingredient library · Inventory · WaterProfile · Style guidelines
```

- **Recipes are versioned.** Brewing always snapshots a specific version, so editing a recipe never
  rewrites history.
- **Batch lifecycle:** Planned → Brewing → Fermenting → Conditioning → Packaged → Archived.
- **Units:** stored in SI, displayed in your units (gal, °F, lb/oz) with a per-user toggle.

## Notes: what went well, what to change

This is the feature you said you want most, so it gets first-class treatment:

- **Batch journal:** a timeline that mixes automatic entries with your own notes. Automatic entries come
  from the controller: "Mash step 2 reached 152 °F at 10:42", "Sparge ended at 7.1 gal", "manual
  override: RIMS element off". You add notes and photos from your phone at any moment. Notes taken
  during a brew are pinned to that moment on the charts.
- **Retrospective** when the batch closes: *went well*, *change next time*, and the measured numbers vs.
  plan (efficiency, volumes, OG/FG, timings) filled in automatically.
- **Carry-forward:**
  - Open "change next time" items appear when you open the recipe, and again on the Brew it review
    screen for the next batch.
  - One click creates a new recipe version from them.
  - Items are marked resolved when that version is brewed.
- **Tasting notes** at packaging and later (aroma, flavor, mouthfeel, score), so you can compare versions.
- Full-text search across all notes ("what did I say about Citra last time?").

## Feature scope (inspired by Brewfather/BeerSmith, built for this brewery)

**5a: core (no hardware dependency; can be built alongside Phases 1–3)**
- Recipe editor with live OG, FG, ABV, IBU, color as you type.
  - IBU formula choice (Tinseth default, Rager).
  - Color via Morey.
  - Equipment-aware volumes and efficiency.
- Mash, boil and fermentation schedules as editable step lists.
- Recipe versioning and scaling to batch size.
- Batches, journal, measurements, retrospective, tasting notes, carry-forward, search.
- **Import your history:**
  - Brewfather API v2 (recipes, batches, readings, inventory)
  - BeerXML and BeerJSON files (BeerSmith, Brewfather, others)
  - BeerSmith `.bsmx` if needed

**5b: integration**: Brew it, plan compiler, review screen, live journal, batch archive.

**5c: fermentation**: profiles → glycol, gravity/attenuation charts, alerts (stuck fermentation, temp drift).

**5d: depth**
- Water chemistry: source and target profiles, salt additions, mash pH estimate.
- Inventory with automatic deduction when a batch is brewed, plus a shopping list.
- BJCP style guidelines with in-range indicators on the recipe.
- Calculators: ABV, refractometer, hydrometer temp correction, carbonation, strike/infusion, yeast pitch.

**Not planned:** a public recipe library, multi-user sharing, cloud sync. This is a single-brewery app on
your LAN.

## Open questions

1. **Brewfather after migration:** cut over completely, or keep syncing for a while (e.g. its phone app
   during the transition)? Two-way sync is possible via its API but adds work.
2. **Ingredient and style data:** the seed source needs a licensing check before bundling. Options:
   your own Brewfather/BeerSmith libraries via import (simplest, already yours), open BeerJSON sample
   data, and the BJCP 2021 guidelines.
3. **How much history to import:** every past batch with readings, or recipes only?
