# Baking Formula App — Build Spec

A personal baking recipe app. Every recipe is stored as a baker's percentage formula. New recipes arrive as Claude-generated JSON files committed to the repo.

---

## 1. What it is

Andreas's own recipe collection — sourdough, yeasted breads and pizza, pastries, cakes and cookies. Not a community app, not a general recipe database. One baker, his own formulas.

**The one job of the main screen:** show the collection as photos, and get to a formula in one tap.

**Everything is grams. Everything is baker's percentage.**

---

## 2. Stack and hosting

| Decision | Choice |
|---|---|
| App | Single self-contained `index.html` (HTML + CSS + JS inline, no build step) |
| Data | One JSON file per recipe in `/recipes/`, plus a manifest |
| Photos | One JPEG per recipe in `/photos/` |
| Hosting | GitHub Pages |
| Adding recipes | Claude generates JSON → Claude Code commits and pushes |

Why a single HTML file: no build step, nothing to install, deploys to Pages instantly, and it will still work in five years. The recipes live outside the app as data, so adding a bake never means touching the app code.

### Repo structure

```
baking-formulas/
├── index.html                     ← the whole app
├── recipes/
│   ├── index.json                 ← manifest of all recipes
│   ├── country-sourdough.json
│   └── brown-butter-choc-chip.json
└── photos/
    ├── country-sourdough.jpg
    └── brown-butter-choc-chip.jpg
```

`recipes/index.json` exists because a static website cannot look inside its own folders. It is the list the app reads on load.

```json
{
  "recipes": [
    {
      "id": "country-sourdough",
      "name": "Country Sourdough",
      "category": "sourdough",
      "photo": "photos/country-sourdough.jpg",
      "tags": ["high hydration", "overnight"],
      "updated": "2026-07-29"
    }
  ]
}
```

---

## 3. Recipe JSON schema

This is the contract between Claude and the app. Claude writes it; the app reads it. Nothing else needs to agree on anything.

```json
{
  "id": "country-sourdough",
  "name": "Country Sourdough",
  "category": "sourdough",
  "tags": ["high hydration", "overnight", "dutch oven"],
  "photo": "photos/country-sourdough.jpg",
  "summary": "Open crumb, deep blistered crust. Overnight cold retard.",

  "yield": { "count": 2, "unit": "loaf", "unitWeight": 900 },

  "flours": [
    { "name": "Bread flour", "percent": 80 },
    { "name": "Whole wheat", "percent": 20 }
  ],

  "ingredients": [
    { "name": "Water",  "percent": 76, "group": "liquid" },
    { "name": "Levain", "percent": 15, "group": "leaven",
      "isLevain": true, "levainHydration": 100 },
    { "name": "Salt",   "percent": 2,  "group": "salt" }
  ],

  "steps": [
    { "text": "Mix flours with all but 50g of the water. Rest 45 minutes.", "timerSeconds": 2700 },
    { "text": "Add levain and salt. Work in the reserved water until smooth." },
    { "text": "Bulk ferment at room temperature, folding every 45 minutes.", "timerSeconds": 16200 },
    { "text": "Shape, basket, cold retard overnight.", "timerSeconds": 43200 }
  ],

  "bake": {
    "tempC": 245,
    "minutes": 45,
    "notes": "Covered 20 min, uncovered 25 min. Steam the first half."
  },

  "notes": "Crust was best yet. Could push water to 78%."
}
```

### Rules Claude must follow when generating a recipe file

1. `flours[].percent` must sum to exactly **100**.
2. Every entry in `ingredients` is a percentage **of total flour weight**.
3. `group` is one of: `liquid`, `leaven`, `salt`, `fat`, `sugar`, `egg`, `dairy`, `inclusion`, `other`. It drives grouping and colour in the display.
4. Any starter, levain, poolish, biga or preferment gets `"isLevain": true` and a `levainHydration` value (100 for a standard 1:1:1 starter). One levain entry maximum.
5. `category` is one of: `sourdough`, `bread`, `pizza`, `pastry`, `cake`, `cookie`.
6. `timerSeconds` only on steps that involve waiting. Omit it on hands-on steps.
7. `id` is lowercase with hyphens, and must match the filename and the photo filename.
8. Filled in, always. No placeholders, no `TBD`.

---

## 4. The percentage math

### Scaling

The app scales from any one of three inputs — Andreas picks which:

| Start from | Formula |
|---|---|
| Target flour weight | `F` is given directly |
| Target total dough weight | `F = target / (sum of all percentages / 100)` |
| Number of items | `F = (count × unitWeight) / (sum of all percentages / 100)` |

Then every ingredient's grams = `F × percent / 100`. Round to the nearest gram; round salt and yeast to 0.1 g.

### Levain slider

The levain percentage is adjustable in the app with a slider. **Default 15%.** Range 5–30%, 1% steps. Changing it recalculates grams live and does not modify the saved recipe file unless explicitly saved.

### Hydration toggle

Two readings, switchable:

- **General hydration** — the water line as written. `water% `
- **True hydration** — includes the flour and water hidden inside the levain.

```
levainFlour = F × (levain% / 100) ÷ (1 + levainHydration/100)
levainWater = levainFlour × levainHydration / 100
trueHydration = (water + levainWater) ÷ (F + levainFlour)
```

Worked example — 76% water, 15% levain at 100% hydration:
general **76.0%**, true **77.7%**.

The toggle is a single control on the formula view, and it remembers its last position.

---

## 5. Screens

### Collection (home)
Photo-first grid, two columns on a phone. Each tile: the photo, the name, and the hydration or key ratio as a small figure. Search field at the top. Tag and category filters as a horizontal scrolling row of chips. Sort by most recently updated by default.

### Formula view
The core screen.

- Photo at the top, full bleed.
- Name, category, summary.
- **The formula ledger** — the ingredient table, one row per ingredient: name, percentage, grams.
- Scaling control: pick flour weight / total dough weight / item count, enter a number, everything recalculates.
- Levain slider.
- Hydration toggle.
- Method, as numbered steps. Steps with a `timerSeconds` show a start button that runs a countdown in the app.
- Bake details: temperature, time, notes.
- Personal notes.
- Print / PDF button.

### Formula maker (new recipe)
The same ledger, empty and editable. Add flour rows and ingredient rows, type percentages, name it, save. Output is the same JSON as an imported recipe — so a formula built by hand and one written by Claude are indistinguishable to the app.

### Import
Paste a JSON file or pick one from the device. The app validates it against the rules in section 3, shows what it found, and reports errors in plain language ("your flours add up to 105%, they need to total 100%"). Imported recipes are held in the browser until committed to the repo.

---

## 6. Design direction

The artifact this is modelled on: **a baker's formula sheet** — the pencilled ledger of ingredient, percentage and weight, on a paper flour bag.

**Palette**

| Name | Hex | Use |
|---|---|---|
| `kraft` | `#DFD3BC` | page background — flour bag paper |
| `dust` | `#EFE7D6` | cards and the ledger surface |
| `graphite` | `#2A2824` | all text — pencil, not black ink |
| `rye` | `#7A6A52` | secondary text, hairline rules |
| `scorch` | `#8E3B18` | the single accent — active states, over-limit warnings. Rare. |

**Type**

- Display: **Bricolage Grotesque** — recipe names and screen titles.
- Body: **Newsreader** — method steps and notes, 17px, line height 1.6.
- Numbers: **IBM Plex Mono**, tabular figures — every percentage and gram weight. Numbers are data; they get their own face.

Scale from a 16px base at 1.25.

**Signature element — the percentage bar in the ledger row.**

Every ingredient row has its percentage drawn as a horizontal fill behind the row, measured against flour at full width. The recipe's shape is legible at a glance: a 76% hydration dough looks visibly different from a 62% one before you read a single number.

The interaction that makes it worth building: **when you scale a recipe, the grams change and the bars do not move.** That is the entire idea of baker's percentage, made visible. The bars are the recipe's identity; the grams are just today's batch.

Everything else on the screen stays plain pencil on paper.

**Non-negotiables**

- Built phone-first, tested at 375px wide.
- Numbers stay readable and aligned — tabular figures, right-aligned columns.
- Timers keep running while scrolling.
- Print stylesheet strips the photo and the chrome, leaving a clean formula sheet on white.
- Real content only — the repo ships with two of Andreas's actual recipes, not samples.

---

## 7. The recipe-intake workflow

1. Bake something. Take one photo on the phone.
2. Tell Claude, in chat: what went in, roughly how it was made, how it turned out.
3. Claude returns the recipe JSON, written to the schema above.
4. Claude Code: saves the JSON to `/recipes/`, resizes and saves the photo to `/photos/`, adds the entry to `index.json`, commits, pushes.
5. Live on every device in about a minute.

**Known friction:** step 4 needs the desktop, or a photo upload through GitHub's mobile web interface. Phone photos also need resizing — a 4 MB original will make the grid crawl. Claude Code should resize every photo to 1200px on the long edge during the commit.

---

## 8. Build order

1. Repo, `index.html` shell, GitHub Pages live and reachable. Confirm the URL works on the phone before anything else.
2. The formula ledger with the percentage bars, rendering one hardcoded recipe.
3. Scaling, levain slider, hydration toggle.
4. Load from `/recipes/` and `index.json`; the collection screen.
5. Search, tags, timers.
6. Formula maker and JSON import.
7. Print stylesheet.
8. Critique pass, then the two real recipes.
