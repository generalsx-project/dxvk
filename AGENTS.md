# AGENTS.md

## What this repo is

- Fork of `doitsujin/dxvk` (remote: `generalsx-project/dxvk`): Direct3D 8/9/10/11 + DXGI → Vulkan translation layer. C++17, Meson ≥ 0.58. Version lives in root `meson.build` + `RELEASE` (currently 2.7.1).
- **GeneralsX Integration**: Provides native Direct3D 8/9 translation to Vulkan using SDL3 WSI for GeneralsX (`GameClient`) on macOS and Linux.
- **Branching Hierarchy (CRITICAL)**:
  - `master`: Strictly mirrors upstream `doitsujin/dxvk`. **Never land GeneralsX work directly on `master`.**
  - `generalsx-macos-v2.6`: **Primary active production branch** consumed by GeneralsX `GameClient` (referenced in `cmake/dx8.cmake`). All GeneralsX/macOS fixes and feature PRs must target this branch.
  - `generalsx-macos-upstream-pr-prep`: Cleaned-up, rebased commits intended for upstream pull requests to `doitsujin/dxvk`.
  - `fix/macos-*`: Working topic branches for isolated fixes.

## Setup — do this first

- Submodules are mandatory and frequently uninitialized. Without them, configure fails with `Missing Vulkan-Headers`, `Missing SPIRV-Headers`, or subproject errors:
  ```bash
  git submodule update --init --recursive
  ```
  Covers `include/vulkan`, `include/spirv`, `include/native/directx`, `subprojects/dxbc-spirv`, and `subprojects/libdisplay-info`.
- No `.gitignore` in the repository — build directories (`build/`, `build.64/`, `build.w64/`, `build-msvc-*`) appear untracked. Never commit build directories or temporary files.

## Build

### Native macOS (Apple Silicon arm64 — GeneralsX Target)
GeneralsX on macOS consumes native dylibs for D3D8 and D3D9 built with SDL3 WSI:
```bash
# Configure Meson with Apple Silicon flags and SDL3 WSI
CC=clang CXX=clang++ \
CFLAGS="-arch arm64 -mcpu=apple-m1" \
CXXFLAGS="-arch arm64 -mcpu=apple-m1" \
LDFLAGS="-arch arm64" \
meson setup build-macos-arm64 . \
  --native-file cmake/meson-arm64-native.ini \
  -Dnative_sdl3=enabled \
  --buildtype=release \
  --reconfigure

# Build only the D3D8 and D3D9 libraries needed by GeneralsX
ninja -C build-macos-arm64 src/d3d9/libdxvk_d3d9.0.dylib src/d3d8/libdxvk_d3d8.0.dylib
```

### Native Linux
GeneralsX on Linux builds native shared objects with SDL3 WSI:
```bash
meson setup build-linux . \
  -Dnative_sdl3=enabled \
  --buildtype=release
ninja -C build-linux src/d3d9/libdxvk_d3d9.so src/d3d8/libdxvk_d3d8.so
```

### Windows DLLs (MinGW cross-compilation)
```bash
./package-release.sh <version> <dest> --no-package --dev-build
```
Abort occurs if `<dest>/dxvk-<version>` already exists. Use `--64-only` or `--32-only` to skip an architecture. Incremental rebuild: `cd <dest>/build.64 && ninja install`.

### MSVC (Windows native)
First download legacy `d3d8.h`, `d3d8types.h`, and `d3d8caps.h` into `include/` (from `NovaRain/DXSDK_Collection`), then:
```bash
meson --buildtype release --backend vs2022 build-msvc-x86
msbuild -m build-msvc-x86/dxvk.sln
```

## Integration with GeneralsX GameClient

- **Local Fork Mode in GameClient**: To compile and link against this local checkout from `GameClient`, configure GameClient with:
  ```bash
  cmake -B build/macos-vulkan -DSAGE_DXVK_USE_LOCAL_FORK=ON
  ```
- **Runtime Libraries**: `GameClient` expects `libdxvk_d3d9.0.dylib` and `libdxvk_d3d8.0.dylib` (with unversioned symlinks `.dylib`). D3D8 links against D3D9 via `@rpath`, so both must be co-located.
- **Mandatory Environment Variable**: GeneralsX native windowing requires setting the WSI driver at runtime:
  ```bash
  export DXVK_WSI_DRIVER=SDL3
  ```

## Architecture Map

- `src/util`, `src/spirv`, `src/vulkan`, `src/wsi` → Core support libraries and window-system integration.
- `src/dxvk` → Vulkan device, context, memory allocator, and pipeline core.
- `src/dxso` → D3D8/D3D9 bytecode to SPIR-V shader compiler. Subproject `dxbc-spirv` handles DXBC.
- API layers: `src/d3d8`, `src/d3d9`, `src/d3d10`, `src/d3d11`, `src/dxgi`.
- Native platform split: Root `meson.build` branches on `platform == 'windows'`. Native builds use `include/native/` shims (e.g. `HWND` → `SDL_Window*`) and version-scripts (`.sym`).
- Runtime options: Documented in `dxvk.conf` (overridable via `DXVK_CONFIG_FILE`).
- Profile requirements: `VP_DXVK_requirements.json` documents Vulkan features; keep in sync when modifying Vulkan hardware requirements.

## Known Gotchas & macOS Pitfalls

1. **Submodules**: Always run `git submodule update --init --recursive` before configuring; otherwise headers under `include/vulkan` and `include/spirv` will be missing.
2. **Missing `<cstddef>`**: Clang on modern macOS requires explicit `#include <cstddef>` for `size_t` in utility headers.
3. **`libc++` Tuple Extraction (`std::piecewise_construct`)**: Modern Apple Clang `libc++` fails with `try_key_extraction` when inserting into maps using curly brace tuples. Use `std::piecewise_construct` and `std::forward_as_tuple()` instead.
4. **Retina HiDPI Pixel Sizing**: On macOS Retina displays, swapchains must be sized in real physical pixels via `SDL_GetWindowSizeInPixels()` rather than logical points (`SDL_GetWindowSize()`), preventing viewport scaling degradation.
5. **Generated Headers**: Shaders compile to headers via `glslangValidator`, and `version.h` is generated via `git describe --dirty=+`. Never edit generated files directly.
6. **No Test Suite**: Verification is successful compilation (`ninja`). Validate with GeneralsX replay tests or smoke runs.

## Code Conventions & Contribution

- **Annotate Changes**: When adding GeneralsX-specific changes, follow the GeneralsX header convention:
  ```cpp
  // GeneralsX @keyword author DD/MM/YYYY Description
  ```
  Keywords: `@bugfix`, `@feature`, `@performance`, `@refactor`, `@tweak`, `@build`.
- **Commit Style**: Use Conventional Commits (`fix(macos): ...`) or DXVK subsystem prefixes (`[wsi] ...`, `[d3d9] ...`).
- **PR Target**: Pull requests for GeneralsX work must always target `generalsx-macos-v2.6` on `generalsx-project/dxvk`.
