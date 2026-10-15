# Extending AutoSEM

This guide is for anyone who wants to fork AutoSEM for another SEM, other scan presets or another output format. Part 1 is for people. It explains how the file is put together and walks through the common changes. Part 2 is a dense reference to give to an AI coding agent along with `AutoSEM.html`. People who want exact names and formats can use it too.

**About line numbers.** Every line number in this guide refers to `AutoSEM.html` as released in this repository (3573 lines). They will drift as soon as anything is edited. Each reference therefore also gives a stable anchor: the `// SECTION` banner in the script, a CSS banner, or a function, variable or element id. Search for the anchor first and treat the line number as a hint.

## Contents

- [Part 1: For people](#part-1-for-people)
  - [1.1 How the file is organised](#11-how-the-file-is-organised)
  - [1.2 How data flows](#12-how-data-flows)
  - [1.3 Settings a fork is most likely to change](#13-settings-a-fork-is-most-likely-to-change)
  - [1.4 Recipes for common forks](#14-recipes-for-common-forks)
  - [1.5 Testing a change](#15-testing-a-change)
- [Part 2: For AI coding agents](#part-2-for-ai-coding-agents)
  - [2.1 How to work on this file](#21-how-to-work-on-this-file)
  - [2.2 Symbol index](#22-symbol-index)
  - [2.3 DOM contract](#23-dom-contract)
  - [2.4 File-format contracts](#24-file-format-contracts)
  - [2.5 Invariants and gotchas](#25-invariants-and-gotchas)
  - [2.6 Change map](#26-change-map)

---

# Part 1: For people

## 1.1 How the file is organised

AutoSEM is one HTML file. It has no build step and uses no external JavaScript libraries. You open the file in a browser and it runs. The only thing it fetches from the internet is the Inter font from Google Fonts (lines 8-10). On an offline instrument PC the page falls back to the system font and otherwise works the same.

The file has four blocks.

| Block | Lines | Contents |
|---|---|---|
| `<head>` | 1-10 | meta tags (dark colour scheme only), `<title>AutoSEM</title>`, font link |
| `<style>` | 11-501 | All CSS. The colours, fonts and sizes used throughout are CSS variables in `:root` (13-33); components follow under `/* ══ … ══ */` banners. Line 500 is one ~31 KB line holding the base64 PNG for the eLine Plus model card. |
| `<body>` markup | 503-940 | icon sprite, step header, the three step pages, status bar, modals and popups |
| `<script>` | 941-3570 | all behaviour, written as plain global-scope JavaScript (`var` and `function`), split into sections by `// ═══` banners |

Some editors and line-based tools handle line 500 badly. If you read the file in ranges, skip it.

The markup follows the three-step workflow the user sees.

| Part | id | Lines | Contents |
|---|---|---|---|
| Header | `#step-header` | 519-530 | brand, step tabs `#tab-1`..`#tab-3`, Back and Next buttons |
| Step 1, "GDSII Viewer" | `#page-1` | 532-590 | toolbar `#tb-1`, Cells and Layer/Datatype sidebar, `#gds-canvas` |
| Step 2, "Add Scan Frames" | `#page-2` | 592-726 | toolbar `#tb-2`, `#scan-sidebar` (manual placement and import), `#scan-canvas`, `#frames-sidebar` (frame list and groups) |
| Step 3, "Export to SEM" | `#page-3` | 728-843 | toolbar `#tb-3`, `#export-canvas`, the frame table, the SEM model picker `#sem-models-panel` (769-777) and the Raith form `#raith-eline-panel` (778-839) with previews and download buttons |
| Status bar | `#statusbar` | 845-852 | cells, layers, frames, unit, mode, cursor |
| Modals | `#import-modal`, `#download-modal`, `#clear-modal` | 854-889 | message box, save-link fallback, clear-all confirmation |
| Popups | `#array-popup`, `#arr-min-tip`, `#fgo-popup` | 891-939 | scan-array form, minimum-spacing tooltip, group options |

The script is divided into sections, each opened by a banner such as `// STEP NAV`. In order:

| Banner | Line | What it does |
|---|---|---|
| `GDS PARSER` | 943 | `parseGDS` turns the GDSII bytes into the `lib` object (cells with boundaries, paths, texts and references) |
| `GDS WRITER` | 986 | `writeFullGDS` serialises a `lib` object. Only used by `downloadGDS` when there are no original file bytes (the Demo, or after a parse error), to write a `.gds` holding just the frames in one cell, `SCAN_FRAMES`. |
| `DEMO` | 1076 | `buildDemo`, a small built-in layout with two boxes on layer 500/4 |
| `ZOOM SENSITIVITY` | 1095 | mouse-wheel zoom factor and the "Zoom speed" popovers; also the page-wide click handler that closes popups (1131) |
| `STATE` | 1148 | most global state (layout, frames, groups, view, tool modes, undo), the size preset tables (`DT_TO_SIZE`, `SIZE_TO_PROFILE`, …) and colour palettes. Other globals sit next to their code, for example `zoomSensitivity`, the `.pls` constants and `WF_TABLE`, the canvases, ruler and zoom-box state, `previewFrame` and `selectedSEMModel`; 2.2 lists them all. |
| `UNIT HELPERS` | 1195 | `toUm`, `umToMM`, `snapToNm` |
| `SCAN FRAME OVERLAY` | 1206 | `rebuildFrameOverlay` adds the frames to the active cell as layer-500 boxes; `rebuildLayerList` |
| `FLAT RENDER CACHE` | 1264 | flattens the cell hierarchy into a list of shapes for drawing |
| `PLS / JS / TXT GENERATORS` | 1310 | `.pls` column definitions, writefield table, export order, form reader `getP`, and the three text generators |
| `GDS DOWNLOAD` | 1439 | builds the annotated `.gds` by editing the original bytes |
| `RASTER SORT` | 1627 | `rasterSort`, the row-by-row order used for imported frames |
| `IMPORT` | 1657 | the three importers (all of 500/\*, one 500/dt, any layer/datatype) |
| `SNAP` | 1822 | "Snap all" |
| `GO TO FRAME` | 1831 | zoom to a frame from the list |
| `MODAL` | 1854 | the generic message box |
| `PREVIEW / DOWNLOAD` | 1871 | preview tabs, live preview refresh, saving files |
| `STEP NAV` | 1932 | `goStep`, moving between the three pages |
| `CANVAS SETUP` | 1964 | the three canvases, coordinate transforms `w2s`/`s2w`, `fitView` |
| `DRAW` | 2004 | drawing the layout, frames, grid, ruler and zoom box |
| `TOOL SELECTION HELPERS` | 2195 | Pan / Select / Ruler / Zoom box buttons |
| `EDGE SNAP` | 2230 | ruler snapping to layout edges |
| `HIT TEST FRAME CENTRE` | 2259 | hidden-frame test, frame picking, rubber-band selection |
| `APPLY ZOOM BOX` | 2309 | zoom-box tool |
| `PAN / ZOOM` | 2330 | `setupPanZoom`, the mouse handling on the three canvases (press, move, release, wheel, right-click), including placing frames |
| `GLOBAL ESC HANDLER` | 2574 | Enter, Delete/Backspace and Escape |
| `TOOLBAR ZOOM BUTTONS` | 2685 | Fit, zoom in, zoom out |
| `SCAN FRAME CONTROLS` | 2699 | single-frame button, undo, clear all; the `── FGO popup ──` sub-banner (2805) covers the group-options popup |
| `FRAME GROUPS` | 2890 | group helpers, row delete, row selection |
| `FRAME LIST` | 2955 | rebuilds the Placed Frames list |
| `BUILD EXPORT` | 3170 | SEM model selection and the Step 3 frame table |
| `LOAD LIB` | 3227 | installs a parsed layout, picks the active cell |
| `FILE I/O` | 3266 | Open GDS, drag and drop, Demo |
| `RESIZE` | 3298 | keeps canvases sized to their panels |
| `MAPPING TOOLTIP` | 3313 | the 500/dt to size table tooltip |
| `ARRAY POPUP & PLACEMENT` | 3338 | the "+ Scan array" feature |
| `SIDEBAR RESIZE` | 3525 | drag handles on the side panels |

Functions are global, and several are called by name from `onclick`/`oninput` attributes in the markup. Keep the script a plain `<script>`. Changing it to `<script type="module">` would hide those functions from the markup.

## 1.2 How data flows

```
 .gds file bytes
   │  Open GDS (#file-input, FILE I/O 3268) or drop on page (3286)
   ├───────────────────────────────► rawGDSBuffer   (original bytes, kept for export)
   ▼
 parseGDS (GDS PARSER 945) ──► lib = {name, units, dbunit, cells{name: cell}}
   ▼
 loadLib (3229) ─► buildCells (3242) ─► selectCell (3257) ─► activeCell
   ▼
 buildRenderCache (FLAT RENDER CACHE 1266) ─► flattenCell (1272) ─► renderCache
   ▼
 drawCached (DRAW 2008) ─► #gds-canvas, #scan-canvas, #export-canvas
                                  ▲
                                  │ layer-500 overlay boxes (_scanOverlay)
 scanFrames[] ────────────────────┤ rebuildFrameOverlay (1209) after every change
  {id, sizeUm, cx, cy}            │
   ▲  click on #scan-canvas (2345), placeArrayAt (3413),
   │  importAutoFrames (1689), importSingleLayer500 (1757), importCustomFrames (1797)
   │      imports pass through rasterSort (1629)
   ▼
 frameGroups[] ─ createFrameGroup (2920); sortGroupFrames (2917) puts each group smallest-first
   ▼
 getOrderedFramesForExport (1342): groups in creation order, then "Unlinked"
   ├─► generatePLS (1381) ─────────► .pls  (via downloadFile 1923)
   ├─► generateTXT (1422) ─────────► .txt
   └─► buildExport (3183) ─────────► Step 3 table
 scanFrames in array order ─► generateJS (1412) ─► .js
 rawGDSBuffer + scanFrames ─► downloadGDS (1588) ─► annotated .gds
```

**Loading.** When a file is opened or dropped, the raw bytes are stored unchanged in `rawGDSBuffer` and also parsed by `parseGDS` into `lib`. Each cell holds `boundaries`, `paths`, `texts` and `refs`. `refs` are single placements of another cell (GDSII SREF records). Arrayed placements (AREF) are skipped by the parser. `loadLib` installs the library and calls `buildCells`. `buildCells` lists the cells and passes the first one whose name does not start with `$$$` (or the first cell, if all do) to `selectCell`, which sets `activeCell`. Frames already placed stay in `scanFrames` and are added to the new active cell. Everything after that works on the active cell.

**Drawing.** The active cell is flattened into `renderCache` by following references, with translation only (rotation, mirroring, magnification and arrays are not read by the parser). `drawCached` draws that list on all three canvases. In Steps 2 and 3, `drawFrames` (DRAW 2049) then draws each visible frame straight from `scanFrames` on top (dashed outline, `#n` label, centre dot); Step 1 shows frames only as the layer-500 overlay boxes. The three canvases share one view transform (`zoom`, `panX`, `panY`), and `w2s`/`s2w` (1974-1975) convert between database units (the integer grid of the GDS file, often 1 nm) and screen pixels, flipping y because GDS y points up.

**Scanframes.** A scanframe is a plain object in the global array `scanFrames`: `{id, sizeUm, cx, cy}`, plus `bboxYmin`/`bboxYmax` on imported frames. `sizeUm` is the edge of a square frame in µm and should be one of the preset sizes. `cx` and `cy` are the frame centre in the active cell's database units, not in µm. After every change, `rebuildFrameOverlay` copies each frame into `activeCell.boundaries` as a box on layer 500 (datatype from the size) flagged `_scanOverlay`. That is how frames appear in the Step 1 viewer and in the layer list.

**Groups and order.** `frameGroups` holds named groups, each with references to frame objects. Imports and arrays make a group automatically; the user can make more. Imported frames are put in raster order by `rasterSort` (rows from the bottom of the layout upwards, left to right in each row). Every time a group is created, merged into or moved into, its frames are re-sorted by size, smallest first. At export, `getOrderedFramesForExport` lists the groups in the order they were created, then all ungrouped frames under the name "Unlinked". The `.pls`, the `.txt` and the Step 3 table all use this order. The `.js` does not; it uses the raw `scanFrames` order (see 2.5 item 18). The `#n` labels on the canvas and in the Placed Frames list are positions in `scanFrames`. The list also shows groups in the order of their first frame in `scanFrames`. So neither has to match the numbering in the `.pls`, the `.txt` or the Step 3 table.

**Export.** `getP` (1359) reads the Step 3 form. The three generators return strings, which are shown in the preview tabs and saved by `downloadFile`. `downloadGDS` takes a different route: it removes the old layer-500 boxes from the active cell in the original bytes and splices in one new box per frame, so the rest of the file is the original byte for byte. Without original bytes (the Demo, or after a parse error) it writes a new file containing only the frames, through `writeFullGDS`.

**Units and conversions.**

| Quantity | Unit | Where it is set or converted |
|---|---|---|
| layout geometry, `cx`, `cy` | GDS database units (integers in the file; imported centres can be half-units) | parsed as Int32 in `parseGDS` (968) |
| `lib.dbunit` | metres per database unit (second real of the UNITS record) | `parseGDS` (956); 1e-9 assumed when no file is loaded |
| `sizeUm`, array spacings | µm | converted with `sizeUm*1e-6/dbunit` wherever a size is needed in database units |
| screen | pixels | `w2s`, `s2w` (CANVAS SETUP 1974-1975) |
| database units to µm | µm | `toUm` (UNIT HELPERS 1197); used by the generators, the Step 3 table, the frame list and the cursor read-out |
| Step 3 table X, Y and U, V | µm, 3 decimals; mm, 6 decimals | `buildExport` (3216-3218) |
| `.pls` U and V | mm, 6 decimals | `umToMM` (1198), called in `generatePLS` (1402-1403) |
| `.js` x, y | µm, 6 decimals (divided by 1000 inside the generated script) | `generateJS` (1418) |
| `.txt` x, y | µm, 6 decimals | `generateTXT` (1428-1429) |
| annotated `.gds` | integer database units, `Math.round` | `buildScanFrameRecords` (1458-1459) |
| placement grid | 1 nm | `snapToNm` (1199), applied when placing, moving, placing arrays and "Snap all", but not on import |

**Origin.** AutoSEM does not shift the origin. U and V are the frame centre measured from the active cell's own (0, 0). The paper describes the coordinate system as set by the floorplan. In practice the floorplan sets it because the alignment markers are in the same `.gds` coordinates, and the three-point alignment on the tool maps those coordinates onto the sample. `getBBox` (1977) computes the cell extent, but only `fitView` uses it. If your tool needs a different origin or a y-down stage, make the change in the generators, as in 1.4 (e). Leave `toUm` as it is. It is called with x and y alike, so it cannot apply a different offset or sign to each, and the cursor read-out and the frame list also use it (2.5 item 2).

## 1.3 Settings a fork is most likely to change

| Setting | Where (anchor, line) | Current value | What it controls | Notes |
|---|---|---|---|---|
| SEM model list | markup `.sem-models` (774); `selectedSEMModel`, `selectSEMModel`, `showSEMModels` (BUILD EXPORT 3172-3182) | one card, `selectSEMModel('raith-eline-plus')` | which parameter panel appears in Step 3 | There is no model table in the script. `selectedSEMModel` is written but never read, and `selectSEMModel` always shows `#raith-eline-panel`. The `.pls` and `.js` generators are Raith-specific; the `.txt` and `.gds` exports are not. |
| Preset scanframe sizes | `DT_TO_SIZE`, `SIZE_TO_DT` (STATE 1184-1185) | 1, 5, 10, 15, 20, 25, 30, 50, 75, 100 µm on datatypes 0-9 | sizes offered, and the layer-500 datatype for each | Copied by hand into five places in the markup (630-641, 655, 661-670, 682-688, 917-934). Sizes are read with `parseInt`, so they must be whole µm unless you change that. |
| Scan-profile names | `SIZE_TO_PROFILE`, `profileFor` (STATE 1186-1189) | `'20um_bss8nm'` etc. | text in the `.pls` Comment column of each scan and the 4th field of each `.js` entry | Should match the presets stored in your tool's scan manager. Unknown sizes get `size+'um'`. |
| Beam step per size | `SIZE_TO_MIN_SPACING` (STATE 1190) | 0.001-0.040 µm | `.txt` step_um column and the minimum scan-array spacing | Equals the `bss` (beam step size) part of the profile names. Does not follow the Points-U/V inputs. |
| Writefields | `WF_TABLE`, `SIZE_TO_WF_IDX`, `getWF` (PLS / JS / TXT GENERATORS 1319-1338) | 1, 5, 10, 20, 50, 100, 200 µm with labels such as `'6500 x, 20.0 um'` | WRITEFIELD rows in the `.pls` and the Write Field column of the Step 3 table | The chosen writefield is the next one strictly larger than the frame. Labels are written verbatim into the `.pls`. |
| Scanframe GDS layer | literal `500` in many places (2.5 item 19) | 500 | auto-import, the overlay, the annotated `.gds` | No named constant. The layer is reserved in the active cell: export removes all boundaries on it. |
| Mirror layer | literal `501` (IMPORT 1714, 1718, 1774, 1777) | 501 | hidden in-memory copy of imported 500 boxes, for later Custom import | Never written to the exported `.gds`. |
| Datatype for unknown sizes | literal `4` (1217, 1226, 1456, 1613) | 4 (the 20 µm slot) | overlay and `.gds` datatype when a size is missing from `SIZE_TO_DT` | Such frames come back as 20 µm on re-import. |
| `.pls` column set | `PLS_FORMAT`, `PLS_COLS`, `TRAILING` (1312-1314) | 46 data columns | the `.pls` header and the field count of every row | Change all three and the three row templates together. |
| `.pls` header lines | `generatePLS` (1385) | `WAFERLAYOUT=DEFAULT.wlo`, `LotID=`, `WaferID=`, `Slot=`, `MinimizeWin=FALSE` | fixed header keys | LotID, WaferID and Slot take the values of `#p-lot`, `#p-wafer` and `#p-slot` (788-790), empty by default. |
| `.pls` row templates | `generatePLS` (1397, 1405, 1408) | WRITEFIELD, WAIT (macro) and SLOWSCAN rows | everything per frame | Field-by-field layout in 2.4. |
| Wait macro | `generatePLS` (1405) | `%UserRoot%SCRIPT\WAIT.JS`, comment `wait_<s>s` | the settle step before each scan | AutoSEM does not create `WAIT.JS`; it must exist on the tool PC. |
| Pixels per frame | `#p-pts-u`, `#p-pts-v` (792-793); `getP` (1368-1369) | 2500 × 2500 | Points-U/V columns | Changing it does not change the profile names or `.txt` step. |
| Averaging and direction | `#p-avg` (794), `#p-dir` (796), `#p-frame-avg` (801), `#p-lavg` (810); `getP` (1370-1373) | Avg 2, Dir U, Frame 1, LAvg 2 | Avg, Dir, Frame and LAvg columns | |
| Wait between scans | `#p-wait` (818); `getP` (1374) | 5 s | the WAIT row comment | `\|\|5` turns an entered 0 into 5. |
| SSC image naming (`.js` only) | `#p-ssc` (786), `#p-ssc-start` (787); `getP` (1360-1363) | `ARPsi_%04d.SSC`, start 1 | `ScanManager.SSCName` and `SSCIndex` in the `.js` | Site-specific default. `\|\|1` turns 0 into 1. The index is always written with at least four digits (`padStart(4,'0')`, 1361), independent of the width in the prefix. |
| Output file names | `#p-plsfile`..`#p-gdsfile` (820-823); `getP` (1375-1378) | `scan_frames.pls`, `scan_frames.txt`, `raith_scan.js`, `scan_frames.gds` | suggested save names | Change the markup value and the `getP` fallback together. |
| Unit conversion and precision | `toUm`, `umToMM`, `snapToNm` (UNIT HELPERS 1197-1203) | µm; mm with 6 decimals; 1 nm grid | all exported coordinates | |
| Layers hidden on load | `loadLib` (3232) | `'1/10'` | layer 1/10 is hidden every time a file loads | A local layout convention. |
| Custom-import defaults | `#import-layer` (676), `#import-dt` (678), `#import-size-sel` (682-688) | 68 / 0, 20 µm | Step 2 "Custom Layer/DT" form | 68 matches the demo layout. |
| Scan-array defaults | `arrayRows`…`arrayColSpacing` (STATE 1165); `#arr-*` (896-908) | 2 × 2, 0.008 µm pitch | "+ Scan array" | Spacing is centre-to-centre; default equals `SIZE_TO_MIN_SPACING[20]`. The popup refills its inputs from the state variables each time it opens (3469-3472). "Reset to defaults" hard-codes 2 × 2 (3513-3514, 3521) and resets spacing to `getArrayMinSpacing()`. |
| UI colours, type, radii | `:root` variables (13-33) | dark palette, Inter | the whole interface | Some CSS colours are literals; see 1.4 (f). |
| Canvas colours | `LPAL`, `SCAN_COLORS` (STATE 1181-1182); literals in DRAW | 11 and 10 hex colours | layer and frame colours on the canvases | Not linked to the CSS variables. `LPAL`, `SCAN_COLORS` and some DRAW literals must stay `#rrggbb` (2.5 item 47). |
| Wheel zoom speed | `zoomSensitivity`, `_sensToFactor` (ZOOM SENSITIVITY 1099-1100) | 1.15 per wheel step | scroll-wheel zoom | Toolbar +/- use a fixed 1.5 (2689-2696). The Step 3 zoom-speed panel does not work (2.5 item 48). |

## 1.4 Recipes for common forks

Each recipe lists the steps in order. Section 2.6 repeats them in a terse form with exact strings for an agent.

### (a) Change the existing SEM preset, or add another SEM model

**Same tool, different site.** This is probably the most common fork: another eLine Plus (or a close relative) with its own scan-manager presets and conventions.

1. Edit `SIZE_TO_PROFILE` (STATE 1186) so every name matches a preset that exists in your scan manager.
2. Edit `WF_TABLE` (1319) so the `label` strings match your tool's writefield names and magnifications, and the `size` strings are what your tool expects in Size-U/V of a WRITEFIELD row.
3. Change the form defaults in the markup (786-823) and the matching fallbacks in `getP` (1360-1378): SSC prefix, Points-U/V, Avg, Dir, Frame Avg, LAvg, Wait, file names.
4. If your wait macro lives elsewhere, edit the path in the WAIT row (1405).
5. Update the help and tooltip text that describes these (756, 785, 787, 800, 809, 818).

**A second SEM model next to the Raith one.** AutoSEM has the picker but no real model handling behind it, so a bit of plumbing is needed.

1. Markup: add a card inside `.sem-models` after line 774, for example `onclick="selectSEMModel('my-sem')"`. The card image comes from `.sem-model-img`, which line 500 fills with the eLine photo for every card, so give the new image span an extra class (say `sem-model-img--my-sem`) and add a CSS rule with its own `background-image` after line 500, just before `</style>`. Line 500 is the last rule in the stylesheet and has the same specificity, so a one-class rule placed earlier, for example next to `.sem-model-img` at 406, loses to it. `.sem-model-img.sem-model-img--my-sem` works anywhere.
2. Markup: add a parameter panel next to `#raith-eline-panel` (778-839). The simplest route is to copy the Raith panel and prefix every id inside it (for example `q-pts-u`, `q-tab-pls`, `q-preview-pls`, `q-dl-pls-btn`), because the existing ids are looked up by name. Add CSS like lines 410-411 so the new panel is hidden until it has class `visible`. Inputs keep class `param-input` and so refresh the preview live, but the three copied `<select>`s do not. Add their ids (for example `#q-dir,#q-frame-avg,#q-lavg`) to the selector at 1886 (2.5 item 41).
3. Script, BUILD EXPORT: make `selectSEMModel` and `showSEMModels` show and hide panels by model id instead of the hard-coded `'raith-eline-panel'` (3176, 3181). A small lookup object is enough:

   ```js
   var SEM_MODELS={
     'raith-eline-plus':{panel:'raith-eline-panel'},
     'my-sem':{panel:'my-sem-panel'}
   };
   function selectSEMModel(model){
     selectedSEMModel=model;
     document.getElementById('sem-models-panel').style.display='none';
     Object.keys(SEM_MODELS).forEach(function(k){
       document.getElementById(SEM_MODELS[k].panel).classList.toggle('visible',k===model);
     });
     refreshPreviews();
     buildExport();   // rebuild the Step 3 table for this model (step 5)
   }
   function showSEMModels(){
     selectedSEMModel=null;
     document.getElementById('sem-models-panel').style.display='';
     Object.keys(SEM_MODELS).forEach(function(k){
       document.getElementById(SEM_MODELS[k].panel).classList.remove('visible');
     });
   }
   ```

   Each entry in `SEM_MODELS` can also hold the model's id prefix, form reader and writefield lookup, for example `{panel:'my-sem-panel', prefix:'q-', getP:getP_MySEM, getWF:getWF_MySEM}`, so that `switchPreview`, `downloadFile`, `downloadGDS` and `buildExport` can find them from `selectedSEMModel`.

4. Write the new model's form reader and generators (copy `getP` and the generators as a starting point), and make `refreshPreviews`, `downloadFile` and `switchPreview` (or prefixed copies of them) work for the new panel. `refreshPreviews` is called by `buildExport` before any model is chosen (`goStep(3)` resets to the picker, 1959), so it has to cope with `selectedSEMModel === null`. Add the new panel's download button ids to the list in `buildExport` (3190). The copied buttons have `disabled` in the markup and nothing else enables them.
5. The Step 3 frame table shows writefield rows and a Write Field column from `getWF` (3197), and `generatePLS` calls `getWF` directly (1393). If the new SEM has its own writefield table, give it its own table and lookup, have its generator use that lookup, and make `buildExport` use the selected model's lookup. `buildExport` runs when Step 3 opens, before a model is chosen, so call it again from `selectSEMModel`, as in the sketch above; otherwise the table keeps the Raith writefields. If the new SEM has no writefields, branch in `buildExport` around the writefield rows (3207-3214) and the Write Field cell of each frame row (3218), drop the column from the table header (765), and change `colspan="7"` in the group and writefield rows (3203, 3211).
6. These parts are not Raith-specific and can be reused as they are: `getOrderedFramesForExport`, `toUm`, `umToMM`, `saveFileAs` and the GDS export. `downloadGDS` reads its file name from `#p-gdsfile` in the Raith panel through `getP()` (1590, 1623), so point that call at the selected model's reader, or the new panel's `.gds file` box has no effect.

### (b) Change the scanframe sizes or the writefield table

1. STATE: edit `DT_TO_SIZE` (1184) and `SIZE_TO_DT` (1185). `SIZE_TO_DT` must be the exact inverse of `DT_TO_SIZE`.
2. STATE: give every size an entry in `SIZE_TO_PROFILE` (1186) and `SIZE_TO_MIN_SPACING` (1190).
3. GENERATORS: give every size an entry in `SIZE_TO_WF_IDX` (1329), pointing at the right `WF_TABLE` row. If your writefields differ, edit `WF_TABLE` (1319). If a writefield equal to the frame size is acceptable on your tool, change the `>` in `getWF` (1334) and the map. `SIZE_TO_WF_IDX` holds row indices into `WF_TABLE`, so inserting a row (say 150 µm between 100 and 200) shifts the index of every later row; recheck every entry.
4. Markup, five copies to update by hand:
   - `#frame-size-sel` options (631-640), including which one is `selected` and the "(Layer 500/k)" text, where k must equal `SIZE_TO_DT[size]`;
   - the mapping tooltip table inside line 655;
   - the `.import-dt-btn` buttons (661-670): the `importSingleLayer500(k)` argument and the `dt-size` text;
   - `#import-size-sel` options (683-687);
   - the `#arr-min-tip` table (922-931), values in nm.
5. If you change the default size, also change the array spacing defaults: `arrayRowSpacing`/`arrayColSpacing` (1165) and the `value` of `#arr-row-sp`/`#arr-col-sp` (904, 908) should equal `SIZE_TO_MIN_SPACING` of the new default. `getArrayMinSpacing` falls back to 20 (3341).
6. Check the demo: its two boxes `R(500,4,…,20,20)` (1088) assume datatype 4 is 20 µm.
7. If datatype 4 no longer makes sense as the fallback for unknown sizes, change the literal `4` at 1217, 1226, 1456 and 1613.
8. More than ten sizes: the import loops follow `DT_TO_SIZE.length`, but the markup lists and the message `'No boundaries on layers 500/0–500/9.'` (1709) do not.
9. Non-integer sizes (say 2.5 µm): change `parseInt` to `parseFloat` where sizes are read (1801, 2346, 2470, 3341, 3414, 3439). Object keys such as `{2.5: …}` work as they are.

### (c) Use a different layer or datatype convention for auto-import

The scanframe layer 500 is written as a literal throughout. We'd suggest first adding constants next to `DT_TO_SIZE` in STATE, for example `var SCAN_LAYER=500, MIRROR_LAYER=501, FALLBACK_DT=4;`, and then replacing the literals:

1. Numeric `500` in the script: `buildDemo` (1088), `rebuildFrameOverlay` (1221, 1226), `rebuildLayerList` (1250), `buildScanFrameRecords` (`b16(500)`, 1461), `stripLayer500FromCell` (1516), `stripLayer500DTFromCell` (1567), `downloadGDS` stub (1617), `importAutoFrames` (1699, 1714, 1721), `importSingleLayer500` (1764, 1773, 1780). Replace only these. A file-wide search for `500` also finds `2500` (792-793, 1368-1369), `'6500 x, 20.0 um'` (1323), the font link (10), CSS font weights and `z-index: 500` (486), the sidebar maximum width (3566-3568), and names such as `sz500` and `foundLayer500`.
2. `501`: numeric at 1714 and 1774, and inside the hidden-layer keys `'501/'+hdt` (1718) and `'501/'+dt` (1777), which become `MIRROR_LAYER+'/'+hdt` and `MIRROR_LAYER+'/'+dt`. Pick a layer your designs never use.
3. Text that mentions 500: messages and group names at 1709, 1731, 1734, 1769, 1789, 1792, and the markup at 631-640, 651, 655, 656, 658, 661-670. Comments at 1225, 1461, 1490, 1594, 1710, 1723 and 1771 also mention 500 and can be updated; they do not affect behaviour. When retesting with 1.5, read your new layer wherever steps 2, 3, 6, 7 and 8 say 500.
4. Function names contain "500" (`importSingleLayer500`, `stripLayer500FromCell`, `stripLayer500DTFromCell`). They can stay as they are; if you rename `importSingleLayer500`, update the ten `onclick` attributes (661-670).
5. GDS layer numbers are read as signed 16-bit values (964), so layers above 32767 never match even though `#import-layer` allows up to 65535.

If your convention encodes the size differently, for example one layer per size instead of one datatype per size, the places to change are `importAutoFrames` (the loop at 1694-1708 and the mirror and removal code at 1711-1721), `importSingleLayer500`, the layer/datatype chosen in `rebuildFrameOverlay`, `buildScanFrameRecords` and the stub branch of `downloadGDS` (1613-1617), the size label in `rebuildLayerList` (1250), and the strip functions.

Keep in mind that the scan layer is reserved in the active cell: `downloadGDS` removes every boundary on it before writing the frames (1595).

### (d) Change or add a column in the `.pls`, or a line in the `.js`

**Changing a value that already has a column.** Edit the row template in `generatePLS`: WRITEFIELD at 1397, WAIT at 1405, SLOWSCAN at 1408. The column order is in 2.4. For example, a value for the `Stepsize` column (field 30) on scan rows means replacing the 24 empty fields in `TRAILING` (fields 21-44) with an explicit run in which field 30 holds your value. On the SLOWSCAN row, replace `+p.avg+TRAILING+','+p.frameAvg` with `+p.avg+',,,,,,,,,,'+val+',,,,,,,,,,,,,,'+','+p.frameAvg`. Ten commas lead to field 30, fourteen more give the empty fields 31-44, and the existing `','` comes before Frame (field 45). Every row must keep 46 fields, so count them after editing.

**Adding a user setting.** Add a `.param-row` with an input of class `param-input` inside `#raith-eline-panel` (the class makes the preview refresh live, 1886; a `<select>` needs its id added to that selector string). Read it in `getP` (1359) with a fallback. If 0 is a valid value, do not copy the `||default` pattern, which turns 0 into the default (2.5 item 50); test with `isNaN` instead. Then write it in the template.

**Adding a column the format does not have.** Append `,Name,0` to `PLS_FORMAT` (1312), add a `Name=W:…` line to `PLS_COLS` (1313), and add one field to all three row templates. We have not tested whether eLine Plus accepts columns beyond those its own export writes, so check on the tool.

Comma counting is the error-prone part. A fork that edits the `.pls` often may want to build rows from names instead:

```js
// Column names in .pls order (46), derived from PLS_FORMAT
var PLS_FIELDS=(function(){
  var parts=PLS_FORMAT.replace('FORMAT=','').split(',');
  var names=['ID','X','Y','Z','R','T','U','V','W','Attribute','Template','Comment'];
  for(var i=1;i<parts.length;i+=2)names.push(parts[i]);
  return names;
})();
function plsRow(vals){  // vals: {ID:1, U:'0.024000', Type:'SLOWSCAN', ...}; missing fields are empty
  return PLS_FIELDS.map(function(n){return vals[n]!=null?vals[n]:'';}).join(',');
}
```

To keep the output byte-identical with the released file, pass `'0.000000'` for X, Y, Z, R, T and W (and for U, V on WRITEFIELD rows), and keep the double quotes around the WRITEFIELD label.

**The `.js`.** The whole script is one string at line 1420; per-frame entries are built at 1418. Fix the comma bug first (2.5 item 49): each entry ends in `   // #k` and the entries are joined with `',\n'`, so with two or more frames the comma sits inside the comment and the generated script fails. One fix:

```js
lines.push('  ['+toUm(f.cx).toFixed(6)+', '+toUm(f.cy).toFixed(6)+', '+f.sizeUm+', "'+profileFor(f.sizeUm)+'"]'
  +(i<scanFrames.length-1?',':'')+'   // #'+(i+1));
// ...and join with '\n' instead of ',\n' at 1420
```

Inside the template, `\\\\` in the source becomes `\\` in the generated file, and `"` is used for the Raith script's own strings. Values pasted into it (such as the SSC prefix) are not escaped. If the `.js` should describe the same run as the `.pls`, switch it to `getOrderedFramesForExport()` and add the writefield and wait steps; at present it does neither.

### (e) Export for a different tool's position-list format

Keep:
- everything up to the export order: parser, viewer, frame placement, groups, the importers, `rasterSort`, `getOrderedFramesForExport`;
- the unit helpers `toUm` and `snapToNm`;
- the annotated `.gds` export and the `.txt` lookup table, which do not depend on the tool.

Replace or adapt:
- `generatePLS` and the `PLS_FORMAT`/`PLS_COLS`/`TRAILING` constants, with a generator for your format;
- `generateJS`, or drop it;
- `WF_TABLE`/`SIZE_TO_WF_IDX`/`getWF` if the tool has no writefields, together with every use in `buildExport`: the `getWF` call (3197), the writefield rows (3207-3214), the Write Field cell of each frame row (3218), the table header (765) and `colspan="7"` in the group and writefield rows (3203, 3211);
- `SIZE_TO_PROFILE`/`profileFor` if the tool names presets differently;
- `getP` and the parameter panel (778-839), the panel heading (783) and the Step 3 hint (756). The script looks up the panel's elements by id with no null check, so if you remove the panel or change its ids, update `showSEMModels` (3181) and the button list in `buildExport` (3190), which run each time Step 3 opens, as well as `selectSEMModel` (3176), `switchPreview` (1875-1877), `refreshPreviews` (1882-1884) and the `getP()` call in `downloadGDS` (1590);
- the file names and save-dialog types in `downloadFile` (1923-1928).

To add a new output file type `xyz`:
1. Markup: a tab `<div class="preview-tab" id="tab-xyz" onclick="switchPreview('xyz')">`, a pane `<div class="preview-code" id="preview-xyz">`, a download button and a file-name `.param-row`. Do not use `1`, `2` or `3` as the type name, because the step tabs already use `tab-1`..`tab-3`.
2. Script: add `'xyz'` to the array in `switchPreview` (1875), a line in `refreshPreviews` (1881-1884), a branch in `downloadFile` before the final `else` (1927), the button id in the list in `buildExport` (3190), and the file name in `getP`.

Coordinate conventions differ between tools. If yours wants µm instead of mm, a different origin, or y pointing down, convert in your generator from `toUm(f.cx)`, `toUm(f.cy)`. The Step 3 table (3216-3218) shows U/V in mm and should probably follow.

### (f) Restyle the UI

1. Edit the variables in `:root` (13-33). Most of the interface follows them.
2. Some CSS colours are literals and need editing by hand: lines 42, 80, 82, 84, 85, 169 (the `<select>` chevron is a data URI with `stroke='%236e7681'`), 200, 208, 209, 285, 308, 310, 311, 330, 331, 346, 369, 386, 387, 403, 406 (white tile behind the product photo), 444, 460, 463 and 482.
3. Canvas colours are set in the script and do not read the CSS: `LPAL` and `SCAN_COLORS` (STATE 1181-1182), the canvas background `'#0a0c10'` (2150, 2163, 2185), and literals in DRAW (2023, 2061-2062, 2085-2091, 2110-2124, 2138, 2171-2173), the array preview (3445) and the frame-list dot (2970). Fills are made by appending a two-digit alpha to the colour string, so every palette entry must stay a 7-character `#rrggbb`.
4. Fonts: the Google Fonts link (8-10), `--font` and `--mono` (25-26), and the canvas font strings `'Inter, system-ui, sans-serif'` (2023, 2069, 2119, 2142). Remove the link for a fully offline build.
5. The page is dark only (`<meta name="color-scheme">` at 6, `color-scheme` at 14). A light theme needs a second set of variables and light canvas colours.
6. Names: `<title>` (7), `.brand` (520), the step labels (521, 523, 525). Change the text only; `goStep` finds tabs and pages by their order.
7. Keep every class the script toggles, with a rule if it should look different (list in 2.3). `.inactive` on `#place-btn` and `.open` on `#fgo-toggle-btn` are toggled but have no rule.
8. The CSS is split into banner-commented sections: Buttons (54-85), Step header (87), Toolbar (118), Popovers (134), Inputs (155), Panels & layout (175), Step 2 (242), Step 3 (375), Modals (441), Tips (469) and Scan array popup (485).

## 1.5 Testing a change

There is no automated test suite. This is the manual check we'd suggest after any change, using a Chromium-based browser (Chrome or Edge) so that the native save dialog is used. Safari and Firefox fall back to the download-link modal.

Section 2.6 (g) has a short shell script that runs the generators without a browser, using Node.js or, on a Mac, `osascript`. It counts the fields in every `.pls` row and tries the `.js`. It is worth running before the manual steps.

1. Open `AutoSEM.html` from disk with the developer console open. Any error in the console counts as a failure, apart from the known issues listed below.
2. In Step 1, click Demo. The cell list should show `TOP`; the layer list should show 1/0, 9/0, 10/0, 68/0 and `500/4 (20 × 20 µm)`; the status bar unit should read `1.00 nm`.
3. In Step 2, click "Import all 500/\* layers": two frames in a group named `500/* Import`. Place a single frame. Place a 2 × 2 array with a spacing larger than the frame (say 30 µm for a 20 µm frame). Run a Custom import of layer 68/0 at 10 µm ("Custom Layer/DT" tab, "Import box centres"): six frames. Try "Undo last", the Delete key on a selection, "Snap all", "Add group" and "Merge groups" (in the "Group options" menu).
4. In Step 3, pick the model card. Check the table: group header rows, a "Set write field" row before the first frame and wherever the writefield changes, and coordinates that look right. Change Points-U and check that the `.pls` preview updates.
5. Download the `.pls`, `.txt` and `.gds` (and the `.js`, which in the released file only works with a single frame).
6. Diff against a reference. Before editing, export a reference from the unmodified file and keep those files in the fork, for example under `tests/reference/`. Build the reference only from steps that give the same positions every time (Demo, "Import all 500/\* layers", the Custom import of 68/0). A frame placed by clicking lands wherever the click was and will not match between runs. The `.pls` and `.txt` contain no timestamp, so an export with unchanged settings should match exactly (`diff -u reference.pls new.pls`). The `.js` has a timestamp on line 2 (`diff <(sed 2d ref.js) <(sed 2d new.js)`).
7. Check the `.gds` in KLayout. Open the annotated file: there should be one box per frame on 500/dt, centred where the frames were placed, and every other layer should be unchanged. KLayout's XOR tool (Tools menu), run against the original, should only show differences on the scan layer. For the demo, which has no source file, expect a small library `SCANFRAMES` with one cell `SCAN_FRAMES` holding only the boxes.
8. Round trip. Load the exported `.gds`, select the cell that was active when you exported (the frames are written into that cell only, and AutoSEM selects the first cell not starting with `$$$` on load), and run "Import all 500/\* layers". You should get the same number of frames with the same sizes and centres, to within 1 database unit. Groups and order will differ, since the `.gds` carries no group information.
9. On the tool, load the `.pls` in the eLine Plus position list, check that it opens with no errors and that a few positions land where expected after alignment. Run a short list before a long one.
10. Repeat with one of your own layouts that has hierarchy, since the demo is a single flat cell. The parser skips AREFs and ignores SREF rotation, mirroring and magnification, so geometry placed that way is missing or misplaced in the viewer and in imports. The released file behaves the same, so this is not a regression. The exported `.gds` still contains it, since the original bytes are copied.

Things that should still hold after any change (details in 2.5):
- `cx`/`cy` stay in database units and `sizeUm` in µm; conversions go through `lib.dbunit`.
- No origin offset is added unless you meant to add one, and then in every generator and the Step 3 table.
- The `.pls` scan IDs 1..N, the `.txt` scan numbers and the table "No." column agree, and all follow `getOrderedFramesForExport`. The `#n` labels on the canvas and in the Step 2 frame list, and the order of the `.js`, follow `scanFrames` instead, so they can differ from the scan IDs once frames are grouped.
- A WRITEFIELD row comes before the first scan and before any scan whose writefield differs from the previous one; a WAIT row precedes every scan at the same U, V.
- Every `.pls` data row has the same number of fields as `PLS_FORMAT` declares (46 at present).
- The preset tables and their five copies in the markup agree.

Known issues in the released file that are not regressions (see 2.5 for more):
- The Step 3 "Zoom speed" panel (`#zsw-3`, 744-753) calls a function that does not exist (`updateZoomSens`) and throws a ReferenceError when used.
- The generated `.js` fails with two or more frames.
- Entering 0 for Wait or SSC start index is replaced by the default (5 s and 1).
- Hiding a group (or the Unlinked set) hides the frame outlines and labels on the Step 2 and Step 3 canvases. The layer-500 boxes stay visible unless that layer is hidden, and the frames still appear in the Step 3 table and in every export.

---

# Part 2: For AI coding agents

Reference for making changes to `AutoSEM.html`. Line numbers refer to the released file and will drift; locate code by the anchor (banner, function, variable or id) and use the line only as a starting point.

## 2.1 How to work on this file

- One file, `AutoSEM.html`: `<head>` 1-10, `<style>` 11-501, markup 503-940, one `<script>` 941-3570. No build, no bundler, no npm, no framework. The only external resource is the Google Fonts link (8-10). Do not add dependencies unless asked.
- Do not split the file into separate CSS/JS files unless the user asks. Users open it straight from disk on instrument PCs.
- Keep the script a classic `<script>` in global scope. Markup calls these functions by name from `onclick`/`oninput`: `goStep`, `toggleZoomSens`, `updateZoomSensSlider`, `updateZoomSensNum`, `setImportMode`, `toggleImportSinglePanel`, `importSingleLayer500`, `selectSEMModel`, `showSEMModels`, `switchPreview`, `downloadFile`, `downloadGDS`, `closeModal`, and the global `currentStep`. Wrapping the script in a module or IIFE breaks them. Markup also calls `updateZoomSens` (746, 750), which is not defined anywhere (see 2.5 item 48).
- Syntax is ES5: `var`, `function`, string concatenation, no arrow functions, template literals, `let`/`const` or classes. Match it so the file stays consistent. The code does use later built-ins (`Set`, `Object.values`, `Object.assign`, `padStart`, `Array.prototype.fill`, Promises), so it already needs a current browser.
- Keep the section banners and their titles exactly. Each banner is a `// ═══…` rule, a `// TITLE` line (sometimes followed by short comment lines) and a closing `// ═══…` rule. The line numbers in 2.2 point at the title line. These are the anchors this guide and others use. Add new code inside the relevant section, or add a new banner in the same form.
- Find code by anchor (`grep -n "function getWF"`, `grep -n "// BUILD EXPORT"`), never by line number alone.
- Line 500 is a ~31 KB single line (base64 PNG in CSS). Exclude it when reading ranges (e.g. `sed -n '392,499p'` then `sed -n '501,600p'`).
- Preserve every CSS class the script toggles (list in 2.3) and every id the script looks up.
- Preset sizes and the scan layer are duplicated in markup and script with no single source. When changing one copy, change all of them; 2.6 (b) and (c) list them.
- After a change, run the checks in 1.5 and the headless check in 2.6 (g). Diff `.pls`/`.txt` output against a reference from the unmodified file when the change should not affect them.

## 2.2 Symbol index

### Functions

Columns: name, line, purpose, then state read (R) and written (W). "DOM" means it reads or writes elements directly.

#### GDS PARSER (943)

| Function | Line | Purpose | State |
|---|---|---|---|
| `parseGDS(buffer)` | 945 | GDSII ArrayBuffer → `lib` object (record dispatch at 954-978) | local only; returns new lib |
| `readStr(v,s,l)` | 982 | ASCII payload, stops at NUL, trims | pure |
| `readF64(v,o)` | 983 | 8-byte GDSII real (sign bit, excess-64 base-16 exponent, 56-bit mantissa) → Number | pure |

#### GDS WRITER (986)

| Function | Line | Purpose | State |
|---|---|---|---|
| `writeFullGDS(gdsLib,scanBounds,activeCellName)` | 988 | serialise a lib to Uint8Array: HEADER v7, BGNLIB, LIBNAME, UNITS, per cell boundaries (≥3 pts, `_scanOverlay` skipped; `scanBounds` appended in `activeCellName`), paths, texts, SREFs, ENDLIB. Only caller: `downloadGDS` stub (1621) | args only |
| nested `rec`, `i16`, `i32`, `str2`, `f64ibm`, `xyRec` | 991-1013 | record framing (odd length padded), int16, int32 (`Math.round`), even-padded string, IBM real, XY payload | closure `records` |

#### DEMO (1076)

| Function | Line | Purpose | State |
|---|---|---|---|
| `buildDemo()` | 1078 | lib `'demo'`, units `[0.001,1e-9]`, one cell `TOP`; nested `R(l,dt,x,y,w,h)` (1081) makes a closed rectangle from µm × `S=1000` | returns new lib |

#### ZOOM SENSITIVITY (1095)

| Function | Line | Purpose | State |
|---|---|---|---|
| `_sensToFactor(s)` | 1100 | `1+s*0.15` | pure |
| `_sliderToSens(p)` | 1101 | `10^((p-50)/50)`, slider 0-100 → 0.1-10 (50 → 1.0) | pure |
| `_sensToSlider(s)` | 1102 | inverse, clamps s to [0.1,10] | pure |
| `_applySens(s)` | 1103 | apply sensitivity, sync all zoom widgets, label thresholds at 1109 | W `zoomSensitivity`; DOM `.zoom-sens-*` |
| `updateZoomSensSlider(pos)` | 1112 | `oninput` of sliders in `#zsw-1`/`#zsw-2` | via `_applySens` |
| `updateZoomSensNum(val)` | 1115 | `oninput` of number boxes; valid 0.1-10, else error state | via `_applySens`; DOM |
| `toggleZoomSens(e,wrapId)` | 1123 | open/close the panel in `#zsw-1..3`, close the others; `e.stopPropagation()` | DOM |
| (document `click` listener) | 1131 | close `.zoom-sens-panel`s, `#array-popup` (via `closeArrayPopup`), `#import-single-panel`, `#fgo-popup` unless click is inside | DOM |

#### STATE (1148), functions defined among the state declarations

| Function | Line | Purpose | State |
|---|---|---|---|
| `profileFor(sz)` | 1189 | `SIZE_TO_PROFILE[sz]` or `sz+'um'` | R `SIZE_TO_PROFILE` |
| `getLC(l,dt)` | 1192 | colour for `'l/dt'`, assigned round-robin from `LPAL` on first use | R/W `layerColors`, `layerColorIdx` |

#### UNIT HELPERS (1195)

| Function | Line | Purpose | State |
|---|---|---|---|
| `toUm(v)` | 1197 | database units → µm: `v*(lib?lib.dbunit:1e-9)*1e6` | R `lib` |
| `umToMM(um)` | 1198 | `(um/1000).toFixed(6)` (string) | pure |
| `snapToNm(v)` | 1199 | `Math.round(v/nmPerUnit)*nmPerUnit`, `nmPerUnit=1e-9/dbunit` (database units per nm) | R `lib` |

#### SCAN FRAME OVERLAY (1206)

| Function | Line | Purpose | State |
|---|---|---|---|
| `rebuildFrameOverlay()` | 1209 | drop `_scanOverlay` boxes from `activeCell.boundaries`, push one closed square per frame on layer 500, dt `SIZE_TO_DT[sizeUm]` (else 4); register colours; call `rebuildLayerList` | R `lib`, `activeCell`, `scanFrames`; W `activeCell.boundaries`, `renderCacheDirty=true` |
| `rebuildLayerList()` | 1232 | list every boundary/path `layer/dt` in all cells, sorted; label adds `(S × S µm)` for 500/dt; click toggles visibility | R `lib`, `hiddenLayers`, `DT_TO_SIZE`; W `hiddenLayers`; DOM `#layer-list`, `#st-layers` |

#### FLAT RENDER CACHE (1264)

| Function | Line | Purpose | State |
|---|---|---|---|
| `buildRenderCache()` | 1266 | rebuild `renderCache` from `activeCell` | W `renderCache`, `renderCacheDirty=false` |
| `flattenCell(cell,cells,tx,ty,vis)` | 1272 | recursive flatten, SREF translation `r.xy[0]` only, per-branch cycle guard; does not skip `_scanOverlay` | W `renderCache` |

#### PLS / JS / TXT GENERATORS (1310)

| Function | Line | Purpose | State |
|---|---|---|---|
| `getWF(sizeUm)` | 1330 | `WF_TABLE[SIZE_TO_WF_IDX[sizeUm]]`; else first entry with `wf > sizeUm`; else last entry | R `WF_TABLE`, `SIZE_TO_WF_IDX` |
| `getOrderedFramesForExport()` | 1342 | `[{f, groupName}]`: each group's frames in `frameGroups` order, then ungrouped frames in `scanFrames` order as `'Unlinked'`; ignores hidden flags | R `frameGroups`, `scanFrames` |
| `getP()` | 1359 | read Step 3 form with `\|\|` fallbacks; returns `{ssc, sscStart, lot, wafer, slot, ptsU, ptsV, avg, dir, frameAvg, lineAvg, waitSec, plsfile, txtfile, jsfile, gdsfile}` | R DOM `#p-*` |
| `generatePLS()` | 1381 | `.pls` text (2.4) | R `getP`, `getOrderedFramesForExport`, `getWF`, `profileFor`, `toUm`, `umToMM`, `PLS_*` |
| `generateJS()` | 1412 | `.js` text (2.4); iterates `scanFrames` directly | R `scanFrames`, `getP`, `profileFor`, `toUm` |
| `generateTXT()` | 1422 | `.txt` text (2.4) | R `getOrderedFramesForExport`, `SIZE_TO_MIN_SPACING`, `toUm` |

#### GDS DOWNLOAD (1439)

| Function | Line | Purpose | State |
|---|---|---|---|
| `buildScanFrameRecords()` | 1442 | Uint8Array, 64 bytes per frame: BOUNDARY, LAYER 500, DATATYPE, XY (5 pts, `Math.round`), ENDEL; nested `wr` 1445, `b16` 1451, `b32` 1452 | R `scanFrames` (array order), `lib.dbunit`, `SIZE_TO_DT` |
| `findCellEndStrPos(buf,cellName)` | 1474 | byte offset of ENDSTR of the structure whose STRNAME is `cellName`, or -1 | pure |
| `stripLayer500FromCell(buf,cellName)` | 1489 | new ArrayBuffer without BOUNDARY elements whose LAYER is 500 (any dt) inside `cellName` | pure |
| `stripLayer500DTFromCell(buf,cellName,targetDt)` | 1549 | same, LAYER 500 and DATATYPE `targetDt` only | pure |
| `downloadGDS()` | 1588 | if `rawGDSBuffer && activeCell`: strip, find ENDSTR (fallback `length-4`), splice records, save; else stub lib `SCANFRAMES`/`SCAN_FRAMES` via `writeFullGDS`. Returns early if no frames | R `rawGDSBuffer`, `activeCell`, `scanFrames`, `lib`, `getP` |

#### RASTER SORT (1627)

| Function | Line | Purpose | State |
|---|---|---|---|
| `rasterSort(frames)` | 1629 | sort by ymin, group into rows by any y-overlap (touching counts), sort rows by `cx`, concat rows in creation order (bottom-up). Uses `bboxYmin/bboxYmax` if present | R `lib.dbunit` |

#### IMPORT (1657)

| Function | Line | Purpose | State |
|---|---|---|---|
| `setImportMode(mode)` | 1659 | `'auto'`/`'custom'` tabs; sets panel `display` `'block'`/`'flex'`/`'none'` | W `importMode` (never read); DOM |
| `collectBoundaries(cell,cells,tx,ty,vis)` | 1666 | all boundaries under a cell, translated (SREF translation only), skipping `_scanOverlay` → `[{layer,datatype,xy}]` | pure |
| `polyBBox(xy)` | 1684 | `{mnX,mnY,mxX,mxY}` | pure |
| `importAutoFrames()` | 1689 | for dt in `0..DT_TO_SIZE.length-1`: boxes on 500/dt → frames (size `DT_TO_SIZE[dt]`, centre = bbox midpoint), raster-sorted per dt, concatenated in dt order. Mirrors all layer-500 boundaries to 501 in every cell, hides `501/0..N-1`, removes layer 500 from every cell, strips 500 from `rawGDSBuffer` (active cell only). Replaces all frames and groups, clears undo, makes group `'500/* Import'` | W `scanFrames`, `frameGroups`, `activeGroupId`, `undoStack`, `lib.cells`, `hiddenLayers`, `rawGDSBuffer` |
| `toggleImportSinglePanel()` | 1736 | show/position `#import-single-panel` above (or below) `#import-single-btn` | DOM |
| `importSingleLayer500(dt)` | 1757 | one 500/dt; same mirror/strip for that dt only; appends; pushes `{type:'import',count}`; group `getUniqueGroupName('500/'+dt+' ('+size+'µm)')` | W as above (one dt), `undoStack` |
| `importCustomFrames()` | 1797 | boxes on `#import-layer`/`#import-dt` → frames of size `#import-size-sel`; raster-sorted, appended, undoable; group `'Layer L/D (Nµm)'`; does not modify `lib` or `rawGDSBuffer` | W `scanFrames`, `undoStack`, `frameGroups` |
| `reassignIds()` | 1819 | `scanFrames[i].id=i+1` | W `scanFrames[].id` |

#### SNAP (1822), GO TO FRAME (1831), MODAL (1854)

| Function | Line | Purpose | State |
|---|---|---|---|
| `snapAllFrames()` | 1824 | `snapToNm` every `cx`,`cy`; not undoable | W `scanFrames`; DOM `#scan-hint` |
| `goToFrame(idx)` | 1833 | zoom so the frame is 1/3 of canvas height, centre it, select it | W `zoom`, `panX`, `panY`, `selectedFrameIndices`, `lastSelectedIdx`, `selectedFrameIdx` |
| `showModal(title,desc,result,warn)` | 1856 | fill and show `#import-modal`; focus OK after 50 ms | DOM |
| `closeModal()` | 1868 | hide `#import-modal` | DOM |

#### PREVIEW / DOWNLOAD (1871)

| Function | Line | Purpose | State |
|---|---|---|---|
| `switchPreview(w)` | 1873 | toggle `active` on `'tab-'+t` and `'preview-'+t` for t in `['pls','js','txt']`; then `refreshPreviews` | W `activePreview` (never read) |
| `refreshPreviews()` | 1881 | fill `#preview-pls/js/txt` with all three generators | DOM |
| (input listener) | 1886 | on `input` in `.param-input,#p-dir,#p-frame-avg,#p-lavg`, `refreshPreviews()` if `currentStep===3`; bound once at load | R `currentStep` |
| `saveFileAs(blob,suggestedName,types)` | 1889 | `showSaveFilePicker` if present; AbortError ignored; other errors and no-API → `showDownloadModal` | none |
| `showDownloadModal(blob,filename)` | 1902 | data: URI on `#dl-modal-link`, show `#download-modal` | DOM |
| `downloadFile(type)` | 1923 | `'pls'` → `generatePLS`, `'js'` → `generateJS` (mime `text/javascript`), anything else → `generateTXT`; names from `getP` | R `getP` |

#### STEP NAV (1932), CANVAS SETUP (1964)

| Function | Line | Purpose | State |
|---|---|---|---|
| `goStep(n)` | 1934 | n in 1..3; hide `#array-popup`, cancel array placing, toggle `.step-page`/`.step-tab` by DOM index, Back/Next state and label; after 20 ms resize target canvas and shift pan; step 1: `rebuildFrameOverlay`+`redraw`; 2: `redrawScan`; 3: `showSEMModels` then `buildExport` | W `currentStep`, `panX`, `panY` |
| `resizeCanvas(c)` | 1973 | canvas size = parent `clientWidth/Height` (no devicePixelRatio) | DOM |
| `w2s(x,y)` | 1974 | `[x*zoom+panX, -y*zoom+panY]` | R view |
| `s2w(sx,sy)` | 1975 | `[(sx-panX)/zoom, -(sy-panY)/zoom]` | R view |
| `getBBox(cell,cells,vis)` | 1977 | recursive bbox of boundaries and path centrelines (SREF translation only); empty → `{0,0,1000,1000}` | pure |
| `fitView(c)` | 1993 | fit `activeCell` bbox with 20 px margin per side | W `zoom`, `panX`, `panY` |

#### DRAW (2004)

| Function | Line | Purpose | State |
|---|---|---|---|
| `drawCached(c2,cw,ch)` | 2008 | draw `renderCache` (rebuild if dirty), skip hidden layers. poly/path: viewport cull, skip if screen bbox diagonal < 1.5 px; polys filled `col+'28'` (when `zoom>0.0001`) and stroked; paths stroked along the centreline at `(width\|\|1)*zoom` (min 1 px), never filled. text: `str` at 8-20 px, culled by screen position only | R `renderCache`, `hiddenLayers`, view |
| `drawFrames(c2,noHL)` | 2049 | draw visible frames: dashed box, label `'#'+(i+1)+' '+sizeUm+'µm'`, centre dot; `noHL` disables selection highlight | R `scanFrames`, `selectedFrameIndices`, `movingIdx`, `movingMultiple`, `moveMultiOrigins` |
| `drawSelectBoxOverlay(c2)` | 2081 | rubber band (screen px) | R `selectBox*` |
| `drawGrid(c2,c)` | 2089 | grid every 1000 database units, hidden if < 6 px | R view |
| `axisSnap(wx,wy)` | 2098 | constrain to horizontal/vertical from `rulerPt1` (Shift) | R `rulerPt1` |
| `drawRulerOverlay(c2,cw,ch)` | 2107 | ruler line and µm label (3 dp) | R `rulerPt1`, `rulerPt2`, `lib` |
| `drawZoomBoxOverlay(c2,activeCanvas)` | 2131 | zoom box; up-left drag = zoom out | R `zoomBox*` |
| `redraw()` | 2146 | Step 1 paint via rAF: background, grid, layout, tool overlays (no `drawFrames`) | W `RAF_ID_DRAW` |
| `redrawScan()` | 2159 | Step 2 paint: + `drawFrames`, placing preview, array preview | W `RAF_ID_SCAN` |
| `redrawExport()` | 2181 | Step 3 paint: + `drawFrames(ectx,true)`; no select overlay | W `RAF_ID_EXPORT` |

#### TOOL SELECTION HELPERS (2195), EDGE SNAP (2230), HIT TEST FRAME CENTRE (2259), APPLY ZOOM BOX (2309)

| Function | Line | Purpose | State |
|---|---|---|---|
| `getCurrentTool()` | 2197 | tool of `currentStep` | R `tool1..3` |
| `activateToolBtn(step,tool)` | 2199 | ids map 2200, `toolMap` 2201; move `active` class | DOM |
| `setTool(step,tool)` | 2207 | set `toolN`, cursor, clear ruler/zoom-box/select-box/multi-move state, `#st-mode`, redraw | W `tool1..3` and those states |
| (toolbar wiring) | 2222 | `[step,id,tool]` rows → `setTool` | — |
| `snapToEdge(wx,wy)` | 2232 | nearest visible vertex/edge within 12 px (ruler only) | R `renderCache`, `hiddenLayers`, `zoom` |
| `isFrameHidden(i)` | 2261 | in a hidden group, or ungrouped while `unlinkedHidden` | R `frameGroups`, `unlinkedHidden` |
| `hitTestFrameForSelect(wx,wy)` | 2269 | topmost visible frame whose centre or border is within 8 px; -1 if none | R `scanFrames`, `zoom` |
| `framesInSelectBox(sxMin,syMin,sxMax,syMax)` | 2291 | indices of visible frames whose centre (or all corners) lie in the screen box | R `scanFrames`, view |
| `applyZoomBox(activeC)` | 2311 | apply zoom box if > 8 px both ways; clear state | W `zoom`, `panX`, `panY`, `zoomBox*` |

#### PAN / ZOOM (2330), GLOBAL ESC HANDLER (2574), TOOLBAR ZOOM BUTTONS (2685)

| Function | Line | Purpose | State |
|---|---|---|---|
| `setupPanZoom(c,onRedraw,step)` | 2332 | attaches all canvas mouse handlers; called at 2569-2571 for steps 1-3 | local `dragging`, `dragStart`, `moveDragStart` |
| ↳ `mousedown` | 2338 | priority: step 2 placing (2345: `parseInt(#frame-size-sel)`, `snapToNm`, push frame and `{type:'place'}`), step 2 array placing (2358: `placeArrayAt`), select (2363: hit → Shift toggles the frame, plain click on an unselected frame replaces the selection, on a selected one keeps it; on step 2 this starts a multi-move of the selection. Miss → clear selection unless Shift, start rubber band), ruler (2397), zoom box (2408), pan (2418); middle button pans | W `scanFrames`, `undoStack`, `placingMode`, selection, tool states |
| ↳ `mousemove` | 2425 | multi-move with `snapToNm`, pan, rubber band (Shift makes it square), ruler/zoom-box preview, placing and array previews, `#st-cursor`, zoom label (2480) | W frames, view, previews |
| ↳ `mouseup` | 2483 | finish multi-move (push `{type:'moveMulti'}`), rubber band, zoom box | W `undoStack`, selection |
| ↳ `mouseleave`, `wheel`, `contextmenu` | 2535, 2545, 2553 | leave: clear placing/array previews, abort zoom-box and rubber-band drags; does not end a pan or multi-move (2.5 item 56). wheel: zoom about the cursor by `zoomSensitivity` per event. contextmenu: recentre on the clicked point; with Ruler active also clear the ruler | W view, ruler, previews, `zoomBox*`, `selectBox*` |
| (document `keydown`) | 2576 | Enter: on `#clear-modal`, cancel only if `#clear-modal-cancel` (labelled Back) has `primary`, otherwise nothing; close `#import-modal`; on `#download-modal`, click `#dl-modal-link` if its href is set (and is not the page URL), then hide the modal after 200 ms. Delete/Backspace (focus not INPUT/TEXTAREA/SELECT): delete selection, push `{type:'deleteSelected'}`. Escape: cancel placing and array placing; in move/merge pick mode, `cancelPickMerge` and stop; with Select active, abort a multi-move (positions restored) or rubber band, clear the selection, and stop if any of these applied; otherwise clear ruler and zoom box and `setTool(currentStep,'pan')` | many |
| `zoomBtn(c,f,fn)` | 2687 | zoom about canvas centre; listeners 2688-2696 use 1.5 and 1/1.5; Fit calls `fitView` | W view |

#### SCAN FRAME CONTROLS (2699), incl. `── FGO popup ──` (2805)

| Function | Line | Purpose | State |
|---|---|---|---|
| (`#place-btn` click) | 2701 | toggle `placingMode`; alert if no `lib` | W `placingMode`, `previewFrame`; DOM |
| `undoLastAction()` | 2714 | pop and reverse `place` (2729), `move` (2733), `moveMulti` (2737), `import` (2740, removes last `count` frames), `deleteSelected` (2744); empty stack → restore `clearAllUndo` | W `scanFrames`, `frameGroups`, `undoStack`, `clearAllUndo`, selection |
| `clearAllFrames()` | 2759 | open `#clear-modal` with description | DOM |
| `closeClearModal()` | 2772 | hide `#clear-modal` | DOM |
| `doClearAll()` | 2775 | snapshot to `clearAllUndo`, then empty frames, groups, undo | W those |
| (bindings) | 2791-2803 | clear-modal hover swaps `primary`/`danger`; confirm, cancel, Undo last, Clear all, Snap all | — |
| `toggleFGOPopup()` | 2806 | open/position `#fgo-popup`; enable Move/Merge/Unlink by state | DOM |
| `closeFGOPopup()` | 2824 | hide it | DOM |
| `cancelPickMerge()` | 2830 | leave move/merge pick mode | W `fgoPickMode`, `mergeSelectedGroupIds` |
| (`#add-frame-group-btn`) | 2839 | selected frames → new group (removed from old groups first) | W `frameGroups`, selection |
| (`#fgo-move-btn`, `#fgo-merge-btn`, `#fgo-unlink-btn`) | 2848, 2857, 2866 | enter move pick; enter merge pick; drop all groups | W `fgoPickMode`, `frameGroups` |
| (`#confirm-merge-btn`) | 2872 | merge selected groups into the first in `frameGroups` order; re-sort | W `frameGroups` |
| (import bindings) | 2886-2887 | `#import-auto-btn` → `importAutoFrames`, `#import-custom-btn` → `importCustomFrames` | — |

#### FRAME GROUPS (2890), FRAME LIST (2955)

| Function | Line | Purpose | State |
|---|---|---|---|
| `getUniqueGroupName(base)` | 2892 | `base`, or `base+' (n)'` with smallest free n | R `frameGroups` |
| `makeGroupName()` | 2898 | `'Frame Group '+nextGroupNum++` made unique | W `nextGroupNum` |
| `getFrameGroup(fObj)` | 2899 | group containing the object (by reference) or null | R `frameGroups` |
| `removeFrameFromAnyGroup(fObj)` | 2903 | remove from its group; delete the group if empty; fix `activeGroupId` | W `frameGroups`, `activeGroupId` |
| `sortGroupFrames(grp)` | 2917 | sort `grp.frames` by `a.sizeUm-b.sizeUm` (stable, smallest first) | W `grp.frames` |
| `createFrameGroup(frameObjs,name)` | 2920 | new `{id,name,collapsed:false,frames}`, sorted, pushed, made active; null if empty | W `frameGroups`, `nextGroupId`, `activeGroupId` |
| `deleteFrame(idx)` | 2927 | row ✕: remove one frame; not undoable | W `scanFrames`, `frameGroups`, selection |
| `handleFrameClick(idx,e)` | 2934 | row selection: Shift+click selects the `scanFrames` index range from `lastSelectedIdx` (replacing the selection unless Ctrl/Cmd is also held); Ctrl/Cmd+click toggles; plain click selects only that row, or clears the selection if it was the only one selected | W selection |
| `updateFrameList()` | 2957 | rebuild `#frame-list`: groups in order of their first member in `scanFrames`, then "Unlinked Scans"; nested `buildFrameItem` 2961, `buildGroupHeader` 3006 (rename, collapse, eye, unlink, delete, pick/merge clicks 3074) | R most state; W via handlers |
| `updateStatusFrames()` | 3167 | `#st-frames` = count | DOM |

#### BUILD EXPORT (3170), LOAD LIB (3227), FILE I/O (3266), RESIZE (3298), MAPPING TOOLTIP (3313)

| Function | Line | Purpose | State |
|---|---|---|---|
| `selectSEMModel(model)` | 3173 | hide `#sem-models-panel`, add `visible` to `#raith-eline-panel` (hard-coded) | W `selectedSEMModel` (never read) |
| `showSEMModels()` | 3178 | reverse of the above | W `selectedSEMModel=null` |
| `buildExport()` | 3183 | count, empty state, enable 4 download buttons, build `#frame-tbody` (group rows, writefield rows, 7-cell frame rows), `refreshPreviews()` if N>0, `fitView(exportCanvas)` | R `getOrderedFramesForExport`, `getWF` |
| `loadLib(parsed)` | 3229 | set `lib`; reset `hiddenLayers` to `{'1/10'}`, colours; `buildCells`, `rebuildLayerList`; status bar; hide drop overlay; re-inject existing frames. Does not clear frames | W `lib`, `hiddenLayers`, `layerColors`, `renderCacheDirty` |
| `buildCells()` | 3242 | fill `#cell-list`; select first cell not starting with `$$$` (else the first cell) | DOM |
| `selectCell(name)` | 3257 | set `activeCell`, `rebuildFrameOverlay`, `fitView(canvas)`, `redraw` | W `activeCell`, `renderCacheDirty` |
| (file input / dblclick / demo / drop) | 3268, 3277, 3280, 3284-3295 | read ArrayBuffer → `rawGDSBuffer` and `loadLib(parseGDS(...))`; parse error → `rawGDSBuffer=null` + alert; Demo sets `rawGDSBuffer=null` and selects the last cell; `#canvas-wrap` dblclick opens the file picker only while no `lib` is loaded | W `rawGDSBuffer`, `lib` |
| `resizeAll()` | 3300 | size all three canvases to parents, redraw; `ResizeObserver` `ro` 3306-3309; first call 3310 | DOM |
| (mapping tooltip IIFE) | 3315 | hover positioning of the first `.mapping-tip` | DOM |

#### ARRAY POPUP & PLACEMENT (3338), SIDEBAR RESIZE (3525)

| Function | Line | Purpose | State |
|---|---|---|---|
| `getArrayMinSpacing()` | 3340 | `SIZE_TO_MIN_SPACING[size]\|\|size`, size `parseInt(#frame-size-sel)\|\|20` | R DOM |
| `positionAndShowArrayPopup()` | 3344 | show `#array-popup` under `#array-btn` | DOM |
| `closeArrayPopup()` | 3355 | hide; save valid rows/cols (1-49) and spacings (≥0, no minimum check) | W `arrayRows..arrayColSpacing` |
| `validateArrayInputs()` | 3366 | rows/cols 1-49, spacing ≥ minimum; marks `.err`, ≤2 messages in `#arr-err` | DOM |
| `startArrayPlacing()` | 3384 | validate, store, enter `arrayPlacingMode` | W array state, `placingMode` |
| `cancelArrayPlacing()` | 3405 | leave array mode, reset buttons | W `arrayPlacingMode`, `previewArrayPos` |
| `placeArrayAt(blX,blY)` | 3413 | rows×cols frames, centre `(blX+c*colSp, blY+r*rowSp)` (µm → database units), `snapToNm`; group `getUniqueGroupName('Scan Array')`; push `{type:'import',count}` | W `scanFrames`, `frameGroups`, `activeGroupId`, `nextGroupId`, `undoStack`, `arrayPlacingMode`, `previewArrayPos` |
| `drawArrayPreview(c2,blWorld)` | 3437 | dashed preview of the grid | R array state |
| (listeners) | 3458, 3484, 3485-3487, 3488, 3496, 3510 | `#array-btn` toggle; Place array; input validation; `#frame-size-sel` change resets spacings to minimum; min-spacing tooltip; Reset to defaults | — |
| `setupSidebarResize(sidebarEl,handleEl,minW,maxW,flipDrag)` | 3527 | drag width; calls 3566-3568 (120-500 px), the third (`#frames-sidebar`, 3568) with `flipDrag=true` | DOM |
| `setupRowResize(topEl,handleEl,minH)` | 3545 | drag height; call 3569 (min 60 px) | DOM |

### Top-level variables and constants

| Variable | Line | Initial value | Meaning | Written by |
|---|---|---|---|---|
| `zoomSensitivity` | 1099 | `1.15` | wheel zoom factor per event | `_applySens` |
| `lib`, `activeCell`, `hiddenLayers` | 1150 | `null`, `null`, `Set()` | parsed library; active cell object; hidden `'layer/dt'` keys | `loadLib`, `selectCell`, layer list, importers |
| `rawGDSBuffer` | 1151 | `null` | original file bytes for `.gds` export | file I/O, demo, importers (strip) |
| `currentStep` | 1152 | `1` | 1, 2 or 3 | `goStep` |
| `panX`, `panY`, `zoom` | 1153 | `0, 0, 1` | shared view; `zoom` = screen px per database unit | view functions |
| `scanFrames` | 1154 | `[]` | `[{id, sizeUm, cx, cy, bboxYmin?, bboxYmax?}]` | placement, importers, undo, delete |
| `placingMode`, `movingIdx`, `selectedFrameIdx`, `movingOrigin` | 1155 | `false, -1, -1, null` | single-placement flag; legacy drag (never set); last clicked (never read); legacy | various |
| `selectedFrameIndices`, `lastSelectedIdx` | 1156 | `Set()`, `-1` | selection by `scanFrames` index; range anchor | selection code |
| `selectBoxStart`, `selectBoxEnd`, `selectBoxDragging` | 1157 | `null, null, false` | rubber band, screen px | Select tool |
| `movingMultiple`, `moveMultiOrigins`, `moveMultiDragStart` | 1158 | `false, null, null` | Select-tool drag of selected frames | Select tool |
| `frameGroups`, `nextGroupId`, `nextGroupNum`, `activeGroupId` | 1159 | `[]`, `1`, `1`, `null` | groups `{id,name,collapsed,hidden?,frames}`; counters; highlighted group | group code, importers |
| `undoStack` | 1160 | `[]` | `{type:'place'\|'move'\|'moveMulti'\|'import'\|'deleteSelected',…}` | placement, imports, moves, delete |
| `clearAllUndo` | 1161 | `null` | one-level snapshot of Clear all | `doClearAll`, `undoLastAction` |
| `unlinkedHidden` | 1162 | `false` | hide ungrouped frames | Unlinked eye button |
| `importMode` | 1163 | `'auto'` | written, never read | `setImportMode` |
| `activePreview` | 1164 | `'pls'` | written, never read | `switchPreview` |
| `arrayRows`, `arrayCols`, `arrayRowSpacing`, `arrayColSpacing` | 1165 | `2, 2, 0.008, 0.008` | array size; centre-to-centre pitch in µm | array popup code |
| `arrayPlacingMode`, `previewArrayPos` | 1166 | `false, null` | array placing; preview anchor (world) | array code, mousemove |
| `fgoPickMode` | 1167 | `null` | `null\|'move'\|'merge'` | FGO popup |
| `mergeSelectedGroupIds` | 1168 | `Set()` | groups picked for merge | group header click |
| `tool1`, `tool2`, `tool3` | 1171 | `'pan'` | `'pan'\|'select'\|'ruler'\|'zoombox'` per step | `setTool` |
| `renderCache`, `renderCacheDirty` | 1174-1175 | `[]`, `true` | flattened records in world database units: `{type:'poly',layer,datatype,xy,wxmin,wxmax,wymin,wymax}`, `{type:'path',…same…,width}`, `{type:'text',layer,datatype,str,wx,wy}`; rebuild flag | `buildRenderCache`; set dirty at 1228, 3235, 3260 |
| `frameOverlayBounds` | 1179 | `[]` | unused | — |
| `LPAL` | 1181 | 11 colours | layer palette | — |
| `SCAN_COLORS` | 1182 | 10 colours | frame palette by index | — |
| `DT_TO_SIZE` | 1184 | `[1,5,10,15,20,25,30,50,75,100]` | datatype → size µm | — |
| `SIZE_TO_DT` | 1185 | `{1:0,5:1,10:2,15:3,20:4,25:5,30:6,50:7,75:8,100:9}` | inverse | — |
| `SIZE_TO_PROFILE` | 1186 | `{1:'1um_bss1nm',5:'5um_bss2nm',10:'10um_bss4nm',15:'15um_bss6nm',20:'20um_bss8nm',25:'25um_bss10nm',30:'30um_bss12nm',50:'50um_bss20nm',75:'75um_bss30nm',100:'100um_bss40nm'}` | scan-manager preset names | — |
| `SIZE_TO_MIN_SPACING` | 1190 | `{1:0.001,5:0.002,10:0.004,15:0.006,20:0.008,25:0.010,30:0.012,50:0.020,75:0.030,100:0.040}` | beam step µm | — |
| `layerColors`, `layerColorIdx` | 1191 | `{}`, `0` | colour cache | `getLC`, reset in `loadLib` |
| `PLS_FORMAT`, `PLS_COLS`, `TRAILING` | 1312-1314 | see 2.4 | `.pls` schema | — |
| `WF_TABLE` | 1319 | 7 entries, see 2.4 | writefields | — |
| `SIZE_TO_WF_IDX` | 1329 | `{1:1,5:2,10:3,15:3,20:4,25:4,30:4,50:5,75:5,100:6}` | size → `WF_TABLE` index | — |
| `canvas`/`ctx`, `scanCanvas`/`sctx`, `exportCanvas`/`ectx` | 1966-1971 | `#gds-canvas`, `#scan-canvas`, `#export-canvas` | canvases and 2D contexts | — |
| `RAF_ID_DRAW`, `RAF_ID_SCAN`, `RAF_ID_EXPORT` | 2006 | `0` | rAF handles for redraw coalescing | redraw functions |
| `rulerPt1`, `rulerPt2`, `rulerComplete` | 2097 | `null, null, false` | ruler points (world) | ruler code |
| `zoomBoxStart`, `zoomBoxEnd`, `zoomBoxDragging` | 2105 | `null, null, false` | zoom box (screen px) | zoom-box code |
| `previewFrame` | 2158 | `null` | `{cx,cy,sizeUm}` ghost while placing | mousemove |
| `selectedSEMModel` | 3172 | `null` | chosen model id; never read | `selectSEMModel`, `showSEMModels` |
| `ro` | 3306 | `ResizeObserver(resizeAll)` | canvas resize | — |

## 2.3 DOM contract

Ids the script reads or writes, by area, with the code that uses them. Ids the script never looks up by name: `#main`, `#layers-section`, `#frame-list-wrap`, `#export-page`, `#statusbar`, `#step-header` (styled by CSS); `#tab-1..3`, `#page-1..3` (reached by DOM order, see Header); `#tb-1..3` (unused). The SVG symbols `#i-*` (505-516) are referenced only by `<use href>` in the markup.

### Header
- `.step-tab` (`#tab-1..3`, 521-525) and `.step-page` (`#page-1..3`): `goStep` toggles `active`/`done` by DOM index (1942-1947). Order matters.
- `#btn-prev`, `#btn-next` (527-528): `goStep` (1948-1950). Inline `onclick` uses global `currentStep`.

### Step 1
- `#file-input` (537): `change` listener 3268; `.click()` from `#canvas-wrap` dblclick 3278, only while no library is loaded. The Open GDS `<label for="file-input">` (536) and the CSS rule that hides the input (128) also use this id.
- `#demo-btn` (538): 3280.
- `#btn-fit`, `#btn-zoomin`, `#btn-zoomout` (540-542): 2688-2690.
- `#btn-pan`, `#btn-select`, `#btn-ruler`, `#btn-zoombox` (544-547): `activateToolBtn` 2200, wiring 2222.
- `#zsw-1` (549): `toggleZoomSens`. `#zoom-info` (563): mousemove 2480.
- `.sidebar` + `#sb1-handle` (566-567): 3566. `#cells-section` + `#sb1-row-handle` (568, 574): 3569.
- `#cell-list` (570): `buildCells`. `#layer-list` (577): `rebuildLayerList`.
- `#canvas-wrap` (580): dblclick 3277, `ResizeObserver` 3307. `#gds-canvas` (581): `canvas` 1966. `#drop-overlay` (582): `loadLib` 3238.

### Step 2, left panel
- `#s2-fit`, `#s2-zoomin`, `#s2-zoomout` (596-598): 2691-2693. `#s2-pan`, `#s2-select`, `#s2-ruler`, `#s2-zoombox` (600-603): 2200, 2223.
- `#zsw-2` (605). `#zoom-info-2` (621): mousemove 2480 (also written on step 3).
- `#scan-sidebar` + `#sb2-handle` (624-625): 3567.
- `#frame-size-sel` (630): read with `parseInt` at 2346, 2470, 3341, 3414, 3439; `change` listener 3488.
- `#place-btn` (644): 2701; class/text changes at 2351-2353, 2632-2633, `startArrayPlacing`, `cancelArrayPlacing`, `placeArrayAt`, `#array-btn` handler.
- `#array-btn` (645): 3458; `positionAndShowArrayPopup`, `startArrayPlacing`, `cancelArrayPlacing`, `placeArrayAt`.
- `#imode-auto`, `#imode-custom`, `#import-auto-panel`, `#import-custom-panel` (651-673): `setImportMode`.
- `#import-auto-btn` (656): 2886. `#import-single-btn`, `#import-single-panel` (657-660): `toggleImportSinglePanel`, document click 1135-1139, `importSingleLayer500` 1794-1795.
- `#import-layer`, `#import-dt`, `#import-size-sel` (676-682): `importCustomFrames` 1799-1801. `#import-custom-btn` (690): 2887.
- `.mapping-tip-trigger`, `.mapping-tip` (655): IIFE 3315 (first match only).

### Step 2, canvas and right panel
- `#scan-canvas` (695): `scanCanvas` 1968; parent `.canvas-area` observed 3308.
- `#frames-sidebar` + `#sb3-handle` (697-698): 3568.
- `#snap-all-btn` (702): 2803. `#scan-hint` (704): status text written by many functions.
- `#undo-btn2` (706): 2801. `#clear-btn2` (707): 2802.
- `#add-frame-group-btn` (710): 2839. `#fgo-toggle-btn` (711): 2847, `toggleFGOPopup`, `closeFGOPopup`, document click.
- `#pick-action-row`, `#cancel-pick-btn` (713-715): 2853, 2870, `cancelPickMerge`.
- `#merge-action-row`, `#confirm-merge-btn`, `#cancel-merge-btn` (717-719): 2861-2862, 2871, 2872, 3095-3096, `cancelPickMerge`.
- `#frame-list` (721): `updateFrameList`. `#frame-count` (723): 3164.

### Step 3
- `#s3-fit`, `#s3-zoomin`, `#s3-zoomout` (733-735): 2694-2696. `#s3-pan`, `#s3-select`, `#s3-ruler`, `#s3-zoombox` (737-740): 2200, 2224.
- `#zsw-3` (742): `toggleZoomSens` opens it as on the other steps, but its slider (746) and number input (750) call `updateZoomSens`, which is not defined (2.5 item 48).
- `#export-canvas` (759): `exportCanvas` 1970; parent `.export-preview` observed 3309.
- `#exp-frame-count`, `#no-frames-msg`, `#frame-table`, `#frame-tbody` (761-766): `buildExport`. Table header (765) has 7 columns; `buildExport` uses `colspan="7"` (3203, 3211) and 7 cells (3218).
- `#sem-models-panel` (770), `#raith-eline-panel` (779): `selectSEMModel`, `showSEMModels`.
- `#p-ssc`, `#p-ssc-start`, `#p-lot`, `#p-wafer`, `#p-slot`, `#p-pts-u`, `#p-pts-v`, `#p-avg`, `#p-dir`, `#p-frame-avg`, `#p-lavg`, `#p-wait`, `#p-plsfile`, `#p-txtfile`, `#p-jsfile`, `#p-gdsfile` (786-823): `getP` 1360-1378; live refresh listener 1886 (class `param-input` plus the three selects by id).
- `#tab-pls`, `#tab-js`, `#tab-txt` (826-828): `switchPreview` via `'tab-'+t`. `#preview-pls`, `#preview-js`, `#preview-txt` (830-832): `switchPreview`, `refreshPreviews`.
- `#dl-pls-btn`, `#dl-js-btn`, `#dl-txt-btn`, `#dl-gds-btn` (834-837): inline `downloadFile(...)`/`downloadGDS()`; `disabled` set by `buildExport` 3190.

### Status bar
- `#st-cells` (3236), `#st-layers` (1260), `#st-frames` (3167), `#st-unit` (3237), `#st-mode` (2217, 2515), `#st-cursor` (2479).

### Modals
- `#import-modal`, `#modal-title`, `#modal-desc`, `#modal-result` (854-858); `#import-modal .modal-btn.primary` is focused by `showModal` (1864). Used by `showModal`, `closeModal`, keydown 2588.
- `#download-modal`, `#dl-modal-title`, `#dl-modal-link` (865-872): `showDownloadModal`, keydown 2590-2596.
- `#clear-modal`, `#clear-modal-desc`, `#clear-modal-cancel`, `#clear-modal-confirm` (880-886): `clearAllFrames`, `closeClearModal`, `doClearAll`, 2791-2800, keydown 2579-2585.

### Popups
- `#array-popup` (891): `positionAndShowArrayPopup`, `closeArrayPopup`, `startArrayPlacing`, `goStep` 1936, document click 1133-1134. Has `onclick="event.stopPropagation()"`.
- `#arr-rows`, `#arr-cols`, `#arr-row-sp`, `#arr-col-sp`, `#arr-err` (896-911): array functions and listeners 3458-3522.
- `#arr-clear-btn` (912): 3510. `#arr-min-sp-btn` (913) and `#arr-min-tip` (917): IIFE 3496. `#place-array-btn` (914): 3484 and state changes in array functions and Escape (2639-2640).
- `#fgo-popup`, `#fgo-move-btn`, `#fgo-merge-btn`, `#fgo-unlink-btn` (935-938): `toggleFGOPopup`, `closeFGOPopup`, 2848-2869, document click 1140-1144.

### Classes and inline display

Classes the script queries: `.step-page`, `.step-tab` (1942-1943), `.zoom-sens-panel`, `.zoom-sens-slider`, `.zoom-sens-num`, `.zoom-sens-err-msg`, `.zoom-sens-val` (1103-1130), `.param-input` (1886), `.cell-item` (3258), `.mapping-tip-trigger`, `.mapping-tip` (3316-3317), `.sidebar` (3566), `#import-modal .modal-btn.primary` (1864); in rows built by `updateFrameList`, `.frame-goto`, `.frame-del`, `.frame-unlink`, `.frame-check` (2977-2988).

Classes the script toggles (keep them and their CSS):

| Class | Set on | Note |
|---|---|---|
| `active` | step tabs, pages, tool buttons, `#array-btn`, import tabs, preview tabs and panes, cell items | |
| `done` | step tabs | |
| `hidden` | modals, `.zoom-sens-panel`, `#array-popup`, `#drop-overlay`, `.layer-item` | |
| `visible` | `#raith-eline-panel`, `#pick-action-row`, `#merge-action-row` | |
| `open` | `#import-single-btn`, `#fgo-toggle-btn` | no rule on `#fgo-toggle-btn` |
| `placing` | `#place-btn`, `#place-array-btn` | |
| `inactive` | `#place-btn` | no rule |
| `primary`, `danger` | clear-modal buttons | read by the Enter logic |
| `warn` | `#modal-result` | |
| `err` | array inputs | |
| `zoom-sens-error` | zoom number inputs | |
| `drag-over` | body | |
| `dragging` | resize handles | |
| `editing` | group name | |
| `in-group`, `selected`, `moving` | `.frame-item` | built by `updateFrameList` |
| `checked` | `.frame-check` | |
| `active-group`, `merge-selected`, `pick-active`, `group-vis-hidden` | `.group-header` | |
| `vis-hidden` | `.unlinked-header` | |
| `ft-group-hdr`, `ft-wf-change`, `td-dim` | Step 3 table rows | set by `buildExport` |

Elements shown and hidden with inline `style.display` (once set, the inline value overrides any CSS `display` rule): `#import-auto-panel` (`block`, see 2.5 item 44) and `#import-custom-panel` (`flex`) (1663-1664); `#import-single-panel` and `#fgo-popup` (`block`; their open state is read back as `style.display==='block'` at 1137, 1142, 1739, 2809, so open them with `block`); `#sem-models-panel` (`none`, and `''` to restore the CSS value; 3175, 3180); `#no-frames-msg` (`flex`) and `#frame-table` (`table`) (3188-3189); `#modal-result` (1860); `.zoom-sens-err-msg` (1108, 1120); `.mapping-tip` (3320, 3332); `#arr-min-tip` (3501, 3508).

## 2.4 File-format contracts

All text outputs use `\n` line endings and are saved as Blobs (`text/plain`, or `text/javascript` for the `.js`). Default names come from `getP` (1375-1378).

### `.pls` (Raith position list), `generatePLS` (1381)

Overall layout:

```
[HEADER]
FORMAT=IXYZRTUVWATC,Options,0,Type,0,...,Frame,0,LAvg,0      <- PLS_FORMAT (1312)
WAFERLAYOUT=DEFAULT.wlo
LotID=<#p-lot>
WaferID=<#p-wafer>
Slot=<#p-slot>
MinimizeWin=FALSE
                                                               <- blank line
[COLUMNS]                                                      <- PLS_COLS (1313): '[COLUMNS]' + 47 lines
No.=W:25,!VISIBLE,!SHOWDIM
ID=W:25,VISIBLE,!SHOWDIM
...
LAvg=W:25,VISIBLE,!SHOWDIM
                                                               <- blank line
[DATA]
<data rows>
```

With no frames the output ends at `[DATA]\n` (1387). `PLS_COLS` lines have the form `Name=W:<width>,VISIBLE|!VISIBLE[,DEFAULT:<x>][,DIM:<unit>],SHOWDIM|!SHOWDIM`; the first entry, `No.`, is not a data column.

Data columns, in order. `PLS_FORMAT` declares 12 single-letter columns (`IXYZRTUVWATC`) followed by 34 named columns, each written as `Name,0`: 46 fields per row.

| # | Column | WRITEFIELD row (1397) | WAIT row (1405) | SLOWSCAN row (1408) |
|---|---|---|---|---|
| 1 | ID | helper id | helper id | `i+1` |
| 2-6 | X, Y, Z, R, T | `0.000000` | `0.000000` | `0.000000` |
| 7 | U | `0.000000` | `u` | `u` |
| 8 | V | `0.000000` | `v` | `v` |
| 9 | W | `0.000000` | `0.000000` | `0.000000` |
| 10 | Attribute | `FN` | `MN` | `IS` |
| 11 | Template | `dUV` | `UV` | `UV` |
| 12 | Comment | empty | `wait_<waitSec>s` | `profileFor(sizeUm)`, e.g. `20um_bss8nm` |
| 13 | Options | `STAY` | empty | empty |
| 14 | Type | `WRITEFIELD` | `MACRO` | `SLOWSCAN` |
| 15 | Size-U | `wf.size`, e.g. `50.000` | empty | `sizeUm`, e.g. `20` |
| 16 | Size-V | `wf.size` | empty | `sizeUm` |
| 17 | Points-U | empty | empty | `ptsU` |
| 18 | Points-V | empty | empty | `ptsV` |
| 19 | Dir | empty | empty | `dir` |
| 20 | Avg | empty | empty | `avg` |
| 21-24 | Pos1, Pos2, Pos3, Link | empty | empty | empty (start of `TRAILING`) |
| 25 | File | `"<wf.label>"` (double-quoted) | `%UserRoot%SCRIPT\WAIT.JS` | empty |
| 26-44 | Layer, Area, DoseFactor, Dwelltime, Stepsize, SplDwell, SplStep, CurveStep, CurveDwell, DotDwell, FBMSArea, FBMSLines, SplDot, Time, Timestamp, Method, Dot, Angle, MarkSequences | empty | empty | empty (`TRAILING` = 24 commas covers fields 21-44) |
| 45 | Frame | empty | empty | `frameAvg` |
| 46 | LAvg | empty | empty | `lineAvg` |

Values and formatting:
- `u`, `v` = `umToMM(toUm(f.cx))`, `umToMM(toUm(f.cy))` (1402-1403): frame centre in mm, `toFixed(6)`, negative sign kept, measured from the active cell's origin, no offset.
- `sizeUm` is printed as the JS number (`20`, not `20.000`). `wf.size` is the string from `WF_TABLE` (`'20.000'`).
- `ptsU`, `ptsV`, `avg`, `frameAvg`, `lineAvg` are integers from `parseInt`; `dir` is `U` or `V`; `waitSec` is printed as a JS number (`5`, `2.5`). Because `getP` uses `||default` (1368-1374), 0 or an empty field gives the default (Wait 0 → `wait_5s`, Points 0 → 2500, Avg 0 → 2); see 2.5 item 50.
- In the source, the WAIT file path is written `'%UserRoot%SCRIPT\\WAIT.JS'`; the output has one backslash.
- The WAIT row has the same U, V as the SLOWSCAN that follows it.

Row order and ids (loop 1391-1409), for each frame `i` in `getOrderedFramesForExport()` order (every frame, including those in hidden groups or a hidden Unlinked set; 2.5 item 36):
1. If `i===0` or `getWF(sizeUm).wf` differs from the current writefield: one WRITEFIELD row.
2. One WAIT row.
3. One SLOWSCAN row with ID `i+1`.

Helper rows (WRITEFIELD, WAIT) take ids `N+1, N+2, …` in order of appearance (`nextId`, 1389), so the ID column is not monotonic. SLOWSCAN ids 1..N equal the `.txt` scan_number and the Step 3 table "No." column.

Example `[DATA]` section (unmodified file, defaults; a group `G A` holding a 5 µm frame at (500, 500) µm and a 20 µm frame at (54, 24) µm, then ungrouped frames of 20 µm at (24, 24) µm and 100 µm at (−1, 2.5) µm):

```
5,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,FN,dUV,,STAY,WRITEFIELD,10.000,10.000,,,,,,,,,"13000 x, 10.0 um",,,,,,,,,,,,,,,,,,,,,
6,0.000000,0.000000,0.000000,0.000000,0.000000,0.500000,0.500000,0.000000,MN,UV,wait_5s,,MACRO,,,,,,,,,,,%UserRoot%SCRIPT\WAIT.JS,,,,,,,,,,,,,,,,,,,,,
1,0.000000,0.000000,0.000000,0.000000,0.000000,0.500000,0.500000,0.000000,IS,UV,5um_bss2nm,,SLOWSCAN,5,5,2500,2500,U,2,,,,,,,,,,,,,,,,,,,,,,,,,1,2
7,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,FN,dUV,,STAY,WRITEFIELD,50.000,50.000,,,,,,,,,"2600 x, 50.0 um",,,,,,,,,,,,,,,,,,,,,
8,0.000000,0.000000,0.000000,0.000000,0.000000,0.054000,0.024000,0.000000,MN,UV,wait_5s,,MACRO,,,,,,,,,,,%UserRoot%SCRIPT\WAIT.JS,,,,,,,,,,,,,,,,,,,,,
2,0.000000,0.000000,0.000000,0.000000,0.000000,0.054000,0.024000,0.000000,IS,UV,20um_bss8nm,,SLOWSCAN,20,20,2500,2500,U,2,,,,,,,,,,,,,,,,,,,,,,,,,1,2
9,0.000000,0.000000,0.000000,0.000000,0.000000,0.024000,0.024000,0.000000,MN,UV,wait_5s,,MACRO,,,,,,,,,,,%UserRoot%SCRIPT\WAIT.JS,,,,,,,,,,,,,,,,,,,,,
3,0.000000,0.000000,0.000000,0.000000,0.000000,0.024000,0.024000,0.000000,IS,UV,20um_bss8nm,,SLOWSCAN,20,20,2500,2500,U,2,,,,,,,,,,,,,,,,,,,,,,,,,1,2
10,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000,FN,dUV,,STAY,WRITEFIELD,200.000,200.000,,,,,,,,,"650 x, 200.0 um",,,,,,,,,,,,,,,,,,,,,
11,0.000000,0.000000,0.000000,0.000000,0.000000,-0.001000,0.002500,0.000000,MN,UV,wait_5s,,MACRO,,,,,,,,,,,%UserRoot%SCRIPT\WAIT.JS,,,,,,,,,,,,,,,,,,,,,
4,0.000000,0.000000,0.000000,0.000000,0.000000,-0.001000,0.002500,0.000000,IS,UV,100um_bss40nm,,SLOWSCAN,100,100,2500,2500,U,2,,,,,,,,,,,,,,,,,,,,,,,,,1,2
```

Writefield table (`WF_TABLE`, 1319-1327) and size mapping (`SIZE_TO_WF_IDX`, 1329):

| idx | wf (µm) | size (Size-U/V string) | label (File column) | frame sizes mapped here |
|---|---|---|---|---|
| 0 | 1 | `1.000` | `130000 x, 1.0 um` | none (only via `getWF` fallback for sizes < 1) |
| 1 | 5 | `5.000` | `26000 x, 5.0 um` | 1 |
| 2 | 10 | `10.000` | `13000 x, 10.0 um` | 5 |
| 3 | 20 | `20.000` | `6500 x, 20.0 um` | 10, 15 |
| 4 | 50 | `50.000` | `2600 x, 50.0 um` | 20, 25, 30 |
| 5 | 100 | `100.000` | `1300 x, 100.0 um` | 50, 75 |
| 6 | 200 | `200.000` | `650 x, 200.0 um` | 100 |

Magnification × wf = 130000 in every label. The comment at 1318 says the writefield must exceed the scanframe size, and `SIZE_TO_WF_IDX` follows it. A size missing from `SIZE_TO_WF_IDX` falls through to the fallback in `getWF` (1332-1336), which returns the first entry with `wf` strictly greater than the size, or the 200 µm entry when none is larger. A frame of 200 µm or more therefore gets a writefield no larger than itself.

### `.js` (Raith App script), `generateJS` (1412)

Iterates `scanFrames` in array order (not export order). With no frames it returns `// No scan frames defined.` Otherwise, for two frames:

```js
// Raith eLine scan script
// Generated: 2026-10-13T19:59:17.212Z
// Frames: 2
//
var defaultName = App.GetSysVariable("ScanManager.SSCName");
var DataPath    = App.GetVariable("UserRoot") + "\\Data\\";
App.SetSysVariable("ScanManager.SSCName",  "ARPsi_%04d.SSC");
App.SetSysVariable("ScanManager.SSCIndex", "0001");

// [x_um, y_um, size_um, scan_profile]
var frames = [
  [24.000000, 24.000000, 20, "20um_bss8nm"]   // #1,
  [500.000000, 500.000000, 5, "5um_bss2nm"]   // #2
];

var Pls = App.NewPositionlist();
for (var id = 0; id < frames.length; id++) {
  var u            = frames[id][0] / 1000;
  var v            = frames[id][1] / 1000;
  var sz           = frames[id][2];
  var scan_profile = frames[id][3];
  var img1 = "ID," + id
    + ",U," + u + ",V," + v
    + ",Attribute,IS,Template,UV,Comment," + scan_profile
    + ",Type,SLOWSCAN,Size-U," + sz + ",Size-V," + sz
    + ",Points-U,2500,Points-V,2500"
    + ",Dir,U,Avg,2,Frame,1,LAvg,2";
  Pls.Add(img1);
}
// App.SetSysVariable("ScanManager.SSCName", defaultName);
```

- SSC name and index come from `#p-ssc` and `#p-ssc-start` and are used only here. The index is zero-padded to 4 digits; 0 or an empty field gives `0001` (`||1`, 1360). An empty prefix gives `ARPsi_%04d.SSC` (1363).
- Points, Dir, Avg, Frame and LAvg are baked into the string; `#p-wait` is not used; no WRITEFIELD or WAIT entries are written; IDs are 0-based. The position list is never saved or started by the script.
- Bug: note the `,` inside `// #1,`. With two frames the array literal parses as `[…][…]` (the second bracket is read as an index into the first), `frames` becomes `[undefined]` and the loop throws a TypeError at `frames[id][0]`. With three or more, `[…][…][…]` indexes `undefined` and the TypeError is thrown while `frames` is being built. One frame works (checked by evaluating the output against a stub `App`). Fix in 2.6 (d).

### `.txt` (lookup table), `generateTXT` (1422)

```
# scan_number, x_um, y_um, size_um, step_um, group_name
1, 500.000000, 500.000000, 5, 0.002, # G A
2, 54.000000, 24.000000, 20, 0.008, # G A
3, 24.000000, 24.000000, 20, 0.008, # Unlinked
4, -1.000000, 2.500000, 100, 0.04, # Unlinked
```

- Order is `getOrderedFramesForExport()`; `scan_number` = `.pls` SLOWSCAN ID.
- x, y: `toUm(...).toFixed(6)`, µm, active-cell origin.
- `step_um` = `SIZE_TO_MIN_SPACING[sizeUm]` (0 if missing), printed via `parseFloat(step.toFixed(6)).toString()`.
- `group_name` is written after `# ` unescaped; ungrouped frames are `Unlinked`. Names can contain commas.
- Ends with a trailing `\n`; empty string when there are no frames.

### `.gds` (annotated layout), `downloadGDS` (1588)

With a loaded file (`rawGDSBuffer && activeCell`):
1. `stripLayer500FromCell(rawGDSBuffer, activeCell.name)`: remove every BOUNDARY element whose LAYER is 500 (any datatype) inside the active cell's structure. Other cells, and PATH/TEXT elements, are untouched.
2. `findCellEndStrPos`: offset of that structure's ENDSTR; if not found, `length-4` (assumed ENDLIB).
3. Insert `buildScanFrameRecords()` at that offset. All other bytes, including HEADER, BGNLIB timestamps and UNITS, are the original.

Records per frame, in `scanFrames` array order (64 bytes):

| Record | Bytes | Payload |
|---|---|---|
| BOUNDARY | `00 04 08 00` | none |
| LAYER | `00 06 0D 02` + int16 | 500 (`01 F4`) |
| DATATYPE | `00 06 0E 02` + int16 | `SIZE_TO_DT[sizeUm]`, or 4 if missing |
| XY | `00 2C 10 03` + 5 × (int32 x, int32 y) | `(x0,y0),(x1,y0),(x1,y1),(x0,y1),(x0,y0)` with `x0=Math.round(cx-h)`, `x1=Math.round(cx+h)`, same for y, `h=sizeUm*1e-6/(2*dbunit)` |
| ENDEL | `00 04 11 00` | none |

No group, id or order information is written to the `.gds`.

Without source bytes (demo, or after a parse error): `writeFullGDS(stubLib,null,'')` writes HEADER version 7, BGNLIB (24 zero bytes), LIBNAME `SCANFRAMES`, UNITS `[0.001, dbunit]`, one structure `SCAN_FRAMES` holding the same boxes, ENDLIB.

### GDS input, `parseGDS` (945)

| Record (type/dtype) | Handled as |
|---|---|
| `0x0206` LIBNAME | `lib.name` |
| `0x0305` UNITS | `lib.units=[user, db]`, `lib.dbunit=db` (metres) |
| `0x0502` BGNSTR, `0x0606` STRNAME, `0x0700` ENDSTR | BGNSTR starts a cell named `cell_N`; STRNAME renames it and registers it in `lib.cells` (a repeated name overwrites); ENDSTR closes it |
| `0x0800` BOUNDARY, `0x0900` PATH, `0x0C00` TEXT, `0x0A00` SREF | new element |
| `0x0D02` LAYER, `0x0E02` DATATYPE, `0x1602` TEXTTYPE | signed int16; TEXTTYPE stored in `datatype` |
| `0x0F03` WIDTH | int32 |
| `0x1003` XY | int32 pairs |
| `0x1206` SNAME, `0x1906` STRING, `0x2A06` (string fallback) | strings |
| `0x1100` ENDEL | push element into the current cell |

Not handled (silently skipped): AREF (`0x0B00`) and COLROW, STRANS/MAG/ANGLE, BOX, NODE, PATHTYPE, properties. SREFs keep `mag:1, angle:0` and are applied as translation only.

## 2.5 Invariants and gotchas

Checked against the released file. Numbers are referenced elsewhere in this guide.

### A. Units and coordinates

1. `scanFrames[].cx/cy` are GDS database units of the active cell after translation-only flattening; `sizeUm` is µm. Physical conversion always goes through `lib.dbunit` (fallback `1e-9`). Do not store µm in `cx/cy`.
2. No origin offset is applied anywhere; U/V/x/y are measured from the active cell's (0, 0). To add one, change `generatePLS` (1402-1403), `generateJS` (1418), `generateTXT` (1428-1429) and `buildExport` (3216-3218) together, and leave the `.gds` in layout coordinates. The frame list (2972), the placement hint (2354) and the cursor and zoom read-outs (2478-2480) also call `toUm` with no offset, so they will disagree with the files unless they are changed too.
3. GDS y is up. `w2s`/`s2w` flip y. `rasterSort` emits rows bottom-up.
4. Rounding: `snapToNm` (1 nm grid) runs on single placement (2347), array placement (2359, 3423), Select-tool moves (2435-2436) and Snap all. Imported centres are bbox midpoints and can be half database units. The `.gds` rounds to integers (1458-1459); the `.pls`/`.txt` use the unrounded value.
5. In `snapToNm`, `nmPerUnit = 1e-9/dbunit` is database units per nm; with `dbunit > 1e-9` it snaps to fractional database units.
6. Frames are square and centred. The half-size `sizeUm*1e-6/(2*dbunit)` is recomputed in `rebuildFrameOverlay` (1218), `buildScanFrameRecords` (1457), `downloadGDS` stub (1614), `rasterSort` (1635), `drawFrames` (2054), `redrawScan` preview (2169), `hitTestFrameForSelect` (2276), `framesInSelectBox` (2298), `drawArrayPreview` (3441/3450). `goToFrame` (1838) uses the full size to set the zoom. Rectangular frames need all of these plus the frame object and every generator.
7. One view (`zoom`, `panX`, `panY`) is shared by all canvases; `zoom` is screen px per database unit, so it depends on `dbunit`. `goStep` shifts pan by half the canvas size change.

### B. Preset tables

8. `DT_TO_SIZE`, `SIZE_TO_DT` (exact inverse), `SIZE_TO_PROFILE`, `SIZE_TO_MIN_SPACING`, `SIZE_TO_WF_IDX` and five markup copies (`#frame-size-sel` 631-640, mapping tooltip 655, `.import-dt-btn` 661-670, `#import-size-sel` 683-687, `#arr-min-tip` 922-931) must list the same sizes. Nothing generates the markup from the tables.
9. A size missing from the tables fails silently: datatype 4 (1217, 1226, 1456, 1613), profile `size+'um'`, `.txt` step 0, minimum array spacing = the size (3342), writefield from the `getWF` fallback.
10. `SIZE_TO_MIN_SPACING` equals the `bss` value in `SIZE_TO_PROFILE` names (size/2500 for 5-100 µm; 0.001 for 1 µm). Neither follows `#p-pts-u/v`.
11. Array spacing is a centre-to-centre pitch in µm (3421-3422). The clicked point is the centre of the bottom-left frame (the hint at 3403 says "BL corner"). Default 0.008 µm equals `SIZE_TO_MIN_SPACING[20]` for the default 20 µm size (1165, 904, 908). Changing `#frame-size-sel` resets both spacings to the new minimum (3488). `closeArrayPopup` saves spacings without the minimum check; `startArrayPlacing` re-validates.
12. Sizes are read with `parseInt` (1801, 2346, 2470, 3341, 3414, 3439): non-integer sizes are truncated.

### C. Export order, ids, rows

13. `getOrderedFramesForExport` is the only source of order for `.pls`, `.txt` and the Step 3 table: groups in `frameGroups` (creation) order, each group's `frames` order, then ungrouped frames in `scanFrames` order as `'Unlinked'`. Hidden flags are ignored, so hidden frames are exported.
14. `.pls` SLOWSCAN ID = `.txt` scan_number = table "No." = position in that order, 1-based. Helper rows take N+1… . The sidebar `#N` is `scanFrames` index + 1 and differs once groups reorder frames.
15. `buildExport` (3197-3214) repeats the writefield change logic of `generatePLS` (1393-1400) for display. Keep them identical.
16. A WRITEFIELD row precedes the first scan and every scan whose `getWF(size).wf` differs from the previous scan's. A WAIT row precedes every SLOWSCAN at the same U,V (the `#p-wait` tooltip says "after each scan"). Size-sorted groups minimise writefield switches; mixed-size groups and multiple groups can switch back and forth.
17. Every data row has 46 fields (12 letters + 34 named columns of `PLS_FORMAT`). `PLS_COLS` has 47 entries (`No.` + 46). Change `PLS_FORMAT`, `PLS_COLS`, `TRAILING` and the three row literals together.
18. The `.js` differs from the `.pls`: `scanFrames` order, 0-based ids, no WRITEFIELD or WAIT, and it alone uses `#p-ssc`/`#p-ssc-start`.

### D. GDS handling

19. Layer 500 is a literal, not a constant: `buildDemo` 1088; `rebuildFrameOverlay` 1221, 1226; `rebuildLayerList` 1250; `buildScanFrameRecords` 1461; `stripLayer500FromCell` 1516; `stripLayer500DTFromCell` 1567; `downloadGDS` 1617; `importAutoFrames` 1699, 1714, 1721; `importSingleLayer500` 1764, 1773, 1780; strings 1709, 1731, 1734, 1769, 1789, 1792; markup 631-640, 651, 655, 656, 658, 661-670. The layer is reserved in the active cell: export removes all BOUNDARY elements on it (any datatype) before writing frames.
20. Layer 501 is an in-memory mirror of imported 500 boxes (1714, 1774), hidden by default (1718, 1777), so Custom import can still find them. It never reaches the exported `.gds`. A design that already uses 501 gets mixed with it.
21. Auto and single imports modify `lib.cells` (all cells: add 501 copies, remove 500) and `rawGDSBuffer` (active cell only). Undo does not reverse this; a second Auto import then finds nothing on 500.
22. Layer-500 boxes in referenced sub-cells are imported (flattened by `collectBoundaries`) but stay in `rawGDSBuffer`, because stripping is limited to the active cell (1502, 1559, 1595). The exported `.gds` then has both the old sub-cell boxes and the new top-cell boxes; re-importing duplicates them.
23. Overlay boxes live in `activeCell.boundaries` with `_scanOverlay:true`. `writeFullGDS` (1026) and `collectBoundaries` (1672) skip them; `flattenCell`, `rebuildLayerList`, `snapToEdge` and `getBBox` do not (so frames are ruler snap targets and are drawn by `drawCached` under `drawFrames`). Call `rebuildFrameOverlay()` after every change to `scanFrames`; it is also what marks the render cache dirty for frame changes (1228). It only cleans the current `activeCell`, so after `selectCell` the previous cell keeps stale overlay boxes.
24. Parser limits: no AREF, STRANS, MAG, ANGLE, BOX, NODE, PATHTYPE or properties; references are translation only. Rotated, mirrored, scaled or arrayed instances are drawn and imported in the wrong place or not at all. LAYER/DATATYPE are signed int16 (layers > 32767 never match; `#import-layer` allows 65535). A repeated STRNAME overwrites the earlier cell.
25. `writeFullGDS` is only used for the stub export. Routing a parsed library through it would lose everything the parser skips (item 24) plus original timestamps. Keep byte-level injection for real files.
26. `rawGDSBuffer` is `null` for the demo and after a parse error; `downloadGDS` then writes the stub. After a parse error the previous `lib` stays loaded. The ENDLIB fallback (`length-4`, 1600) assumes no trailing padding. `loadLib` does not reset `activeCell`. A file that parses to no structures leaves the previous cell active while `lib` and `rawGDSBuffer` hold the new data, and the `.gds` export then inserts frames at `length-4` of the new bytes.
27. `loadLib` keeps `scanFrames`, `frameGroups` and `undoStack`. Frames are in database units, so loading a file with a different `dbunit` moves them physically.

### E. State, undo and groups

28. Two reference schemes coexist: groups hold frame objects; selection, undo entries, drag state and row numbers use `scanFrames` indices. Every `scanFrames.splice` is followed by clearing the selection and `reassignIds()`; stored undo indices are not remapped.
29. Not undoable and not reflected in `undoStack`: row ✕ `deleteFrame` (2927), group delete (3060), group unlink (3051), Unlink all (2866), Add group (2839), merge (2872), move into group (3076), rename, Snap all (1824). An undo after one of these can hit the wrong index or nothing.
30. Undoing `deleteSelected` restores group membership by group id (2749); if the delete emptied a group, `removeFrameFromAnyGroup` already deleted it and the frames come back ungrouped.
31. `clearAllUndo` persists until used: Undo with an empty stack restores it whenever that happens (2716), replacing current frames and groups, and does not clear the selection.
32. `{type:'import',count}` undo removes the last `count` frames (2741). It is also used by `placeArrayAt` (3427). Anything that inserts frames elsewhere breaks it.
33. `importAutoFrames` replaces all frames and groups and sets `undoStack=[]` (1728-1730). It does not clear `selectedFrameIndices`, so an earlier selection now points at different frames, and Delete would remove them.
34. Group names are free text: trimmed and de-duplicated only. They go into `innerHTML` in `buildExport` (3203), after `# ` in the `.txt`, and a group named `Unlinked` collides with the sentinel (1353, 3202). Group names are not written to the `.pls` or `.gds`.
35. The sidebar shows groups in the order their first member appears in `scanFrames` (3115-3135); the export uses `frameGroups` order. There is no UI to reorder groups.
36. Hiding a group or the Unlinked set skips those frames in `drawFrames` (outline, label and centre dot on Steps 2 and 3), canvas picking and box selection, through `isFrameHidden` (2261). Hiding Unlinked also drops its frames from the sidebar list (3156); a hidden group's frames stay listed. The layer-500 overlay box of each hidden frame is still drawn by `drawCached` on all three canvases unless that 500/dt layer is hidden in the layer list. Hidden frames are still exported.
37. Move-pick (3079-3082) removes each selected frame from its group before pushing it into the target. If the target group's only frame is selected and is processed first (selection insertion order), the group is emptied and deleted. That frame and every selected frame processed after it are pushed into the detached group object and end up unlinked, and `activeGroupId` (3085) points at the deleted group.

### F. UI and DOM

38. `goStep` maps steps to `.step-page`/`.step-tab` by DOM order (1942-1947). Do not reorder or add elements with those classes.
39. Inline handlers call globals by name (2.1). Renaming a function means editing the markup too.
40. `switchPreview` builds ids `'tab-'+t` and `'preview-'+t` (1876-1877). Step tabs use `tab-1..3`, so do not name a preview type `1`, `2` or `3`.
41. The live preview listener (1886) is bound once at load to `.param-input,#p-dir,#p-frame-avg,#p-lavg` on the `input` event, and only acts on step 3. New parameters need class `param-input` (or their id added to the selector) and must exist at load.
42. The document click handler (1131) closes only what it names: every `.zoom-sens-panel`, `#array-popup` (via `closeArrayPopup`), `#import-single-panel` and `#fgo-popup`. A new popup is not closed by outside clicks until it is added there. Once it is added, a click inside the popup or on its opener must not close it. Either stop propagation on both (`onclick="event.stopPropagation()"` on the popup and `e.stopPropagation()` in the opener, as `#array-popup` 891 and `#array-btn` 3459 do), or exclude both with `contains(e.target)` (as at 1137 and 1142).
43. Keep classes the script toggles (2.3). `.inactive` (`#place-btn`) and `.open` on `#fgo-toggle-btn` have no CSS rule.
44. `setImportMode('auto')` sets `display:'block'` (1663), overriding the CSS `display:flex` with gap (255), so the auto panel loses its 8 px gap after switching tabs.
45. Keyboard: Enter on `#clear-modal` only ever cancels, and only while `#clear-modal-cancel` has class `primary` (2583); Delete/Backspace deletes the selection on any step, including steps 1 and 3 where the selection is not drawn on the canvas; there is no Ctrl+Z.
46. `selectedSEMModel` is never read. `goStep(3)` always returns to the model picker and calls `buildExport`, which calls `refreshPreviews` before any model is chosen, and only if there are frames, so preview panes can show stale text from an earlier visit.
47. Canvas colours from `LPAL`, `SCAN_COLORS` and the DRAW literals at 2061-2062 and 2138 get a two-digit alpha appended (`col+'28'` at 2038, also 2066, 2076, 2141); these must be 7-character `#rrggbb`. Canvases do not read CSS tokens.

### G. Known bugs and stale code

48. `#zsw-3` (Step 3 zoom speed, 744-753) calls `updateZoomSens(this.value)`, which does not exist; using it throws a ReferenceError. Its slider range (0.25-1.15) also differs from the others (0-100), and it has no `.zoom-sens-err-msg`. Fix: replace 744-753 with a copy of the `#zsw-2` panel body (607-617).
49. `.js` with two or more frames is broken: the separator comma lands inside `// #k` (1418, 1420).
50. `getP` uses `||default`, so 0 becomes the default: Wait 0 → 5 s and SSC start 0 → 1, despite `min="0"` on `#p-wait` (818) and `#p-ssc-start` (787). A typed 0 for Points-U/V or Avg also becomes 2500 or 2 (their `min` is 64 and 1).
51. Stale text: the hints at 620 and 2708 say a frame can be dragged by its centre. That only works with the Select tool; the older drag path is dead code.
52. Dead state: `frameOverlayBounds` (1179), `movingIdx`/`movingOrigin` and their branches (never set to a live value), `importMode`, `activePreview`, `selectedFrameIdx` (written, never read).
53. Step 3 has no zoom read-out; mousemove on step 3 writes to the hidden `#zoom-info-2` (2480).
54. Canvases ignore `devicePixelRatio` (`resizeCanvas` 1973, `resizeAll` 3300); they are blurry on HiDPI screens.
55. Wheel zoom applies one factor per event regardless of `deltaY` (2549); trackpads zoom faster than mouse wheels.
56. `mouseleave` (2535) does not reset `dragging` or `movingMultiple`, and `mouseup` is bound to each canvas only (2483). If a pan or a Select-tool multi-move is released outside the canvas, the view or the selected frames follow the cursor when it comes back, until the next mouseup on that canvas. Escape (positions restored) or a tool change (frames left where they are, no undo entry) also ends a multi-move; nothing else ends a pan.

## 2.6 Change map

Ordered edits for each recipe in 1.4, with what to re-test. "Retest" items refer to the checklist in 1.5.

### (a) SEM model or site preset

Same tool, other site:
1. `SIZE_TO_PROFILE` (STATE 1186): names = presets on the target scan manager.
2. `WF_TABLE` (1319): `label` = the tool's writefield strings; `size` = Size-U/V string.
3. Defaults: markup `value`/`selected` at 786-823 and `getP` fallbacks 1360-1378, pairwise.
4. WAIT macro path and comment (1405).
5. Help/tooltip strings 756, 785, 787, 800, 809, 818.

Retest: 1.5 steps 4-6, 9; diff `.pls` against a reference made by hand for the new site.

Second model:
1. Markup: new `.sem-model-btn` after 774 with `onclick="selectSEMModel('<id>')"`; image span gets a modifier class; CSS rule with its own `background-image`, placed after line 500 (which sets the shared one with the same specificity) or written as `.sem-model-img.sem-model-img--<id>`.
2. Markup: new panel beside `#raith-eline-panel` (778-839) with prefixed ids, present in the markup at load; CSS hidden by default, `.visible{display:flex}` like 410-411; add the copied selects' ids to the live-preview selector (1886; 2.5 item 41).
3. BUILD EXPORT: model registry, `selectSEMModel`/`showSEMModels` by id, as sketched in 1.4 (a); call `refreshPreviews` after selecting.
4. New `getP`-style reader and generators in PLS / JS / TXT GENERATORS (or a new banner section).
5. `refreshPreviews`, `switchPreview`, `downloadFile` and the `getP()` call in `downloadGDS` (1590, file name only): branch on `selectedSEMModel` or use prefixed copies; handle `null` (called from `buildExport` before selection).
6. `buildExport` enable list (3190) for new buttons. If the model has its own writefield table: its generator's lookup (the copy of 1393) and the lookup in `buildExport` (3197) use that table, and `selectSEMModel` calls `buildExport` so the Step 3 table is rebuilt for the chosen model. If the model has no writefields, branch on every writefield part of `buildExport`: the `getWF` call (3197), the writefield rows (3207-3214), the Write Field cell of the frame row (3218), the header (765) and `colspan="7"` (3203, 3211).

Retest: both models; switching models back and forth; entering step 3 repeatedly; Raith `.pls` byte-identical to reference.

### (b) Sizes and writefields

1. `DT_TO_SIZE`, `SIZE_TO_DT` (1184-1185).
2. `SIZE_TO_PROFILE` (1186-1188), `SIZE_TO_MIN_SPACING` (1190).
3. `SIZE_TO_WF_IDX` (1329); `WF_TABLE` (1319-1327). `SIZE_TO_WF_IDX` holds row indices into `WF_TABLE`, so inserting a row (say 150 µm between 100 and 200) shifts the index of every later row; recheck every entry. For a writefield equal to the frame size, change the map entries and the fallback comparison `>` in `getWF` (1334), which only applies to sizes missing from `SIZE_TO_WF_IDX`.
4. Markup: `#frame-size-sel` (631-640, `selected`, "(Layer 500/k)"), mapping table (655), `.import-dt-btn` (661-670, argument and `dt-size`), `#import-size-sel` (683-687), `#arr-min-tip` (922-931).
5. Array defaults: `arrayRowSpacing`, `arrayColSpacing` (1165), `#arr-row-sp`/`#arr-col-sp` values (904, 908); `getArrayMinSpacing` fallback `||20` (3341).
6. Demo boxes (1088) sized to `DT_TO_SIZE[4]`; fallback dt `4` (1217, 1226, 1456, 1613).
7. >10 datatypes: message 1709 and markup lists. Non-integer sizes: change `parseInt` to `parseFloat` at 1801, 2346, 2470, 3341, 3414, 3439 (2.5 item 12).

Retest: Demo auto-import; place every size; table writefield rows; `.pls` Size-U/V and WRITEFIELD labels; `.txt` step; `.gds` datatypes in KLayout; re-import round trip (1.5 step 8).

### (c) Scan layer / datatype convention

1. Add `var SCAN_LAYER=500, MIRROR_LAYER=501, FALLBACK_DT=4;` in STATE next to `DT_TO_SIZE`.
2. Replace numeric literals listed in 2.5 item 19 (500), item 20 (501), item 9 (4).
3. Update strings 1709, 1731, 1734, 1769, 1789, 1792 and markup 631-640, 651, 655, 656, 658, 661-670.
4. Keep layer ≤ 32767.
5. Different encoding (e.g. layer per size): `importAutoFrames` loop 1694-1708, `importSingleLayer500`, `rebuildFrameOverlay` (1217, 1221, 1226), `buildScanFrameRecords` (1456, 1461), the `downloadGDS` stub (1613, 1617), strip functions, `rebuildLayerList` label 1250, and `buildDemo` (1088) if the demo should use the new encoding.

Retest: auto and single import on a file using the new convention; exported `.gds` has boxes on the new layer and none left on the old one in the active cell; round trip.

### (d) `.pls` column or `.js` line

`.pls`:
1. Locate the field in the 2.4 table.
2. Edit the row literal(s) at 1397 / 1405 / 1408; for fields 21-44 on SLOWSCAN, replace `TRAILING` with an explicit run or use the `plsRow` helper from 1.4 (d).
3. New user value: a `.param-row` in `#raith-eline-panel` holding an `<input class="param-input">` (a `<select>` also needs its id added to the live-preview selector at 1886, 2.5 item 41); field in `getP`; `||` fallback (beware 0).
4. New column: append `,Name,0` to `PLS_FORMAT`, a line to `PLS_COLS`, one field to all three row literals.
5. Run the field counter in (g); expect every row = number of columns in `PLS_FORMAT`.

`.js`:
1. Fix the separator bug first:
   ```js
   // generateJS, per-frame line (1418)
   lines.push('  ['+toUm(f.cx).toFixed(6)+', '+toUm(f.cy).toFixed(6)+', '+f.sizeUm+', "'+profileFor(f.sizeUm)+'"]'
     +(i<scanFrames.length-1?',':'')+'   // #'+(i+1));
   // template (1420): replace lines.join(',\n') with lines.join('\n')
   ```
2. Edit the template string at 1420. In the source, `\\\\` yields `\\` in the output; inserted values are not escaped.
3. Optional: iterate `getOrderedFramesForExport()` and add writefield/wait steps to match the `.pls`.

Retest: (g) with ≥2 frames, evaluating the generated `.js` against a stub `App` object; 1.5 steps 4-6.

### (e) Other tool's position-list format

1. New generator `generateXYZ()` in PLS / JS / TXT GENERATORS using `getOrderedFramesForExport()`, `toUm(f.cx)`, `toUm(f.cy)`, `f.sizeUm`, `item.groupName`. Do unit/origin/y-direction conversion here.
2. Markup: `#tab-xyz` (`onclick="switchPreview('xyz')"`), `#preview-xyz` (`.preview-code`), download button, file-name `.param-row`.
3. Script: `switchPreview` array (1875), `refreshPreviews` (1881-1884), `downloadFile` branch before the `else` at 1927 (mime, name, picker `types`), `buildExport` enable list (3190), `getP` file name.
4. If replacing Raith entirely: remove or adapt `generatePLS`, `generateJS`, `PLS_FORMAT`/`PLS_COLS`/`TRAILING` (1312-1314), and `WF_TABLE`/`SIZE_TO_WF_IDX`/`getWF` with every use in `buildExport` (`getWF` call 3197, writefield rows 3207-3214, Write Field cell 3218, `colspan="7"` 3203/3211, header 765). Also adapt the `.pls`/`.js` calls in `refreshPreviews` (1882-1883), the `switchPreview` array (1875), the `downloadFile` branches (1925-1926), the `.pls`/`.js` tabs, panes and buttons (826-827, 830-831, 834-835; move the `active` class to the new default tab and pane) and their ids in the enable list (3190; a missing id throws), `SIZE_TO_PROFILE`/`profileFor`, panel text (783, 785, 756) and the model card (774, 500).
5. Keep: parser, viewer, frames, groups, importers, `rasterSort`, `getOrderedFramesForExport`, unit helpers, `.gds` injection, `.txt`.

Retest: (g) for the new generator; preview tab and download; 1.5 steps 6-9 with the target tool.

### (f) Restyle

1. `:root` tokens (13-33).
2. CSS literals at 42, 80, 82, 84, 85, 169, 200, 208, 209, 285, 308, 310, 311, 330, 331, 346, 369, 386, 387, 403, 406, 444, 460, 463, 482.
3. Canvas colours: `LPAL`, `SCAN_COLORS` (1181-1182); `'#0a0c10'` (2150, 2163, 2185); DRAW literals (2023, 2061-2062, 2085-2091, 2110-2124, 2138, 2171-2173); 2970; 3445. `LPAL`, `SCAN_COLORS` and the literals at 2061-2062 and 2138 must stay 7-character `#rrggbb`, because an alpha is appended at 2038, 2066, 2076 and 2141 (2.5 item 47). The other literals can be any canvas colour string.
4. Fonts: link 8-10, `--font`/`--mono` 25-26, canvas font strings 2023, 2069, 2119, 2142.
5. Light theme: `<meta name="color-scheme">` 6, `color-scheme` 14, second token block, canvas colours.
6. Branding: 7, 520, 521/523/525 (text only), card image 500.
7. Keep every toggled class (2.3).

Retest: visual pass on all three steps, all popups and modals, selection/placing/moving states, hidden layers, disabled buttons.

### (g) Headless check of the generators

The generators only touch the DOM through `getP` (`.value` reads), so they can be run outside a browser with a stub `document`. This extracts sections by banner, so it survives line drift. It runs with Node (`node harness.js`) or, on macOS without Node, `osascript -l JavaScript harness.js 2>&1`. osascript writes `console.log` output to stderr, hence the `2>&1`.

```sh
F=AutoSEM.html
cat > harness_stub.js <<'EOF'
var DEFAULTS={'p-ssc':'ARPsi_%04d.SSC','p-ssc-start':'1','p-lot':'','p-wafer':'','p-slot':'',
  'p-pts-u':'2500','p-pts-v':'2500','p-avg':'2','p-dir':'U','p-frame-avg':'1','p-lavg':'2','p-wait':'5',
  'p-plsfile':'scan_frames.pls','p-txtfile':'scan_frames.txt','p-jsfile':'raith_scan.js','p-gdsfile':'scan_frames.gds'};
var document={getElementById:function(id){return{value:DEFAULTS[id]||'',classList:{add:function(){},remove:function(){},toggle:function(){}},style:{}};},
  querySelectorAll:function(){return[];},addEventListener:function(){}};
EOF
awk '/^\/\/ DEMO$/,/^\/\/ ZOOM SENSITIVITY/'                  "$F" >  harness_body.js
awk '/^\/\/ STATE$/,/^\/\/ SCAN FRAME OVERLAY/'                "$F" >> harness_body.js
awk '/^\/\/ PLS \/ JS \/ TXT GENERATORS/,/^\/\/ GDS DOWNLOAD/' "$F" >> harness_body.js
awk '/^\/\/ FRAME GROUPS$/,/^\/\/ FRAME LIST$/'                "$F" >> harness_body.js
cat > harness_test.js <<'EOF'
lib=buildDemo();                                   // dbunit 1e-9: 1000 units = 1 µm
scanFrames=[{id:1,sizeUm:20,cx:24000,cy:24000},{id:2,sizeUm:5,cx:500000,cy:500000}];
createFrameGroup([scanFrames[1]],'G A');
var nCols=12+(PLS_FORMAT.split(',').length-1)/2, bad=0;
generatePLS().split('[DATA]\n')[1].split('\n').filter(Boolean).forEach(function(l){
  var n=1,q=false;for(var i=0;i<l.length;i++){if(l[i]=='"')q=!q;else if(l[i]==','&&!q)n++;}
  if(n!==nCols){bad++;console.log('BAD '+n+': '+l);}
});
console.log('columns '+nCols+', bad rows '+bad);
console.log(generateTXT());
var App={GetSysVariable:function(){return'';},GetVariable:function(){return'';},SetSysVariable:function(){},
  NewPositionlist:function(){return{Add:function(s){console.log('Pls.Add '+s);}};}};
try{eval(generateJS());console.log('.js ran');}catch(e){console.log('.js failed: '+e);}
EOF
cat harness_stub.js harness_body.js harness_test.js > harness.js
if command -v node >/dev/null; then node harness.js; else osascript -l JavaScript harness.js 2>&1; fi
```

On the unmodified file this prints `columns 46, bad rows 0`, the `.txt`, and `.js failed: TypeError …` (2.5 item 49). Extend `harness_test.js` with frames that exercise your change and compare the `.pls` text against a saved reference. To see or diff the whole `.pls`, add `console.log(generatePLS());` to `harness_test.js`. If new generator code lives under a new banner, or calls functions from another section, add an `awk '/^\/\/ BANNER$/,/^\/\/ NEXT BANNER/' "$F" >> harness_body.js` line for that section. The stub `document` returns `''` for ids missing from `DEFAULTS`, so a second panel's reader falls back to its `||` defaults unless you add its ids (for example `'q-pts-u':'2000'`). Call the second model's generator explicitly as well.

The harness loads only some sections, so a syntax error elsewhere (for example in PREVIEW / DOWNLOAD or BUILD EXPORT) goes unnoticed, and none of the Step 3 DOM wiring runs. Also check that the whole script parses:

```sh
awk '/^<script>$/{f=1;next} /^<\/script>$/{f=0} f' AutoSEM.html > script.js
node --check script.js    # or, on macOS without Node:
/System/Library/Frameworks/JavaScriptCore.framework/Versions/Current/Helpers/jsc -e "new Function(readFile('script.js'))"
```

jsc prints nothing and exits 0 when the script parses. On a syntax error it prints `Exception: SyntaxError …` and exits 3. Check the Step 3 wiring in the browser (1.5 step 4).
