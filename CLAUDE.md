# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Nature of the repo

Two standalone, self-contained [Grist](https://www.getgrist.com) custom widgets, each a single HTML file (HTML + CSS + inline JS, no build step, no package manager, no tests, no linter). UI text and most identifiers/comments are in French. There is nothing to build or run locally: the files are pasted/hosted as a "Custom widget" (full access) in a Grist document, which injects `grist-plugin-api.js`. Outside Grist, `grist` is undefined and the pages do nothing — verify changes by loading the widget in a Grist doc (or by careful reading).

- `widget-grist-edition-code.html` (~4700 lines) — "Cartographie des entreprises": dashboard over industrial emitters/sites, decarbonisation projects and subsidies (aides). Libraries from CDN: Chart.js, patternomaly, Tom Select, SheetJS (Excel export), html2canvas (copy chart to clipboard), Leaflet map.
- `cartographie-em.html` (~1100 lines) — "Cartographie des positions": kanban-style map of EU member states' stances (Favorable / Ambiguë / Défavorable) on configurable parameters, grouped by camp (pro-réforme / indécis / pro-statu quo), with drag-and-drop.

## Architecture

### widget-grist-edition-code.html
- Data is read once on startup via `grist.docApi.fetchTable` for `Base_projets`, `Base_aides`, `Base_sites`, `Baisse_SNBC_projetee` (see `chargerDonnees()` at the end of the file). Records are column-oriented (`{col: [values]}`), not row objects; helpers like `filtreRecord`, `deepCopyRecord`, `triDecroissantParColonne` operate on that shape.
- Two parallel states: `records_*` (raw) and `filtered_records_*` (after UI filters). `applyFilters()` recomputes the filtered set and re-renders everything (stat blocks, lists, map, charts, top-30 sites table). New views must be wired into `applyFilters()` and `chargerDonnees()`.
- Naming convention: `graphique_*` draws a Chart.js chart; `D_*` builds the data dictionary feeding a chart (e.g. `D_CO2_procede_annee` → `{(année, procédé): MtCO2e/an}`); `render*`/`build*`/`display*` produce DOM; `block*` compute stat-block content. Aides are normalised through `regroupAides` / `allAides` / `projets_Laureat_Aide` before charting.
- Chart legends are custom HTML (`legendGroups`, `getOrCreateLegendContainer`), and the "copy chart" feature composites canvas + legend (`copyChartWithLegendToClipboard`).
- Global year bounds: `anneeMin`, `anneeCourante`, `anneeMax`. Emissions units are Mt/MtCO2e with adaptive formatting (`unitAdaptee`, `formatMtEtiquette`).

### cartographie-em.html
- Bound to a single linked Grist table (one row per member state, ISO code as key). `grist.onRecords` triggers `refresh()`, which reloads config, re-detects the table id (`detectTable`, multiple fallbacks) and re-reads the **whole table** via `fetchTable` (not the onRecords stream) so newly created/hidden columns show up.
- The widget is self-configuring and mutates the Grist schema: parameters are columns (`makeCol`, `repairSchema`, `makeColumnsVisible`, `pushRow`) and settings (camp names, parameters, title) are persisted in a config table (`ensureConfigTable`, `loadConfig`, `writeConfig`) through `grist.docApi.applyUserActions`. `deriveParams` infers parameters from existing columns; `DEFAULT_PARAMS`/`DEFAULT_CAMPS` seed a fresh doc.
- Rendering modes: by camp (`renderByCamp`) or by stance for one parameter (`renderByStance`, chosen via the custom `pfMenu` dropdown); cards are drag-and-drop (`wireDrop`, `wireStanceDrop`) and edit via `openCard`; settings UI is `cfgRender`/`cfgCollect`.

## Notes
- Both call `grist.ready({requiredAccess:'full'})`; changes that touch Grist table/column names must stay consistent with the Grist document's actual schema, which is not in this repo.
- Edit the files directly; keep them self-contained (no splitting into separate JS/CSS files) since they are deployed as single pages.
