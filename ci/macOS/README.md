# macOS CI Directory

## Overview

This directory contains scripts for building Scopy on macOS. The build process uses Homebrew for dependency management and creates a DMG installer for distribution.

### Scripts

#### `macos_config.sh`- Configuration file for macOS builds

#### `build_azure_macos.sh`- Main build script for Azure Pipelines

#### `install_macos_deps.sh`- Installs macOS build dependencies

#### `before_install_lib.sh`- Pre-installation setup for libraries (not used anymore)

#### `package_darwin.sh`- Creates the macOS DMG installer

## Build Process

### Prerequisites

- **Homebrew**: Package manager for macOS. Can be installed from here, [Homebrew Installation](https://docs.brew.sh/Installation).

### Build Steps

1. **Setup the environment and install the dependencies**:

   ```bash
      ./install_macos_deps.sh
   ```

   This will install packages using brew, so you will have to make sure that you have brew installed on the machine. The rest of the dependencies that can’t be found on brew will be built from the source files.

2. **Build Scopy**:

   ```bash
      ./build_azure_macos.sh
   ```

3. **Create Installer**:

   ```bash
      ./package_darwin.sh
   ```

   To run the application, the final step is linking the dependencies to the Scopy binary, enabling the operating system to locate them at runtime.

   This process is handled by a script that manages both the linking and packaging. Once the script completes, inside the build folder, it generates a file named Scopy.app, which can be opened either by running “open Scopy.app” in the terminal or by double-clicking it in the file explorer.

## Output

- **Scopy.app**: macOS application bundle
- **Scopy.dmg**: Distributable disk image installer

## CI Integration

- Built on Azure: See `azure-pipelines.yml` in repository root

## Notes

- x86_64 and native Apple Silicon arm64 builds are possible, but all dependencies must be built with the same architecture.
- On Apple Silicon, prefer native arm64 unless you intentionally build everything under Rosetta. Do not mix x86_64 and arm64 staged dependencies.
- The raw CMake build product is not enough to launch `Scopy.app`; run the dependency fixup/packaging step, or manually copy staged dependencies into `Scopy.app/Contents/Frameworks` and rewrite install names.

## Apple Silicon Native Build Notes

Use Qt 5 explicitly. Homebrew can have Qt 6 linked as `/opt/homebrew/bin/qmake`, while this project requires Qt 5.15.2 or newer:

```bash
export QT5=/opt/homebrew/opt/qt@5
export STAGING_AREA="$PWD/staging"
export STAGING_AREA_DEPS="$STAGING_AREA/dependencies"
export PATH="$QT5/bin:/opt/homebrew/opt/boost@1.85/bin:/opt/homebrew/opt/bison/bin:/opt/homebrew/opt/libtool/libexec/gnubin:/opt/homebrew/opt/libxml2/bin:$PATH"
export PKG_CONFIG_PATH="$STAGING_AREA_DEPS/lib/pkgconfig:/opt/homebrew/opt/libxml2/lib/pkgconfig:/opt/homebrew/opt/libzip/lib/pkgconfig:/opt/homebrew/opt/libffi/lib/pkgconfig:$PKG_CONFIG_PATH"
```

Install or confirm the Homebrew dependencies:

```bash
brew install qt@5 volk spdlog boost@1.85 pkg-config cmake fftw bison gettext autoconf automake libzip glib libusb glog doxygen wget gnu-sed libmatio dylibbundler libxml2 ghr libsndfile libtool ninja ccache
pip3 install --user mako
```

Build the staged source dependencies from the versions in `macos_config.sh`. When building natively on Apple Silicon, patch the staged `libm2k` clone before building it:

```diff
--- a/staging/libm2k/src/CMakeLists.txt
+++ b/staging/libm2k/src/CMakeLists.txt
@@
 if(APPLE)
 	# build universal binaries by default
-	set(CMAKE_OSX_ARCHITECTURES "x86_64")
+	set(CMAKE_OSX_ARCHITECTURES "arm64")
 endif()
```

Make sure Scopy and `libsigrokdecode` use the same Python runtime. A local Apple Silicon build failed when Scopy loaded Python 3.14 but `libsigrokdecode` loaded Python 3.12, causing a crash while loading sigrok decoders. If Homebrew `python@3.14` is the Python used by Scopy, add `python-3.14-embed` before `python-3.12-embed` in the staged `libsigrokdecode/configure.ac`, then rebuild `libsigrokdecode`:

```bash
export PKG_CONFIG_PATH="/opt/homebrew/opt/python@3.14/lib/pkgconfig:$PKG_CONFIG_PATH"
cd staging/libsigrokdecode
git clean -xdf
./autogen.sh
./configure --prefix "$STAGING_AREA_DEPS" PYTHON3=/opt/homebrew/opt/python@3.14/bin/python3.14
make -j8 install
```

Configure and compile Scopy:

```bash
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

Before testing with `open build/Scopy.app`, run `package_darwin.sh` or do the equivalent bundle fixup. At minimum, the app must contain the staged frameworks and dylibs in `Contents/Frameworks`, and load commands must point to `@executable_path/../Frameworks/...` rather than `staging/dependencies` or bare names like `libqwt.6.dylib`.

Useful diagnosis commands:

```bash
build/Scopy.app/Contents/MacOS/Scopy
otool -L build/Scopy.app/Contents/MacOS/Scopy
find build/Scopy.app/Contents -type f \( -perm -111 -o -name '*.dylib' -o -name '*.so' \) -print0 | xargs -0 otool -L 2>/dev/null | grep staging/dependencies
ls -lt ~/Library/Logs/DiagnosticReports/Scopy* | head
```
