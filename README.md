# art3m1s-core

A multi-engine visual novel runtime written in Rust. `art3m1s-core` connects to Artemis/Art3m1s, RFVP, and KRKR/Kirikiri through modular adapters, sharing unified logging, media/FFmpeg pipelines, GPU rendering backends, and a versioned C ABI. Each engine adapter produces an engine-agnostic `DrawList` or composited frame, allowing the rendering backend to consume a single unified rendering contract.

The primary production host is the Flutter application [Art3m1s](https://github.com/Alphaly2K/art3m1s).

Artemis remains the most feature-complete and default engine pipeline. RFVP is an independent optional adapter. The KRKR adapter is in an early integration stage providing ABI definitions, compatibility probes, and cross-platform embedded upstream build paths. The Artemis pipeline does not manage native windows; KRKR currently manages internal application/window lifecycles via upstream C++ runtime, while final surface presentation and audio output remain the responsibility of the Host.

Original creator: **[Alphaly2K](https://github.com/Alphaly2K)**  
License: **Mozilla Public License 2.0 (MPL-2.0)**

---

## Features

- **Multi-Engine Runtime Boundaries**: Independent, versioned ABI facades (`Art3m1sApiV1`, `Art3m1sRfvpApiV1`, and optional `Art3m1sKrkrApiV1`) preventing cross-engine state pollution.
- **Script Engines & Lua Bridge**: Full interpretation of ASB, AST, and IET scripts with Lua 5.1 (Desktop & Android) and Luau (iOS).
- **Scene Composition & Shaders**: Complete scenegraph layer trees, affine transforms, blend modes, transitions, tweens, and pixel-accurate alpha hit testing (`clickablethreshold`). Full Artemis HLSL shader subset support.
- **Shared GPU Backends**:
  - Apple: Native Metal (with optional MetalFX Spatial upscaling).
  - Android & Desktop: OpenGL ES / ANGLE as reference path.
  - Experimental: Vulkan backend.
- **E-Mote 2D Animated Character System** (`crates/art3m1s-emote`):
  - Native playback of PSB animated character portraits at full 60 FPS.
  - Multi-texture format support: compressed **DXT5** and uncompressed 32-bit **RGBA8 / LeRGBA8** (with automatic BGRA-to-RGBA channel correction for natural skin tones).
  - Fast-path direct GPU texture upload for ultra-high-resolution textures (4096×4096).
  - Complete Lua runtime API support (`progress`, `isTimelinePlaying`, timeline queries, variable read/write).
- **Typography & Text Layout**: Text layout engine with Ruby characters, typewriter reveal animations, backlog tracking, host translation injection, and custom font overrides.
- **Archive & File System**: PFS archives (single and multi-volume, encrypted pf8) and raw unpacked directory resources, with structured save data management.
- **Input Pipeline**: Event dispatch for mouse pointer, keyboard, touch, right-click, hover, and drag.
- **Decoupled Host Event Queue**: Lock-free, unidirectional event queue for logs, media events, and UI commands without reverse Dart FFI callbacks.
- **Integrated Media Decoding**: Static FFmpeg (8.1+) software decoding for video cutscenes, fullscreen opening movies, and alpha-blended video effects (OGV/Theora) with skip/seek synchronization.

---

## Repository Structure

```text
src/
  ffi/                Versioned C ABI facades for Art3m1s, RFVP, and KRKR
  runtime/            Artemis runtime state, input dispatch, media, and frame stepping
  compositor/         Scenegraph layer tree, transitions, animations, and DrawList builder
  host/               Host event, resource, media, and logging boundaries
  render_pipeline/    Core-side orchestration of the shared rendering contract
  profiler/           Performance metrics and frame profiling
crates/
  art3m1s-log/        Independent logging crate
  art3m1s-media/      Media and FFmpeg integration boundaries
  art3m1s-render/     Shared DrawList, GPU backends (GL, Metal, Vulkan), surfaces, shaders, post-processing
  art3m1s-rfvp/       RFVP engine adapter and host runtime
  art3m1s-siglus/     Experimental Siglus VM adapter for art3m1s-render
  art3m1s-krkr/       Kirikiri/KRKR adapter, versioned ABI, and smoke test host
  asb-interpreter/    ASB/AST/IET script interpreter and Lua bridge
  art3m1s-emote/      Built-in E-Mote PSB parser, atlas, and animation runtime
  pf8/                PFS archive library (vendored from sakarie9/pfs-rs under MIT; see crates/pf8/VENDORED.md)
  pfs-upk-rust/       C ABI wrapper for PFS archive operations (produces libpfs_upk)
tools/game-probes/    Automated game compatibility probes
tests/                Integration and regression test suites
doc/                  Host integration guides and FFI reference manuals
```

---

## Building & Testing

```bash
cargo fmt --check
./scripts/test-all.sh
cargo build --release
```

Default features include GL, native Metal, experimental Vulkan, `rfvp-engine`, and `krkr-engine`. FFmpeg is optional. When upstream source/build directories are not provided, KRKR falls back to a non-runnable bootstrap shim. To build minimal core modules without GPU backends, use:

```bash
cargo build --no-default-features
```

### KRKR Build Levels

```bash
# Core ABI / bootstrap check (native shim returns unsupported; cannot run games)
cargo test --locked --offline \
  --manifest-path crates/art3m1s-krkr/Cargo.toml \
  --features native-bootstrap

# macOS native upstream runtime smoke test (requires prebuilt KRKRSDL3 checkout)
VCPKG_ROOT=/path/to/vcpkg \
KRKRSDL3_SOURCE_DIR=/path/to/krkrsdl3 \
KRKRSDL3_BUILD_DIR=/path/to/krkrsdl3_build \
cargo build --manifest-path crates/art3m1s-krkr/Cargo.toml \
  --features native-upstream-smoke \
  --bin krkr_upstream_smoke
```

See [crates/art3m1s-krkr/README.md](crates/art3m1s-krkr/README.md) and [crates/art3m1s-krkr/UPSTREAM.md](crates/art3m1s-krkr/UPSTREAM.md) for details on upstream revisions, CMake prerequisites, and smoke test parameters.

---

## Host Integration

- **Threading & Lifecycles**: Frame loop, resource boundaries, and KRKR conventions are documented in [doc/HOST_INTEGRATION.md](doc/HOST_INTEGRATION.md).
- **C ABI Reference**: Complete FFI declarations and data structures are in [doc/FFI_REFERENCE.md](doc/FFI_REFERENCE.md).
- **Shader Pipeline**: Runtime HLSL ABI, resource bindings, compilation flow, and constraints are detailed in [doc/SHADER_ABI.md](doc/SHADER_ABI.md).
- **Post-Processing**: Linear post-processing pipeline and render target layouts are in [doc/POST_PROCESS.md](doc/POST_PROCESS.md).

---

## Current Status & Known Constraints

- Native Vulkan is experimental; Android production targets continue to use ANGLE / OpenGL ES.
- MetalFX Spatial requires macOS 13+ / iOS 16+ with compatible GPU hardware; falls back to standard rendering on unsupported platforms.
- Artemis HLSL compiler targets shader patterns observed in tested visual novels, rather than being an arbitrary full-spec DirectX compiler.
- E-Mote support covers tested PSB structures, DXT5, and RGBA8 formats; proprietary easing curves or custom engine extensions may require case-by-case adapters.
- External host services such as HTTP networking, native platform calls, browser opening, and haptics rely on host callbacks.

---

## Credits & License

Original author and creator: **[Alphaly2K](https://github.com/Alphaly2K)**

Licensed under the **[Mozilla Public License 2.0 (MPL-2.0)](LICENSE)**:
- File-level copyleft: Any modifications made to existing covered files in this repository must remain open under MPL-2.0.
- Larger Works: Host applications, user interfaces, or distribution packages combining other licenses may retain their respective licenses.
- **Exception**: `crates/pf8` retains its upstream **[MIT License](crates/pf8/LICENSE)** (by sakarie9).
- Upstream KRKRSDL3 code in `crates/art3m1s-krkr` is governed by its original upstream licenses; see [THIRD_PARTY_NOTICES.md](crates/art3m1s-krkr/THIRD_PARTY_NOTICES.md).
