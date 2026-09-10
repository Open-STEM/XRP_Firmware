# Firmware Loader Layout & Authoring Guide

This document describes the on-disk layout under `public/firmware-loader/` and how
to author boards, projects, and library/firmware versions. It is the source of
truth for the firmware-install wizard.

## Goals

- **Shared, versioned XRPLib releases.** One library store, referenced by many
  projects. A release bundles the library plus the standard support files.
- **Per-board MicroPython firmware versions.** Each board type keeps its own
  MicroPython firmware versions, separate from the MicroPython *project*.
- **Self-describing projects.** A project is described entirely by its
  `project.json`: version references plus an optional inline list of extras.
- **Specialized firmware support.** A project can ship its own UF2 instead of a
  shared MicroPython version.

## Directory layout

```
public/firmware-loader/
  index.json                          # Board selector (top level)
  wizard-assets.json                  # Image map for the install wizard
  images/                             # Board/project artwork
  spec.md                             # This file
  boards/
    <board>.json                      # Project selector for one board (e.g. xrp-2350.json)

    XRPLib/                            # Shared, versioned XRPLib release store
      index.json                      # Registry of XRPLib releases (friendly id -> dir/version)
      <version>/                      # One XRPLib release (e.g. 2.2.1/)
        XRPLib/  ble/  phew/  XRPExamples/   # The release's bundled files
        files.json                    # GENERATED — do not edit by hand

    <board>/                           # One board type (e.g. xrp-2350/, xrp-beta/)
      micropython-firmware/           # MicroPython firmware version store for this board
        index.json                    # Registry of MicroPython firmware versions
        <version>/                    # One MicroPython version (e.g. 1.28.0/)
          firmware.uf2                # The flashable firmware
      micropython/                    # The "MicroPython" project
        project.json
        main.py                       # Project-specific extras (listed in project.json `files`)
      <project>/                      # Any other project (agxrp, sparkfun-red-vision, wpilib, …)
        project.json
        firmware.uf2                  # Optional: a project-specific/specialized firmware
        AgXRPLib/ …                   # Project-specific extras (listed in project.json `files`)
```

Note the separation: the MicroPython **firmware** versions live in
`micropython-firmware/`, while the MicroPython **project** the user selects lives
in `micropython/`.

## Manifests

### `index.json` (board selector) and `boards/<board>.json` (project selector)

Navigation only. Each lists `boards` and/or `projects` of display cards:

```json
{
  "title": "Choose your board",
  "boards": [
    { "id": "xrp-2350", "name": "XRP (RP2350)", "description": "…",
      "image": "/firmware-loader/images/board-xrp-2350.jpg",
      "nextLevel": "boards/xrp-2350.json" }
  ],
  "projects": [
    { "id": "micropython", "name": "MicroPython", "description": "…",
      "image": "/firmware-loader/images/project-micropython.svg",
      "nextLevel": "boards/xrp-2350/micropython/project.json" }
  ]
}
```

`nextLevel` points either at another selector (`boards/<board>.json`) or at a
project (`…/project.json`).

### `boards/XRPLib/index.json` — XRPLib release registry

```json
{
  "title": "XRPLib versions",
  "versions": [
    { "id": "xrplib-2.2.1", "name": "XRPLib 2.2.1", "dir": "2.2.1", "version": "2.2.1" },
    { "id": "xrplib-2.1.3", "name": "XRPLib 2.1.3", "dir": "2.1.3", "version": "2.1.3" }
  ]
}
```

- `id` — the friendly id projects reference via `xrplib`. **Tie it to the version**
  (e.g. `xrplib-2.2.1`); never reuse an id for changed files.
- `dir` — the release directory under `boards/XRPLib/`.
- `version` — written to `/lib/XRPLib/version.py` on the robot and compared against
  the on-device version to decide whether an update is available.

Each XRPLib release is **atomic**: if any file changes, it is a new version with a
new directory and a new id. There is no per-file override mechanism.

A release **bundles** `XRPLib/`, `ble/`, `phew/`, and `XRPExamples/`. These are all
copied to the robot when a project references the release (see the device-path
convention below).

A few files ship inside a release but are **not** installed on every board — the
NanoXRP's `buzzer.py` and `buzzer_examples.py`, since it is the only board with a
buzzer. Each release declares its own list in `board-only.json` (below); the
generators read that and leave those files out of `files.json`, and a board that
wants one asks for it by name via `xrplibFiles`. Never hand-edit `files.json` to
achieve this — it is regenerated on every build and your edit will be silently
reverted.

### `boards/<board>/micropython-firmware/index.json` — MicroPython firmware registry

```json
{
  "title": "MicroPython versions (XRP / RP2350)",
  "versions": [
    { "id": "micropython-1.28.0", "name": "MicroPython 1.28.0",
      "dir": "1.28.0", "version": "1.28.0", "uf2": "firmware.uf2" }
  ]
}
```

- `uf2` — firmware filename inside the version directory (defaults to
  `firmware.uf2` if omitted).
- A board's "canonical" MicroPython version is whatever its
  `micropython/project.json` references; that is what update checks compare
  against.

### `project.json` — a project

A project declares **what** to install, by reference, plus any project-specific
extras inline. It never has a separate `files.json`.

```json
{
  "title": "AgXRP",
  "description": "…",
  "micropython": "micropython-1.28.0",   // friendly id from micropython-firmware/index.json
  "xrplib": "xrplib-2.1.3",              // friendly id from boards/XRPLib/index.json
  "files": [                             // optional project-specific extras
    ["/lib/AgXRPLib/agxrp_controller.py", "AgXRPLib/agxrp_controller.py"],
    ["/main.py", "main.py"]
  ],
  "xrplibFiles": [                       // optional board-only files from the release above
    ["/lib/XRPLib/buzzer.py", "XRPLib/buzzer.py"]
  ]
}
```

Fields:

- `title` (required), `description` (optional) — shown in the wizard.
- Firmware source — exactly one of:
  - `micropython` — a friendly id resolved against the board's
    `micropython-firmware/index.json`, **or**
  - `uf2` — a path (relative to the project dir) to a project-specific,
    specialized firmware shipped with the project.
- `xrplib` (optional) — a friendly id from `boards/XRPLib/index.json`. When
  present, that whole XRPLib release is copied to the robot.
- `files` (optional) — `[deviceDestination, sourcePathRelativeToProjectDir]`
  pairs for files that belong only to this project (e.g. a project-specific
  `main.py`, or a library like `AgXRPLib/` that is not part of an XRPLib release).
- `xrplibFiles` (optional, requires `xrplib`) — `[deviceDestination,
  sourcePathRelativeToReleaseDir]` pairs naming files from that release's
  `board-only.json` that this board does want. Because the source is relative to the *resolved*
  release directory, bumping `xrplib` carries these files along; never write a
  `../../XRPLib/<version>/…` path in `files`, which silently pins the old
  version when the release is bumped.

Examples:

| Project | `micropython` | `uf2` | `xrplib` | `files` |
|---|---|---|---|---|
| MicroPython | `micropython-1.28.0` | — | `xrplib-2.2.1` | `main.py` |
| MicroPython (NanoXRP) | `micropython-1.28.0` | — | `xrplib-2.2.1` | `main.py` + `xrplibFiles`: buzzer |
| AgXRP | `micropython-1.28.0` | — | `xrplib-2.1.3` | `AgXRPLib/…`, `main.py` |
| SparkFun Red Vision | — | `firmware.uf2` | `xrplib-2.2.1` | — |
| WPILib | — | `firmware.uf2` | — | — |

A project with neither `micropython` nor `uf2` is treated as "not yet available".

## Device-path convention

Both the generated XRPLib release manifest and the project `files` entries use
the same destination paths. The convention (used by the generator and recommended
for hand-written `files` entries) maps the first path segment of a source file:

| Source folder | Device destination |
|---|---|
| `XRPLib/…`    | `/lib/XRPLib/…`   |
| `AgXRPLib/…`  | `/lib/AgXRPLib/…` |
| `ble/…`       | `/lib/ble/…`      |
| `phew/…`      | `/lib/phew/…`     |
| `XRPExamples/…` | `/XRPExamples/…` |
| `main.py` (and other top-level files) | `/main.py` |

Before copying, the wizard wipes the well-known library directories that appear in
the combined manifest (`/lib/XRPLib`, `/lib/AgXRPLib`, `/lib/ble`, `/lib/phew`) so
removed files don't linger.

### `boards/XRPLib/<version>/board-only.json` — board-specific files in a release

```json
{
  "description": "…",
  "boardOnly": ["XRPLib/buzzer.py", "XRPExamples/buzzer_examples.py"]
}
```

Paths are relative to the release directory. Files listed here are omitted from
the generated `files.json`, so they are not installed on every robot; a board
opts in through `xrplibFiles` in its `project.json`.

The list is **owned by XRP_MicroPython** (`board-only.json` at that repo's root)
and copied into each release by its publish workflow, so neither this repo nor
XRPWeb hardcodes any filenames. A release without the file simply has no
board-only files. It is never copied to the robot.

## `files.json` — generated for XRPLib releases only

A static web server cannot list directories at runtime, so the XRPLib release file
lists are generated at build time by `scripts/gen-firmware-manifests.mjs`:

- `boards/XRPLib/<version>/files.json` — an array of
  `[deviceDestination, sourceRelativePath]` pairs covering every file in the
  release. `version.py` is excluded; it is synthesized on-device from the registry
  `version`.

Projects do **not** get a generated `files.json` — list any extras inline in
`project.json`'s `files` field instead.

The generator runs automatically via the `predev`/`prebuild` npm hooks, or
manually:

```bash
npm run gen:firmware-manifests
```

## How to…

### Add a new XRPLib release
1. Create `boards/XRPLib/<version>/` with `XRPLib/`, `ble/`, `phew/`,
   `XRPExamples/` subfolders and drop the files in.
2. Add an entry to `boards/XRPLib/index.json` with a version-tied `id`.
3. Point **every** board's `micropython/project.json` at the new `id` via its
   `xrplib` field — including `xrp-nano`, which is easy to miss.
4. Run `npm run gen:firmware-manifests` and commit the regenerated `files.json`.
   Boards needing board-only files (the NanoXRP's buzzer) pick them up
   automatically through `xrplibFiles`; no per-version paths to update.

### Add a new MicroPython firmware for a board
1. Create `boards/<board>/micropython-firmware/<version>/firmware.uf2`.
2. Add an entry to that board's `micropython-firmware/index.json`.
3. Point projects at the new `id` via their `micropython` field.

### Add a new project
1. Create `boards/<board>/<project>/project.json`.
2. Reference a `micropython` id or add a local `uf2`; optionally add `xrplib`.
3. Drop any project-specific files into the project dir and list them in `files`.
4. Add a project card with `nextLevel` pointing at the new `project.json` in the
   board's `boards/<board>.json`.

### Ship a project with specialized firmware
Place the `.uf2` in the project dir and set `"uf2": "firmware.uf2"` (omit
`micropython`). Add `xrplib` only if the project also needs an XRPLib release.
