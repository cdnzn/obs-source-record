# AGENTS.md

OBS Studio plugin **Source Record**: a filter that lets one source be recorded, streamed, or sent to the replay buffer independently of OBS's main output. Written in C (tabs, OBS/LLVM style). `origin` is the fork `cdnzn/obs-source-record`; upstream is `exeldro/obs-source-record`. Default branch is `master`.

## Layout

- `source-record.c` (~2650 lines) is the whole plugin: the filter source (`source_record_filter_info`, id `source_record_filter`), properties UI, hotkeys, output lifecycle, and obs-websocket vendor requests. `source-record.h` is intentionally empty — it exists only because `CMakeLists.txt` lists it in `target_sources`.
- `obs-websocket-api.h` is a vendored header whose functions go through `proc_handler_call`, i.e. obs-websocket is **not** a link-time dependency. `obs_module_post_load()` resolves two optional libobs symbols at runtime (`obs_encoder_set_frame_rate_divisor`, `obs_encoder_set_gpu_scale_type`) via `os_dlsym`.
- `data/locale/*.ini` — translations. `en-US.ini` is the source of truth (`OBS_MODULE_USE_DEFAULT_LOCALE("source-record", "en-US")`); add new text keys there first.
- `buildspec.json` pins the OBS version (29.0.0-beta1) and obs-deps/Qt hashes. Its `version` field is the source of truth: the CI build scripts rewrite `project(source-record VERSION ...)` in `CMakeLists.txt` from it, so keep the two in sync.
- `cmake/ObsPluginHelpers.cmake` is vendored upstream OBS tooling; it defines the compile flags and install layout.

CMake generates files **into the source tree**, not the build dir: `version.h` (from `version.h.in`), `source-record.rc` (Windows), and `installer.iss`. None of these are tracked and none are covered by `.gitignore` — do not commit them, and expect untracked entries after a build.

## Build

There is no test suite, linter, or typecheck in this repo — CI only builds. Do not look for tests and do not add test tooling.

Two build modes:

- **In-tree (the upstream/CI path).** Check out into `obs-studio/plugins/source-record`, add `add_subdirectory(source-record)` to `plugins/CMakeLists.txt`, then build obs-studio. `BUILD_OUT_OF_TREE` is auto-set to `OFF` when `CMAKE_PROJECT_NAME` is `obs-studio`.
- **Standalone out-of-tree** (needs libobs + obs-frontend-api dev packages):
  `cmake -S . -B build -DBUILD_OUT_OF_TREE=On && cmake --build build`
  The README calls this Linux-only, but the CI scripts support all three platforms and download deps to `../obs-build-dependencies`:
  - Windows: `.github/scripts/Build-Windows.ps1 -Target x64 -Configuration RelWithDebInfo`
  - macOS: `.github/scripts/build-macos.zsh`
  - Linux: `.github/scripts/build-linux.sh`

Warnings are errors on every platform (`-Werror` plus `-Wextra`/`-Wswitch`/`-Wunused-parameter` on POSIX, `/WX` on MSVC; see `cmake/ObsPluginHelpers.cmake`). A new warning fails the build — use `UNUSED_PARAMETER(x)` for unused args.

## Style / formatting

- clang-format **13.0 only** (`.github/scripts/check-format.sh` rejects any other version; pinned in `.github/scripts/.Aptfile`). `.clang-format` is `InheritParentConfig` + `ColumnLimit: 132`, so the real style comes from OBS's root `.clang-format` when built in-tree.
- `check-format.sh` / `check-cmake.sh` exist but are **not** wired into `build.yml`; run them manually if you touch formatting. Note `check-format.sh` edits files in place; `check-changes.sh` then fails if anything changed.

## CI

`.github/workflows/build.yml` runs on push/PR to `master`: macOS (x86_64/arm64/universal), Linux in-tree against OBS 29.0.0-beta1, Windows x86/x64, then packages (CPack / pkg / Inno Setup). There is no lint or test job.
