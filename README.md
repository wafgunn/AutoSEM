# AutoSEM

AutoSEM sets up unattended SEM imaging on a Raith eLine Plus (Raith NanoSuite) from a GDSII layout. You place scanframes on the layout, and AutoSEM writes a position list (.pls), a Raith script (.js), a lookup table (.txt) and an annotated GDS. It is a single HTML file that runs in the browser and uploads nothing. Its only network requests are for the Inter font from Google Fonts; offline it uses a system font.

## Quick start

1. Download `AutoSEM.html` and open it in a recent Chrome, Edge, Firefox or Safari.
2. Step 1: click Open GDS, drop a `.gds` file onto the window, or click Demo. If the cell you will align to (usually the top cell) is not highlighted, click it.
3. Step 2: place scanframes or import them from the layout.
4. Step 3: choose Raith eLine Plus, check the parameters and download the files.

Chrome and Edge open a save dialog. Firefox and Safari show a save link instead, and Safari always saves to Downloads.

## What it does

**Step 1, GDSII viewer.** Click a cell to make it active, and click a layer/datatype to hide or show it. Scanframes are placed, imported and exported in the active cell's coordinates, and keep those coordinates if you change cell or file.

**Step 2, scanframes.**

- Scanframes are square, in ten sizes from 1 to 100 µm. Each size maps to a datatype (0–9) on layer 500 and to a scan profile such as `20um_bss8nm`.
- "+ Single frame", then click the canvas. "+ Scan array": set rows and columns (up to 49) and the spacing in µm, click "Place array", then click where the centre of the bottom-left frame should go. The spacing defaults to the beam step (8 nm for 20 µm), so set it to at least the frame size to avoid overlap.
- "Import all 500/\* layers" puts a frame of the matching size at the centre of every box on 500/0–500/9. It replaces all frames and groups and clears the undo history. "Import one 500/\* layer" adds one datatype, and "Import box centres" (Custom Layer/DT tab) adds frames of a chosen size at the bounding-box centre of every boundary on a given layer/datatype.
- Each import or array forms a group. "Add group" groups the selected frames, and "Group options" moves frames into a group, merges groups or unlinks them all. Right-click a group header to rename it. Group names go into the .txt.
- Frames snap to 1 nm when placed or moved, and "Snap all" snaps every frame. "Undo last" reverses placements, moves, imports, arrays and Delete-key deletions, and "Clear all" can be undone once. The ✕ buttons, "Snap all" and group changes cannot be undone.

**Step 3, export.** Frames are exported group by group in creation order, smallest first within a group, then ungrouped frames. The No. column in the Step 3 table is the scan number in the .pls and .txt. The # labels on the canvas and in the Step 2 list follow placement order, so they can differ. The SSC prefix and start index are used only in the .js.

| File | Contents |
|---|---|
| `scan_frames.pls` | Per frame: a WRITEFIELD row when the writefield changes, a MACRO row running `%UserRoot%SCRIPT\WAIT.JS`, and a SLOWSCAN row with the centre (U, V in mm), scan profile, size, pixels and averaging. |
| `scan_frames.txt` | A header line, then per frame the scan number (= .pls ID), x and y (µm), size (µm), beam step (µm) and group name (`Unlinked` if ungrouped). |
| `raith_scan.js` | Sets the scan-manager file name (SSC) and start index, then builds a position list of SLOWSCAN entries. It does not save or run the list. |
| `scan_frames.gds` | The original layout with the active cell's layer-500 boxes replaced by the scanframes (scanframes only for the Demo). |

## Using the export on the eLine Plus

U and V are frame centres in mm from the active cell's origin, with no offset. Before running the .pls:

1. Put a `WAIT.JS` in `%UserRoot%SCRIPT` on the tool PC; AutoSEM does not supply one. The .pls runs it before each scan, and the Wait value only appears in the MACRO row's comment (e.g. `wait_5s`).
2. Check that the scan profiles and writefields named in the .pls exist on the tool.
3. Set the scan-manager file name (SSC) and start index on the tool.
4. Do the three-point alignment on markers from the same cell, focusing at each one.
5. Run a few positions first.

The .pls lists scans in scan-number order, so if images are numbered consecutively the .txt gives each image's position, size and group.

## Mouse and keyboard

| Input | Action |
|---|---|
| Left-drag (Pan), middle-drag (any tool) | Pan |
| Right-click | Centre the view there |
| Click a frame's centre or edge (Select) | Select; Shift-click adds or removes; drag on empty canvas to box select |
| Drag a frame (Select, Step 2) | Move it, or the whole selection |
| Two clicks (Ruler) | Measure, snapping to vertices and edges; Shift keeps it horizontal or vertical |
| Drag (Zoom box) | Zoom in; drag up-left to zoom out |
| Shift-click / Ctrl- or Cmd-click in the frame list | Select a range / toggle |
| Delete or Backspace | Delete selected frames |
| Esc | Cancel placing and return to Pan, or cancel a group move/merge; in Select, the first press clears the selection |
| Enter | Close a message, or save from the download box |

## Known limitations

- The .js works only for a single frame; use the .pls.
- Frames hidden with an eye button in the frame list are still exported.
- A Wait of 0 s is written as 5 s, and an SSC start index of 0 as 1.
- The Step 3 Zoom speed panel has no effect; the speed set in Step 1 or 2 applies to all steps.
- Importing from layer 500 moves the source boxes to a hidden layer 501. To import them again after "Undo last" or "Clear all", drop the GDS onto the window again.
- Arrayed (AREF) references are skipped, and rotation, mirroring and magnification of references are ignored, so geometry inside them (and frames imported from it) is missing or misplaced.

## Adapting AutoSEM to another SEM or facility

Frame sizes, scan profile names, writefields, the .pls layout and default parameters are set inside `AutoSEM.html`, several in more than one place. [ExtendingAutoSEM.md](ExtendingAutoSEM.md) explains how the file is organised and what to change for other tools or settings.

## Citation

If you use AutoSEM, please cite:

Gunn, W. et al., Converting a superseded Raith electron beam pattern generator into an unattended Critical Dimension SEM (in preparation).

## Licence

MIT. See [LICENSE](LICENSE).
