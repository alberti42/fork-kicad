# Save imported documents to disk on project import

## Summary

When importing a non-KiCad project (e.g. EAGLE) via **File → Import Non-KiCad
Project… → EAGLE Project…**, the imported schematic and board were loaded into memory
and marked modified but **never written to disk**. The PCB editor was raised on top and
looked fine, while the schematic effectively **failed silently**.

This PR persists the imported documents to disk immediately after a successful import,
so a project import leaves both `<project>.kicad_sch` and `<project>.kicad_pcb` on disk.

## Problem

The project-manager import flow dispatches the schematic to eeschema and the board to
pcbnew via `MAIL_IMPORT_FILE`. Each editor loaded its document into memory and called
`SetContentModified()` / `OnModify()`, but neither wrote the result to disk. Concretely:

- The PCB editor is raised last (`kicad/import_proj.cpp`), so the board appears on top and
  "looks imported" — but it is unsaved too.
- The schematic editor sits underneath with the imported, unsaved schematic. Opening the
  schematic from the project tree calls `SCH_EDIT_FRAME::OpenProjectFiles`, where
  `is_new = !IsFileReadable( fullFileName )` is true (the file was never written), so the
  user gets **"Schematic '…' does not exist. Do you wish to create it?"**. Worse, that code
  path unloads the in-memory schematic (`SetScreen( nullptr )`) first, **discarding the
  import**.
- The only way to keep the import was to answer the save-changes prompt when **closing**
  the project, which finally writes the file. This is contrived and easy to miss.

## Fix

Persist the imported documents right after a successful import, using the exact same save
path that the close-time prompt (`AskToSaveChanges`) already invokes:

- eeschema: `SCH_EDIT_FRAME::SaveProject()`
- pcbnew: `PCB_EDIT_FRAME::SavePcbFile( GetBoard()->GetFileName() )`

This means we trigger a known-good, non-interactive save (in the import happy path) rather
than introducing new save logic. `SaveProject()` even has a pre-existing branch handling the
"file doesn't exist yet… we just imported something" case.

### Scoping (important)

`SCH_EDIT_FRAME::importFile()` is shared between the project-manager import and the
standalone **File → Import** menu. The latter can run inside an **existing** project, where an
automatic write could silently overwrite an existing file. To preserve that safe behavior:

- A new `aSaveAfterImport` flag (default `false`) gates the schematic save.
- Only the `MAIL_IMPORT_FILE` handler (the project-manager import) passes `true`.
- **File → Import keeps its previous behavior** (no automatic write).

`PCB_EDIT_FRAME::importFile()` is only ever reached from the project-manager import (there is
no PCB File → Import menu), so no flag is needed there.

### Guards

Both saves are guarded by:

- **import success** — never save an empty schematic/board after a failed or aborted import
  (the importer's `catch` blocks leave the success flag false), and
- **`!Prj().IsNullProject()`** — avoid a spurious *Save As* dialog in standalone mode with no
  real project.

In the project-manager flow the target project is freshly created (the import even steers the
user to a clean directory when the chosen one is not empty), so the auto-save does not clobber
pre-existing files.

## Behavior matrix

| Entry point | Schematic | PCB |
|---|---|---|
| Project manager → Import Non-KiCad Project | auto-saved to disk | auto-saved to disk |
| eeschema → File → Import | not auto-saved (unchanged) | n/a (no such menu) |

## Files changed

- `eeschema/sch_edit_frame.h` — add `aSaveAfterImport` parameter to `importFile()`.
- `eeschema/files-io.cpp` — track import success; save when requested, guarded by a real project.
- `eeschema/cross-probing.cpp` — `MAIL_IMPORT_FILE` handler requests the save.
- `pcbnew/files.cpp` — save the imported board after a successful `OpenProjectFiles`.

## Testing

Manual:

1. Import an EAGLE project (`.sch` + `.brd`) via **Import Non-KiCad Project… → EAGLE Project…**.
2. Confirm `<project>.kicad_sch` and `<project>.kicad_pcb` exist in the new project directory
   immediately, without closing the project.
3. Open the schematic from the project tree — it opens directly, with no "does not exist"
   prompt.
4. **File → Import** of a single schematic into an existing project — confirm it is **not**
   auto-written (unchanged behavior); the user still saves manually.
5. Standalone (no project) import — confirm no spurious *Save As* dialog and no empty-file
   write on a failed/cancelled import.

## Notes

- This addresses the project-manager import path for all non-KiCad formats routed through
  `importFile`, not just EAGLE.
- A detailed code-path analysis is available in `EAGLE_SCHEMATIC_IMPORT_REPORT.md`.
