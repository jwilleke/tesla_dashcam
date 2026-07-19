# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`tesla_dashcam` is a Python CLI that merges the per-minute, per-camera MP4 files Tesla
saves for Dashcam/Sentry events into a single movie. Tesla stores up to six cameras
(front, rear, left/right repeaters, left/right pillars), one file per camera per minute,
grouped into event folders. The tool combines these into one video per event (and can
concatenate events) using **ffmpeg** as the encoding engine.

## Commands

```bash
# Run from source
python -m tesla_dashcam <source_folder> --output <dest_folder>
# or after `pip install -e .`
tesla_dashcam <source_folder> --output <dest_folder>

# Install deps
pip install -r requirements.txt          # runtime
pip install -r requirements_dev.txt       # adds pytest, doc8

# Tests
pytest                                    # all tests
pytest tests/test_tesla_dashcam.py::TestDiamond           # one test class
pytest tests/test_tesla_dashcam.py::TestDiamond::test_x   # one test

# Build standalone binaries (bundles ffmpeg in via PyInstaller)
./bundles/MacOS/create_executable.bash    # -> .dmg  (run from repo root)
bundles\Windows\create_executable.cmd     # -> .zip
```

- **Python 3.13+ is required** (`setup.py` / type-syntax like `str | None` is used throughout).
- **ffmpeg must be on PATH** (or pass `--ffmpeg`). The bundled binaries ship ffmpeg; from source you supply it. GPU encoders are selected via `--gpu`/`--gpu_type` (nvidia/intel/qsv/rpi/vaapi) plus `--encoding x264|x265`.
- There is no linter config beyond `doc8`; the code carries `pylint`/type-ignore comments but no enforced lint step. CI only runs CodeQL (`.github/workflows/codeql-analysis.yml`).

## Architecture

Essentially the entire program lives in one ~6000-line module: `tesla_dashcam/tesla_dashcam.py`.
`__main__.py` and `__init__.py` just expose `main()`. There are no submodules — the
top-of-file TODO notes this is intentional-but-unfinished ("Move everything into classes
and separate files"). When editing, expect to work within this single file.

Two conceptual halves:

**Data model classes** (mirror the on-disk structure, mostly getter/setter property bags):

- `Event` — one event folder; holds `Event_Metadata` (reason/location parsed from Tesla's `event.json`) and a time-ordered map of `Clip`s.
- `Clip` — one minute of footage; maps camera name → `Camera_Clip`.
- `Camera_Clip` — a single camera's MP4 file for that minute, with its `Video_Metadata`.
- `Video_Metadata` / `Chapter` — probed ffmpeg info (dimensions, fps, dar, codec) and chapter markers.
- `Movie` — an output movie assembled from events.

**Layout + rendering classes** (build the ffmpeg filtergraph):

- `MovieLayout` is the base; subclasses `FullScreen`, `Mosaic`, `Cross`, `Diamond`, `Horizontal` implement each `--layout` choice by positioning `Camera`s on the output canvas. `Camera` and `Font` compute scale/position/timestamp-overlay parameters. Perspective variants apply `FFMPEG_LEFT_PERSPECTIVE`/`FFMPEG_RIGHT_PERSPECTIVE` filter strings.

**Processing pipeline** (module-level functions, this is the main flow):

`main()` → parse args → `process_folders()` → `get_movie_files()` (discover + group files
into `Event`/`Clip`/`Camera_Clip`) → `get_metadata()` (batch ffmpeg probe) →
`create_intermediate_movie()` (merge cameras per clip via the layout filtergraph) →
optional `create_title_screen()` → `create_movie()` / `create_movie_ffmpeg()` (concatenate
into the final per-event or merged movie). `--monitor*` flags make `main` wait on a trigger
(e.g. USB insertion) and loop, sleeping `MONITOR_SLEEP_TIME`.

## Configuration conventions

- **Platform behavior is table-driven** via dicts keyed on `sys.platform`: `FFMPEG`, `MOVIE_HOMEDIR`, `DEFAULT_FONT`, plus `MOVIE_QUALITY` (CRF map) and `MOVIE_ENCODING` (codec → ffmpeg encoder, including per-GPU variants). `PLATFORM`/`PROCESSOR` are resolved once at import (Apple-silicon detection via `sysctl`). To test other platforms, the commented `PLATFORM = ...` overrides near the top are the intended hook.
- **Preference presets**: `Preference_Files/*.txt` are argparse response files. The parser (`MyArgumentParser`, `fromfile_prefix_chars="@"`) reads them via `@FILENAME` on the command line; `convert_arg_line_to_args` lets each line hold flags and `#` comments. `RunPreferences.bash` runs the tool across all presets. When adding a CLI option, presets and the README usage section may need updating too.
- **Version** is defined in two places kept in sync: `VERSION` dict in `tesla_dashcam.py` and `tesla_dashcam/__version__.py` (the latter is what `setup.py` reads). `beta > -1` in the `VERSION` dict marks a beta build.
- `check_latest_release()` queries the GitHub releases API (`GITHUB` dict) for the update-check feature.

## Notes for editing

- `README.md` is the authoritative user documentation (very long; the full `--help`/usage is embedded there). Behavior changes should be reflected in its usage section.
- ffmpeg timestamp/text overlays go through `escape_drawtext_literals()` — drawtext escaping is subtle and directly unit-tested (`TestDrawtextEscaping`); don't hand-edit escaping logic without running those tests.
- Tests are pure unit tests: they exercise layout geometry and stub `subprocess.run`/`CompletedProcess` rather than invoking real ffmpeg, so they run without ffmpeg or sample footage.
