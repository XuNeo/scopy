# AGENTS.md

This file is the repository-level guide loaded by coding agents. Keep it current when build steps, generated artifacts, or platform quirks change.

## Project Overview

Scopy is a Qt 5/CMake C++ application with a plugin/package architecture. The root target builds the `Scopy` application, shared internal libraries, and enabled packages/plugins.

Important top-level areas:

- `CMakeLists.txt`: root CMake entry point, Qt 5 requirement, app bundle setup, shared library output paths, package inclusion.
- `common/`, `iioutil/`, `gui/`, `gr-util/`, `pluginbase/`, `pkg-manager/`, `core/`, `iio-widgets/`: core internal libraries.
- `packages/`: product packages and plugins such as `m2k`, `generic-plugins`, `swiot`, `imu`, RF/ADI device plugins.
- `cmake/Modules/`: custom CMake helpers such as `ScopyMacOS`, `ScopyTest`, package utilities, Qwt finder.
- `ci/`: platform build scripts. macOS scripts are in `ci/macOS/`.
- `docs/`, `js/`, `tools/`, `testing_results/`: documentation, JS automation, tooling, and test artifacts.

## General Working Rules

- Treat the repository as a dirty worktree unless proven otherwise. Do not delete or revert user changes.
- Prefer narrow edits in the relevant module or package. Avoid broad formatting or generated-file churn unless the task explicitly calls for it.
- Use `rg` / `rg --files` for search.
- Do not hand-edit build products under `build/` or third-party clones under `staging/` as source changes. If you must patch a staged dependency to make a local build work, document it clearly.
- Generated CMake headers may appear during configure, especially package config/export headers. Do not assume they are source changes without checking their origin.
- This repo uses GPLv3 licensing headers in source files. New source files should follow the surrounding copyright/header style.

## Coding Style

Follow `.github/contributing/coding-guidelines.md`:

- C++ member variables use `m_`.
- Variables use `camelCase`; class names start uppercase.
- File names are lowercase, and class names should match file names case-insensitively.
- Use explicit Qt macros such as `Q_SLOTS`; do not use bare `slots`.
- Prefer existing patterns in the module. Be careful with new `Manager`/`Controller` names; only add them when the abstraction is justified.
- Keep established design-pattern naming consistent with existing strategy, singleton, and factory examples.

Formatting:

- `tools/format.sh` is the documented formatting entry point.
- Avoid running formatting across unrelated files unless the request is a formatting task.

## Build System Basics

Scopy uses CMake and requires Qt 5.15.2 or newer:

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build -- -j8
```

That minimal form only works after all dependencies are discoverable. Key dependencies include Qt 5, libiio, ECM/KF5Archive, Qwt, libsigrokdecode, libm2k, GNU Radio components, KDDockWidgets, and package-specific libraries.

Common CMake options:

- `-DENABLE_TESTING=OFF`: disable unit-test target setup for faster app builds.
- `-DENABLE_TRANSLATION=ON|OFF`: translation resources, default on.
- `-DENABLE_PACKAGE_<NAME>=ON|OFF`: package toggles are defined inside package CMake files.
- `-DENABLE_PLUGIN_<NAME>=ON|OFF`: plugin toggles are defined by package/plugin CMake.

On macOS, internal shared libraries are written into `build/Scopy.app/Contents/Frameworks`, and package plugins into `build/Scopy.app/Contents/MacOS/packages`.

## Tests

CTest is enabled at the root. Unit-test helpers are wired through `cmake/Modules/ScopyTest.cmake`.

Typical checks:

```sh
cmake --build build -- -j8
ctest --test-dir build --output-on-failure
```

JS automated tests are controlled by `ENABLE_AUTOMATED_TESTS` in `tests/CMakeLists.txt` and use scripts under `js/`.

When a task touches a specific package/plugin, prefer the narrowest relevant test target or CTest filter. If tests cannot be run because hardware, emulator, or packaged dependencies are unavailable, report that explicitly.

## macOS Build Guidance

The CI flow is:

```sh
./ci/macOS/install_macos_deps.sh
./ci/macOS/build_azure_macos.sh
./ci/macOS/package_darwin.sh
```

For local machines, do not run `install_macos_deps.sh` blindly. It contains CI-specific cleanup of `/usr/local/bin/python3*` and CMake. Prefer manual Homebrew installation and reuse the relevant dependency build steps.

The raw CMake build product is not enough to launch `Scopy.app`. The app bundle must be fixed up by `ci/macOS/package_darwin.sh` or equivalent copying of staged dependencies into `Contents/Frameworks` plus `install_name_tool` rewriting.

If `open build/Scopy.app` shows the generic macOS "contact the developer" dialog, first run:

```sh
build/Scopy.app/Contents/MacOS/Scopy
```

Then inspect:

```sh
otool -L build/Scopy.app/Contents/MacOS/Scopy
ls -lt ~/Library/Logs/DiagnosticReports/Scopy* | head
```

The useful error is usually a `dyld` missing library message or a crash stack in the `.ips` report.

## Apple Silicon Native Build Notes

Native Apple Silicon builds are possible. Do not assume Rosetta/x86_64 is required. All dependencies and Scopy must use the same architecture, normally `arm64`.

Use Qt 5 explicitly. Homebrew can have Qt 6 linked as `/opt/homebrew/bin/qmake`, but Scopy requires Qt 5:

```sh
export QT5=/opt/homebrew/opt/qt@5
export STAGING_AREA="$PWD/staging"
export STAGING_AREA_DEPS="$STAGING_AREA/dependencies"
export PATH="$QT5/bin:/opt/homebrew/opt/boost@1.85/bin:/opt/homebrew/opt/bison/bin:/opt/homebrew/opt/libtool/libexec/gnubin:/opt/homebrew/opt/libxml2/bin:$PATH"
export PKG_CONFIG_PATH="$STAGING_AREA_DEPS/lib/pkgconfig:/opt/homebrew/opt/libxml2/lib/pkgconfig:/opt/homebrew/opt/libzip/lib/pkgconfig:/opt/homebrew/opt/libffi/lib/pkgconfig:$PKG_CONFIG_PATH"
```

Homebrew dependencies used successfully on Apple Silicon:

```sh
brew install qt@5 volk spdlog boost@1.85 pkg-config cmake fftw bison gettext autoconf automake libzip glib libusb glog doxygen wget gnu-sed libmatio dylibbundler libxml2 ghr libsndfile libtool ninja ccache
pip3 install --user mako
```

Build staged dependencies from versions in `ci/macOS/macos_config.sh`.

Known Apple Silicon staged dependency fixes:

- Patch staged `libm2k/src/CMakeLists.txt` from `set(CMAKE_OSX_ARCHITECTURES "x86_64")` to `set(CMAKE_OSX_ARCHITECTURES "arm64")` before building `libm2k`. Otherwise `libm2k` produces x86_64 objects and fails against arm64 `libiio`.
- Keep Scopy and `libsigrokdecode` on the same Python runtime. A local build crashed while loading sigrok decoders because Scopy linked Python 3.14 while `libsigrokdecode` linked Python 3.12. If Scopy uses Homebrew Python 3.14, add `python-3.14-embed` before `python-3.12-embed` in staged `libsigrokdecode/configure.ac`, then rebuild `libsigrokdecode`.

Scopy configure command used successfully after staged dependencies existed:

```sh
rm -rf build
cmake -S . -B build \
  -DCMAKE_LIBRARY_PATH="$STAGING_AREA_DEPS" \
  -DCMAKE_INSTALL_PREFIX="$STAGING_AREA/scopy-install" \
  -DCMAKE_PREFIX_PATH="$STAGING_AREA_DEPS;$STAGING_AREA_DEPS/lib/cmake;$STAGING_AREA_DEPS/lib/pkgconfig;$STAGING_AREA_DEPS/lib/cmake/gnuradio;$STAGING_AREA_DEPS/lib/cmake/iio;$QT5;/opt/homebrew" \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_VERBOSE_MAKEFILE=ON \
  -DCMAKE_STAGING_PREFIX="$STAGING_AREA_DEPS" \
  -DCMAKE_EXE_LINKER_FLAGS="-L$STAGING_AREA_DEPS/lib" \
  -DENABLE_TESTING=OFF \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5

cmake --build build -- -j8
```

Post-build bundle fixups needed locally:

- Copy staged frameworks/dylibs into `build/Scopy.app/Contents/Frameworks`.
- Rewrite staged absolute paths and bare names such as `libqwt.6.dylib` to `@executable_path/../Frameworks/...`.
- Recopy rebuilt `libsigrokdecode` and decoder files into the app if Python runtime alignment was changed.
- `codesign --deep` may fail on staged `iio.framework` with `unsealed contents present in the root directory of an embedded framework`. Verify launch separately; a failed deep signature does not automatically mean local `open` is impossible.

Keep exact Apple Silicon build commands here or in the relevant platform README when they are expected to be reused.

## Generated And Local Build Artifacts

Common generated paths:

- `build/`, `build-probe/`: CMake build trees.
- `staging/`: third-party dependency clones and staged installs for local macOS builds.
- `build-status`: generated by macOS dependency scripts for about/build info.
- `packages/*/include/*/scopy-*_config.h` and `scopy-*_export.h`: may be generated during CMake configure.

Do not commit local build trees or staged third-party clones unless explicitly requested. If generated headers appear as untracked files, identify whether they are required source artifacts before adding them.

## Packaging

Use platform scripts under `ci/`:

- macOS: `ci/macOS/package_darwin.sh`
- Windows: `ci/windows/`
- Linux AppImage: `ci/x86_64/`, `ci/arm/`
- Flatpak: `ci/flatpak/`

For macOS, packaging is also the runtime dependency fixup step. A successful `cmake --build` does not imply a double-clickable app.

## Documentation For Future Agents

- Keep durable project instructions in this `AGENTS.md`.
- Avoid committing one-off machine-specific transcripts. If exact local package versions matter, summarize only the reusable facts in the relevant platform README.
- When fixing a platform-specific build issue, update both this file with the general rule and the platform README with command-level detail.
