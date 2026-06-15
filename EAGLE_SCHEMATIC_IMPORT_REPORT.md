# EAGLE Project Import — Code Map & Silent-Failure Analysis

**Context:** This is a fork of `kicad-source-mirror`. The goal is to debug why importing an
EAGLE project via **"Import Non-KiCad Project..." → "EAGLE Project..."** imports the **PCB
successfully** but the **schematic fails silently** (no schematic appears, no error dialog,
no log message).

This document maps the complete code path and ranks the realistic causes of the silent
schematic failure, with concrete file/line references and suggested debugging steps.

---

## 1. High-level flow

```
KiCad project manager (kicad/)
  menu "EAGLE Project..."  ──▶ OnImportEagleFiles
                                   │
                                   ▼
                            ImportNonKiCadProject   (picks source file + dest dir)
                                   │
                                   ▼
                            IMPORT_PROJ_HELPER::ImportFiles
                                   │  (EAGLE has no special handler → generic path)
                     ┌─────────────┴──────────────┐
                     ▼                             ▼
        ImportIndividualFile(SCHEMATIC_T)   ImportIndividualFile(PCB_T)
                     │                             │
                     ▼                             ▼
                 doImport                       doImport
                     │   Kiway ExpressMail          │   Kiway ExpressMail
                     │   MAIL_IMPORT_FILE           │   MAIL_IMPORT_FILE
                     ▼                             ▼
       eeschema (FRAME_SCH)             pcbnew (FRAME_PCB_EDITOR)
       cross-probing.cpp handler        cross-probing.cpp handler
                     │                             │
                     ▼                             ▼
       SCH_EDIT_FRAME::importFile        PCB_EDIT_FRAME::importFile / OpenProjectFiles
                     │                             │
                     ▼                             ▼
       SCH_IO_EAGLE::LoadSchematicFile   PCB_IO_EAGLE::LoadBoard
```

**Key point:** the schematic and PCB imports are *entirely separate* code paths (different
frames, different plugins, different Kiway messages). The PCB working tells us nothing about
the schematic path other than "the file was found and dispatched correctly for the board."

---

## 2. Menu wiring (project-manager app, `kicad/`)

| Component | File | Notes |
|-----------|------|-------|
| Menu item "EAGLE Project..." | `kicad/menubar.cpp` | Uses ID `ID_IMPORT_EAGLE_PROJECT` |
| Menu ID enum | `kicad/kicad_id.h` | `ID_IMPORT_EAGLE_PROJECT` |
| Event table entry | `kicad/kicad_manager_frame.cpp:121` | `EVT_MENU( ID_IMPORT_EAGLE_PROJECT, KICAD_MANAGER_FRAME::OnImportEagleFiles )` |
| Handler | `kicad/import_project.cpp:195` | See below |

```cpp
// kicad/import_project.cpp:195
void KICAD_MANAGER_FRAME::OnImportEagleFiles( wxCommandEvent& event )
{
    ImportNonKiCadProject( _( "Import Eagle Project Files" ), FILEEXT::EagleFilesWildcard(),
                           { "sch" },            // schematic extension(s)
                           { "brd" },            // pcb extension(s)
                           SCH_IO_MGR::SCH_EAGLE, PCB_IO_MGR::EAGLE );
}
```

The schematic extension is hard-coded to `sch`, the board to `brd`.

---

## 3. Orchestration (`kicad/import_project.cpp`, `kicad/import_proj.cpp`)

### 3.1 `ImportNonKiCadProject` — `kicad/import_project.cpp:48`
- Opens a file dialog (`EagleFilesWildcard`) and stores the chosen file in
  `importProj.m_InputFile` (line 87).
- Asks for a destination directory and creates the KiCad project.
- For EAGLE (not PADS) it calls `importProj.ImportFiles( aSchFileType, aPcbFileType )`
  (line 171).

### 3.2 `IMPORT_PROJ_HELPER::ImportFiles` — `kicad/import_proj.cpp:607`
EAGLE matches none of the special handlers (EasyEDA Pro, Altium, gEDA), so control falls
through to the generic tail at **lines 674–675**:

```cpp
// kicad/import_proj.cpp:674
ImportIndividualFile( SCHEMATIC_T, aImportedSchFileType );   // SCH_IO_MGR::SCH_EAGLE
ImportIndividualFile( PCB_T,       aImportedPcbFileType );   // PCB_IO_MGR::EAGLE
```

### 3.3 `IMPORT_PROJ_HELPER::ImportIndividualFile` — `kicad/import_proj.cpp:92`
This is the critical function. For the schematic case (`SCHEMATIC_T`):
- `neededExts = m_schExtenstions` = `{ "sch" }`, `frame_type = FRAME_SCH`.
- For each extension, it builds a **candidate from `m_InputFile` with the extension swapped**
  (lines 120–121), i.e. `<selected-file-basename>.sch` in the **same directory** as the file
  the user selected.
- If that candidate exists, it copies it into the new project dir and remembers it.
- **Line 144–145:** if no candidate file was found, it **silently returns**.

```cpp
// kicad/import_proj.cpp:115
for( wxString ext : neededExts )
{
    if( ext == wxS( "INPUT" ) )
        ext = m_InputFile.GetExt();

    wxFileName candidate = m_InputFile;
    candidate.SetExt( ext );              // <basename>.sch next to the picked file

    if( !candidate.FileExists() )
        continue;
    ...
}

if( appImportFile.empty() )
    return;                               // <-- SILENT: no matching .sch found

doImport( appImportFile, frame_type, aImportedFileType );
```

### 3.4 `IMPORT_PROJ_HELPER::doImport` — `kicad/import_proj.cpp:151`
Sends the import request as a Kiway ExpressMail message to the target frame:

```cpp
// kicad/import_proj.cpp:151
void IMPORT_PROJ_HELPER::doImport( const wxString& aFile, FRAME_T aFrameType, int aImportedFileType )
{
    if( KIWAY_PLAYER* frame = m_frame->Kiway().Player( aFrameType, true ) )
    {
        std::stringstream ss;
        ss << aImportedFileType << '\n' << TO_UTF8( aFile );
        for( const auto& [key, value] : m_properties )
            ss << '\n' << key << '\n' << value.wx_str();

        std::string packet = ss.str();
        frame->Kiway().ExpressMail( aFrameType, MAIL_IMPORT_FILE, packet, m_frame );
        ...
    }
}
```

Payload format: `"<fileTypeInt>\n<absolutePath>[\n<key>\n<value>]..."`.

---

## 4. Receiving side — schematic (`eeschema/`)

### 4.1 Mail handler — `eeschema/cross-probing.cpp:1091`
Parses the payload and calls `importFile`:

```cpp
// eeschema/cross-probing.cpp:1091
case MAIL_IMPORT_FILE:
{
    std::stringstream ss( payload );
    char delim = '\n';

    std::string formatStr;
    wxCHECK( std::getline( ss, formatStr, delim ), /* void */ );   // silent void-return if missing

    std::string fnameStr;
    wxCHECK( std::getline( ss, fnameStr, delim ), /* void */ );    // silent void-return if missing

    int importFormat;
    try { importFormat = std::stoi( formatStr ); }
    catch( std::invalid_argument& ) { wxFAIL; importFormat = -1; }

    std::map<std::string, UTF8> props;
    /* ...parse key/value pairs... */

    if( importFormat >= 0 )
        importFile( fnameStr, importFormat, props.empty() ? nullptr : &props );

    break;
}
```

### 4.2 `SCH_EDIT_FRAME::importFile` — `eeschema/files-io.cpp:1485`
Creates the EAGLE plugin and invokes it. Relevant excerpts:

```cpp
// eeschema/files-io.cpp:1503  (EAGLE is one of the handled types)
case SCH_IO_MGR::SCH_EAGLE:
{
    wxCHECK_MSG( aFileName.IsEmpty() || filename.IsAbsolute(), false,
                 wxS( "Import schematic: path is not absolute!" ) );

    try
    {
        IO_RELEASER<SCH_IO>  pi( SCH_IO_MGR::FindPlugin( fileType ) );
        DIALOG_HTML_REPORTER errorReporter( this );
        WX_PROGRESS_REPORTER progressReporter( this, _( "Import Schematic" ), 1, PR_CAN_ABORT );

        // *** Reporter gating — see Silent-Failure cause #2 ***
        if( eeconfig()->m_System.show_import_issues )
            pi->SetReporter( errorReporter.m_Reporter );
        else
            pi->SetReporter( &NULL_REPORTER::GetInstance() );      // issues swallowed

        pi->SetProgressReporter( &progressReporter );

        SCH_SHEET* loadedSheet = pi->LoadSchematicFile( aFileName, newSchematic.get(),
                                                        nullptr, aProperties );
        SetSchematic( newSchematic.release() );

        if( loadedSheet )
        {
            /* ... build top-level sheets, flush errorReporter if it HasMessage(), set up
               screens, RecalculateConnections, TestDanglingEnds ... */
        }
        else
        {
            CreateDefaultScreens();                                // empty schematic, no message
        }
    }
    catch( const IO_ERROR& ioe )
    {
        CreateDefaultScreens();
        DisplayErrorMessage( this, wxString::Format( _( "Error loading schematic '%s'." ),
                             aFileName ), ioe.What() );            // user DOES see this
    }
    catch( const std::exception& exc )
    {
        CreateDefaultScreens();
        DisplayErrorMessage( this, wxString::Format( _( "Unhandled exception occurred loading "
                             "schematic '%s'." ), aFileName ), exc.what() );  // user DOES see this
    }
    ...
}
```

The function always returns `true` (line 1671) regardless of outcome.

### 4.3 EAGLE schematic plugin
- Class: `SCH_IO_EAGLE` — `eeschema/sch_io/eagle/sch_io_eagle.h` (decl ~line 79–308)
- Entry: `SCH_IO_EAGLE::LoadSchematicFile()` — `eeschema/sch_io/eagle/sch_io_eagle.cpp:349`
- Signature:
  ```cpp
  SCH_SHEET* LoadSchematicFile( const wxString& aFileName, SCHEMATIC* aSchematic,
                                SCH_SHEET* aAppendToMe = nullptr,
                                const std::map<std::string, UTF8>* aProperties = nullptr ) override;
  ```

---

## 5. Receiving side — PCB (for contrast; this path works)

- Mail handler: `pcbnew/cross-probing.cpp` (`MAIL_IMPORT_FILE`) → `importFile`.
- `PCB_EDIT_FRAME::importFile` — `pcbnew/files.cpp:1251` routes to `OpenProjectFiles()`
  with `KICTL_NONKICAD_ONLY | KICTL_IMPORT_LIB`.
- `PCB_EDIT_FRAME::OpenProjectFiles` — `pcbnew/files.cpp:474`; loader invoked ~line 703,
  exceptions caught ~lines 711–737.
- Plugin: `PCB_IO_EAGLE::LoadBoard` — `pcbnew/pcb_io/eagle/pcb_io_eagle.cpp:331`.
- Note `pcbnew/files.cpp:659`: PCB loader's reporter is also gated by
  `config()->m_System.show_import_issues`.

---

## 6. The `show_import_issues` setting

| Location | Purpose |
|----------|---------|
| `include/settings/app_settings.h:206` | `bool show_import_issues;` |
| `common/settings/app_settings.cpp:246` | Param `"system.show_import_issues"`, **default `true`** |
| `eeschema/files-io.cpp:915` | Checkbox `FILEDLG_IMPORT_NON_KICAD` shown in eeschema's *own* File→Import dialog |
| `eeschema/files-io.cpp:923` | Persists the checkbox value back to the setting |
| `eeschema/files-io.cpp:1532` | Gates the schematic import reporter |
| `pcbnew/files.cpp:194/207/659` | Same setting on the PCB side |

**Important:** The project-manager import path ("Import Non-KiCad Project...") does **not**
present this checkbox. It uses whatever value is currently persisted. The default is `true`,
but if it was ever set to `false` (e.g. via a prior eeschema File→Import), reported issues
will be silently dropped on every subsequent project import.

---

## 7. Silent-failure causes, ranked

### Cause #1 (most likely): the `.sch` file is never found
`kicad/import_proj.cpp:115-145`. `ImportIndividualFile` only looks for
`<selected-file-basename>.sch` in the **same directory** as the file the user selected. If
the EAGLE schematic:
- has a different base name than the selected file, or
- lives in a different directory,

then no candidate is found and the function `return`s at line 144 **with no dialog, no log,
and no Kiway message ever sent**. This matches "fails silently" exactly, and is consistent
with the PCB working (the `.brd` presumably *does* match the selected base name).

> EAGLE projects normally share a base name between `.sch` and `.brd`, but this is not
> guaranteed — multi-sheet designs, renamed files, or selecting the `.brd` while the `.sch`
> has a different stem all break the assumption.

### Cause #2: import issues routed to `NULL_REPORTER`
`eeschema/files-io.cpp:1532-1535`. The EAGLE schematic plugin reports many problems
(missing/unsupported symbols, parse anomalies) through a *reporter* rather than by throwing.
If `show_import_issues` is `false`, the reporter is `NULL_REPORTER` and the user sees nothing
even though the import partially or fully failed. Default is `true`, but verify the persisted
value.

### Cause #3: `LoadSchematicFile` returns `nullptr`
`eeschema/files-io.cpp:1591-1594`. If the plugin returns no sheet (without throwing), the
code calls `CreateDefaultScreens()` and returns `true` — an empty schematic with no message.

### Cause #4: malformed Kiway payload
`eeschema/cross-probing.cpp:1099,1102`. `wxCHECK(...)` returns void silently if the payload
can't be parsed. Unlikely here, but it is a silent exit.

> Note: genuine `IO_ERROR` / `std::exception` from the plugin **are** surfaced via
> `DisplayErrorMessage` (`files-io.cpp:1596-1620`). A *truly silent* failure therefore points
> away from "hard parse exception" and toward causes #1, #2, or #3.

---

## 8. Suggested debugging steps (in order)

1. **Confirm the `.sch` is found.** Add a temporary `wxLogMessage` (or breakpoint) at
   `kicad/import_proj.cpp:140-147` to check whether `appImportFile` is empty and whether
   `doImport` is reached. If it's empty → Cause #1; the fix is about how the schematic file
   is located (base-name / directory assumption). Check what file the user selected vs. the
   actual `.sch` name on disk.

2. **Confirm the mail arrives in eeschema.** Breakpoint `eeschema/cross-probing.cpp:1131`
   (`importFile` call) to verify `importFormat == SCH_IO_MGR::SCH_EAGLE` and the path is
   correct.

3. **Inspect the plugin result.** Breakpoint `eeschema/files-io.cpp:1539` and inspect
   `loadedSheet` (nullptr → Cause #3) and `errorReporter.m_Reporter->HasMessage()` after the
   call. Temporarily force `show_import_issues` true (or set the reporter unconditionally) to
   surface suppressed issues (Cause #2).

4. **If `LoadSchematicFile` itself misbehaves**, step into
   `eeschema/sch_io/eagle/sch_io_eagle.cpp:349` (`SCH_IO_EAGLE::LoadSchematicFile`) with the
   actual file to see where it bails / what it reports.

---

## 9. File/line quick reference

| What | Path:line |
|------|-----------|
| Menu handler | `kicad/import_project.cpp:195` |
| `ImportNonKiCadProject` | `kicad/import_project.cpp:48` |
| `ImportFiles` (EAGLE tail) | `kicad/import_proj.cpp:607` (tail at `674-675`) |
| `ImportIndividualFile` (silent return) | `kicad/import_proj.cpp:92` (return at `144`) |
| `doImport` (Kiway mail) | `kicad/import_proj.cpp:151` |
| Eeschema mail handler | `eeschema/cross-probing.cpp:1091` |
| `SCH_EDIT_FRAME::importFile` | `eeschema/files-io.cpp:1485` |
| Reporter gating | `eeschema/files-io.cpp:1532` |
| `loadedSheet` null → default screens | `eeschema/files-io.cpp:1591` |
| Exception dialogs | `eeschema/files-io.cpp:1596-1620` |
| EAGLE SCH plugin header | `eeschema/sch_io/eagle/sch_io_eagle.h` |
| EAGLE SCH plugin entry | `eeschema/sch_io/eagle/sch_io_eagle.cpp:349` |
| PCB mail handler | `pcbnew/cross-probing.cpp` (`MAIL_IMPORT_FILE`) |
| PCB import | `pcbnew/files.cpp:1251` / `OpenProjectFiles` at `474` |
| EAGLE PCB plugin entry | `pcbnew/pcb_io/eagle/pcb_io_eagle.cpp:331` |
| `show_import_issues` default | `common/settings/app_settings.cpp:246` (default `true`) |

---

*Generated for handoff. Line numbers reflect the current state of this fork's working tree;
re-verify after any edits.*
