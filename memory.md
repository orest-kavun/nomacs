- [2026-07-11T13:29:38.350Z] **OpenCV Local Windows Build Process for NomaCs (D:\repo\nomacs):**

**Source:** `opencv` directory from git submodule (4.13.0 branch)

**Primary Build Method: Using scripts/make.py**

```bash
python scripts/make.py "D:/Qt/Qt-6.6.2/6.6.2/msvc2019_64/bin" --project opencv --force
# or with custom lib path
python scripts/make.py "qt/bin" --lib-path "D:\repo\nomacs\3rd-party\build" --project opencv
```

**Manual Build (like CI):**

- Source: `opencv` (from submodule)
- Build dir: `opencv/build`
- Install dir: `opencv/install`
- Generator: `-G Ninja -DCMAKE_BUILD_TYPE=Release` (preferred over Visual Studio generator)
- Modules: Only `core,imgproc` via `-DBUILD_LIST=core,imgproc`
- Key flags: `-DBUILD_TESTS=OFF`, `-DBUILD_PERF_TESTS=OFF`, `-DBUILD_EXAMPLES=OFF`
- Codecs: `-DWITH_JPEG=OFF`, `-DWITH_PNG=OFF`, `-DWITH_WEBP=OFF`, `-DWITH_OPENEXR=OFF`
- IO: `-DWITH_TIFF=ON`, `-DWITH_FFMPEG=OFF`, `-DWITH_GSTREAMER=OFF`, `-DWITH_OPENCL=OFF`

**Integration:**
After building OpenCV, configure nomacs with:

```bash
cmake -S ImageLounge -B build ^
  -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_PREFIX_PATH="opencv\install;D:/Qt/qtbase/.." ^
  -DTIFF_BUILD_PATH="opencv\build\3rdparty\libtiff" ^
  -DTIFF_CONFIG_DIR="opencv\3rdparty\libtiff"
cmake --build . --config Release
```

**Common Issues:**

- Stale `cmake_install.cmake`: Delete before reconfigure
- VS generator not found: Use `-G Ninja` instead
- Runtime needs: Only OpenCV Core & ImgProc modules

**Directory Structure After Build:**

```
D:\repo\nomacs\
├── opencv/                    # Source (git submodule)
│   ├── 3rdparty/              # Zlib, libtiff sources
│   └── install/bin/Release/   # Built libs: opencv_core4120.dll, opencv_imgproc4120.dll
├── build/                     # Nomacs output
│   └── nomacs/nomacs.exe
```

**Key Config Class:** `OpenCVConfig` in `scripts/make.py` handles minimal OpenCV build configuration.

- [2026-07-11T13:32:05.943Z] **GITHUB WORKFLOWS IN NOMACS (D:\repo\nomacs\.github\workflows):**

---

### **1. docker.yml - Docker Images for Linux**

**Trigger:** `workflow_dispatch` (manual only)

**Purpose:** Build and push Docker images for KDE Neon/Ubuntu variants

**Matrix Configs:**

- `neon:22.04` → Dockerfile.kde-neon.2204
- `neon:24.04` → Dockerfile.kde-neon.2404
- `ubuntu:24.04` → Dockerfile.ubuntu.2404

**Steps:**

1. Checkout repo (LFS enabled)
2. Login to ghcr.io with GITHUB_TOKEN
3. Copy specific Dockerfile, build, and push image
4. Output: `ghcr.io/nomacs/{name}:{tag}`

---

### **2. linux.yml - Linux Cross-Platform Builds**

**Trigger:** `workflow_dispatch` (manual), was commented out for push/PR

**Purpose:** Multi-matrix Linux builds with AppImage support and clang-format checking

**Matrix Strategy:**

- 5 build variants: minimum-clang, minimum-gcc, default-clang, default-gcc, quazip-gcc-2404
- Container images from `ghcr.io/nomacs/ubuntu:24.04` or `neon:24.04`
- CMake flags control features (OpenCV, RAW, TIFF, Plugins, Quazip)

**Key Steps:**

1. Checkout + LFS
2. Install deps from container image
3. Configure with Ninja + CMake flags
4. Build with `ninja`
5. Test with `ninja check`
6. If AppImage build: get version info, build AppImage, upload artifact

**PR Checks:**

- Separate `check` job for pull requests
- Runs `git clang-format --diff` to verify code style
- Runs `format-cmake.sh --check` for CMake formatting

---

### **3. macos.yml - macOS Builds**

**Trigger:** `workflow_dispatch` (manual)

**Purpose:** Build nomacs on Intel/Apple Silicon macOS with Homebrew packages

**Matrix Strategy:**

- 2 runners: `macos-15-intel`, `macos-15` (arm64)
- Qt6 with qt5compat, qtbase, qtimageformats, qtsvg, qttools
- 2 config types: `full` (with OpenCV, Quazip, KImageFormats), `minimum`

**Key Steps:**

1. Checkout with full history (for git rev-parse)
2. If OpenCV enabled: clone into `${RUNNER_TEMP}`, cache by compiler+commit hash
3. Install Homebrew packages (qt5compat, qtbase, libraw, etc.)
4. Cache OpenCV from `/opt/opencv`
5. Build OpenCV using `scripts/build-opencv.sh`
6. Build Quazip from source (brew has link issues on macOS 15)
7. Configure nomacs with Ninja + CMake flags
8. Build with `ninja`
9. Test with `ninja check`
10. If KImageFormats enabled: build separately
11. Make portable bundle (`ninja portable` → zip)
12. Or make full bundle using `macdeployqt` (slow, requires install_name_tool fix)
13. Upload artifacts as `.app.zip`

**OpenCV Cache:** Uses sha256 hash of compiler+commit for intelligent caching

---

### **4. windows.yml - Windows MSVC Builds**

**Trigger:** `workflow_dispatch`, `push` to master, `pull_request` to master

**Purpose:** Build portable .exe bundles for Windows

**Toolchain Setup:**

- MSVC via `ilammy/msvc-dev-cmd@v1`
- Ninja via `seanmiddleditch/gha-setup-ninja@v4`
- Qt 6.6.2 with qt5compat module
- vcpkg (for exiv2, libraw, vulkan-headers)

**Key Steps:**

1. Checkout + LFS
2. Install Qt via action
3. Setup vcpkg and cache
4. Install vcpkg dependencies (exiv2, libraw, vulkan)
5. Fetch OpenCV source (clone if not present)
6. Cache OpenCV build/install dirs
7. Build OpenCV with:
   - `-G "Visual Studio 17 2022" -A x64`
   - Minimal modules: `core,imgproc`
   - Shared libs, disable most features
8. Configure nomacs with Ninja + vcpkg toolchain
9. Build with `cmake --build`
10. Prepare package (copy exe, deploy Qt runtime with windeployqt)
11. Create ZIP archive
12. Upload artifact as `nomacs-windows.zip`

**OpenCV Cache Key:** Hash of entire OpenCV source tree

---

### **Common Patterns Across All Workflows:**

| Pattern                   | Implementation                                                       |
| ------------------------- | -------------------------------------------------------------------- |
| **Checkout**        | `actions/checkout@v4` with LFS, full history (macOS) or PR ref     |
| **OpenCV Source**   | Clone from GitHub 4.13.0 if not present                              |
| **OpenCV Build**    | Minimal build:`-DBUILD_LIST=core,imgproc`, shared libs, few codecs |
| **Build Generator** | Ninja (except Windows OpenCV which uses VS generator)                |
| **CMake Config**    | `-G Ninja -DCMAKE_BUILD_TYPE=Release`                              |
| **Testing**         | `ninja check` with GoogleTest                                      |
| **Artifacts**       | Upload as`.zip` or `.AppImage` with 30-day retention             |

---

### **OpenCV Handling in Workflows:**

**Windows (windows.yml):**

- Uses VS generator for OpenCV, Ninja for nomacs
- Builds both Release and Debug configs
- Installs to `opencv/install`

**Linux (linux.yml):**

- Uses system packages (`libopencv-dev`) via apt

**macOS (macos.yml):**

- Clones and builds OpenCV from source using `scripts/build-opencv.sh`
- Caches by compiler+commit hash for efficiency

---

### **Trigger Modes:**

- **docker.yml:** Manual only (`workflow_dispatch`)
- **linux.yml:** Manual only (was commented out for auto-trigger)
- **macos.yml:** Manual only (`workflow_dispatch`)
- **windows.yml:** Full triggers (dispatch, push, PR) - most active CI

---

**1. OpenCV Local Windows Build Process**

**Source:** `opencv` directory from git submodule (4.13.0 branch)

**Primary Build Method: Using scripts/make.py**

```bash
python scripts/make.py "D:/Qt/Qt-6.6.2/6.6.2/msvc2019_64/bin" --project opencv --force
# or with custom lib path
python scripts/make.py "qt/bin" --lib-path "D:\repo\nomacs\3rd-party\build" --project opencv
```

**Manual Build (like CI):**

- Source: `opencv` (from submodule)
- Build dir: `opencv/build`
- Install dir: `opencv/install`
- Generator: `-G Ninja -DCMAKE_BUILD_TYPE=Release` (preferred over Visual Studio generator)
- Modules: Only `core,imgproc` via `-DBUILD_LIST=core,imgproc`
- Key flags: `-DBUILD_TESTS=OFF`, `-DBUILD_PERF_TESTS=OFF`, `-DBUILD_EXAMPLES=OFF`
- Codecs: `-DWITH_JPEG=OFF`, `-DWITH_PNG=OFF`, `-DWITH_WEBP=OFF`, `-DWITH_OPENEXR=OFF`
- IO: `-DWITH_TIFF=ON`, `-DWITH_FFMPEG=OFF`, `-DWITH_GSTREAMER=OFF`, `-DWITH_OPENCL=OFF`

**Integration:**
After building OpenCV, configure nomacs with:

```bash
cmake -S ImageLounge -B build ^
  -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_PREFIX_PATH="opencv\install;D:/Qt/qtbase/.." ^
  -DTIFF_BUILD_PATH="opencv\build\3rdparty\libtiff" ^
  -DTIFF_CONFIG_DIR="opencv\3rdparty\libtiff"
cmake --build . --config Release
```

**Common Issues:**

- Stale `cmake_install.cmake`: Delete before reconfigure
- VS generator not found: Use `-G Ninja` instead
- Runtime needs: Only OpenCV Core & ImgProc modules

**Directory Structure After Build:**

```
D:\repo\nomacs\
├── opencv/                    # Source (git submodule)
│   ├── 3rdparty/              # Zlib, libtiff sources
│   └── install/bin/Release/   # Built libs: opencv_core4120.dll, opencv_imgproc4120.dll
├── build/                     # Nomacs output
│   └── nomacs/nomacs.exe
```

**Key Config Class:** `OpenCVConfig` in `scripts/make.py` handles minimal OpenCV build configuration.

---

- [2026-07-11T14:53:22.932Z] # Nomacs Windows Build Fix - Recent Findings Summary

## Original Problem (GitHub Actions Workflow)

**Error**: `CMake Error at cmake/Win.cmake:14 (pkg_check_modules): pkg-config tool not found`
**Location**: `D:\repo\nomacs\.github\workflows\windows.yml` + `ImageLounge/cmake/Win.cmake`

## Root Cause Analysis

The `cmake/Win.cmake` file uses pkg-config calls to find libraries:

```cmake
find_package(PkgConfig)
pkg_check_modules(EXIV2 REQUIRED exiv2>=0.27)  # Exif/IPTC/Xmp metadata library
if(ENABLE_RAW)
    pkg_check_modules(LIBRAW libraw>=0.22.1)   # RAW image support - was failing!
endif()
```

On Windows, **pkg-config isn't installed by default** (unlike Linux/macOS). The vcpkg toolchain provides the libraries but not necessarily the `pkg-config` executable itself without integration.

## Fix Evolution

### Attempt 1: Chocolatey Installation (~2 minutes)

- Added chocolatey step to install pkgconfig package
- **Pros**: Simple, reliable
- **Cons**: External dependency on chocolatey, longer setup time

### Attempt 2: VCPKG Integrate Install (~30 seconds)

```powershell
& "${{ github.workspace }}/vcpkg/vcpkg" integrate install --triplet x64-windows
```

- Uses existing vcpkg infrastructure
- **Cons**: Failed with temp directory cleanup error: `failed to remove_all(C:\Users\RUNNER~1\AppData\Local\Temp\vcpkg)`

### Current Fix (Final - ~5-10 seconds): VCPKG Integrate Export + Robust Fallbacks

```powershell
# Use export instead of integrate install for cleaner env var setup
& "${{ github.workspace }}/vcpkg/vcpkg" integrate export --triplet x64-windows

# Multi-location pkg-config search strategy:
$possiblePaths = @(
  "$Env{VCPKG_ROOT}\tools\bin",              # From integrate export
  "$Env{VCPKG_ROOT}\installed\x64-windows\tools\pkgconfig"  # Direct tools path
)

foreach ($path in $possiblePaths) {
  if (Test-Path "$path") { ... }
}

# Fallback: Look for pkg.exe or pkgconfig.exe in installed directory
```

**Benefits**:

1. Uses `integrate export` instead of `integrate install` - no temp cleanup issues
2. Multi-location search strategy handles different vcpkg installations
3. Sets both PKG_CONFIG_PATH and PATH properly
4. Minimal overhead (~5-10 seconds vs ~7 minutes for dependencies)
