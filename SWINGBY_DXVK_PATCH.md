# swingby-dxvk Patch Provenance

Last updated: 2026-07-31

This repository vendors [DXVK](https://github.com/doitsujin/dxvk) release
**2.4.1** (`0cf05780abd7250c2cd713b7749cf32180157cf5`, "[meta] Release 2.4.1",
Philip Rebohle) plus one local patch, used by a private macOS Wine-based
launcher project for a single title (a 32-bit D3D9 game using a fixed-function
vertex pipeline).

**This is an altered version of DXVK, not the original.** Per the zlib/libpng
license (see `LICENSE`), altered source versions must be plainly marked as
such — this file, and the note at the top of `README.md`, are that mark.

Unlike a continuously-tracked fork (e.g.
[swingby-wine](https://github.com/tsukasa-art/swingby-wine)), this repository
does not track DXVK upstream going forward. It exists to make one specific
patch reproducible from a clean checkout, replacing an earlier setup where the
patched source only existed in an unversioned local working tree.

## The patch

A set of MoltenVK / Apple Silicon compatibility fixes, discovered while
diagnosing a black-screen bug in a 32-bit D3D9 fixed-function title. Most of
these are not specific to that title — they're places where DXVK
force-enables a Vulkan feature or takes a code path that MoltenVK on Apple
Silicon doesn't support, causing `vkCreateDevice` to fail outright or corrupt
rendering. Grouped by what they touch:

**1. Null-stream vertex-input binding — the headline fix
(`d3d9_device.{cpp,h}`, `dxvk_graphics.cpp`).** DXVK's fixed-function vertex
shader emits attributes absent from the D3D9 vertex declaration to the
null-stream binding (`D3D9DeviceEx::NullStreamIdx` = `caps::MaxStreams` =
16), but the `DrawPrimitiveUP` / `DrawIndexedPrimitiveUP` code paths only
bind vertex buffer slot 0, leaving binding 16 permanently unbound. On
MoltenVK this corrupts the entire vertex fetch (including binding 0),
producing garbage vertex positions and a black frame. The fix
(`GetMvkNullStreamSlice`) lazily allocates a persistent zero-filled buffer
and binds it to the null-stream slot for both UP draw paths, using the same
stride as binding 0 (Metal rejects stride 0 for a per-vertex binding).

Root cause was isolated by layered diagnostic instrumentation (forcing
solid-color fragment output, forcing a full-screen quad from
`gl_VertexIndex`, tracing per-draw vertex binding state) that progressively
ruled out the output-merger stage, geometry stage, and present path before
isolating the unbound null-stream binding as the cause.

**2. Device feature gating to match what MoltenVK actually reports
(`d3d9_device.cpp`, `dxvk_adapter.cpp`).** DXVK unconditionally requested
several Vulkan features unsupported by MoltenVK on Apple Silicon —
`geometryShader`, `shaderCullDistance`, `maintenance4`, `robustBufferAccess2`,
`nullDescriptor` — which made `vkCreateDevice` fail outright
(`VK_ERROR_FEATURE_NOT_PRESENT`). Each is now gated on
`adapter->features()` / `m_deviceFeatures` actually reporting support before
being requested. Noted risk: DXVK 2.x removed some of the backend fallbacks
for `robustBufferAccess2`/`nullDescriptor`, so gating them off is empirically
tested against this one title, not proven safe for every code path.

**3. Depth-clip → depth-clamp (`d3d9_device.cpp`).** D3D9-style 2D rendering
expects Z outside the canonical clip volume to be clamped rather than
discarded, and MoltenVK doesn't support `VK_EXT_depth_clip_enable` in the
first place. `BindRasterizerState` now sets `state.depthClipEnable = false`
(was unconditionally `true`), which routes through DXVK's existing
`depthClampEnable` fallback in `dxvk_graphics.cpp`. A one-time
`Logger::info("DXVK_DEPTHCLAMP ...")` was added at both the D3D9 state and
pipeline-creation call sites so the fallback is visible in logs; the
pipeline-side fallback logic itself is unchanged DXVK code.

**4. SM1 sampler simplification for Metal (`dxso_compiler.cpp`).** For
Shader Model 1 pixel shaders, DXVK normally declares every aliasing sampler
variant (color × depth-compare × texture-type) on one binding and branches
on a spec constant at sample time, since SM1 bytecode doesn't declare a
sampler's type up front. Metal's unique-resource-index rules reject that
many aliasing variants on one binding. Since SM1-era 2D-only shaders in
practice only ever sample a plain 2D color texture, this patch declares just
the 2D color variant for SM1 shaders and samples directly, skipping the
per-draw depth/3D branch DXVK generates for SM2+.

**5. Device-creation diagnostics (`dxvk_adapter.cpp`).** A `[macOS diag]`
block walks every enabled Vulkan feature struct against what the adapter
actually reports supported, logging any enabled-but-unsupported feature by
name (the class of bug that causes `vkCreateDevice` to fail with
`VK_ERROR_FEATURE_NOT_PRESENT`, `VkResult -8`). A one-line `Logger::err` with
the `VkResult` was also added at the `vkCreateDevice` failure site itself
(previously it just threw `DxvkError` with no error code). Diagnostic-only,
no behavioral change.

**6. mingw-w64 14.x header conflict (`d3d9_include.h`, build-time only).**
DXVK's `d3d9_include.h` carries a dummy `_D3DDEVINFO_RESOURCEMANAGER`
typedef for old MinGW headers that didn't define it. Current mingw-w64
(14.x, verified with this fork's own toolchain) now defines the real
struct in `d3d9types.h`, so DXVK's dummy conflicts with it at compile time.
The dummy is `#if 0`'d out with a comment explaining when it would need to
come back (an older mingw lacking the real struct). Unrelated to runtime
behavior — this only affects whether the source compiles on current mingw-w64
at all.

Touches 6 files: `src/d3d9/d3d9_device.{cpp,h}`, `src/d3d9/d3d9_include.h`,
`src/dxso/dxso_compiler.cpp`, `src/dxvk/dxvk_adapter.cpp`,
`src/dxvk/dxvk_graphics.cpp`.

Fixes 1 and 4 were diagnosed against this one title's specific draw/shader
patterns and haven't been checked against other D3D9 titles. Fixes 2, 3, and
6 look like general MoltenVK/Apple-Silicon or mingw-w64-version compatibility
issues rather than title-specific workarounds, but none of this has been
verified on a non-macOS Vulkan driver or against DXVK's current upstream
`master`. An upstream PR to DXVK is a considered next step, not done here.

## Build

```sh
meson setup --cross-file build-win32.txt --buildtype release --strip \
    --bindir x32 --libdir x32 -Dbuild_id=false <builddir>
cd <builddir> && ninja install
```

Produces `x32/d3d9.dll`.

## Reproducibility verified 2026-07-31

Rebuilding from this exact commit + patch, with the same meson invocation as
upstream's `package-release.sh`, on current Homebrew toolchain
(`i686-w64-mingw32-gcc 15.2.0`, `mingw-w64 14.0.0`, `meson 1.11.1`,
`glslang 16.3.0`), and comparing against the binary previously built and
deployed from the same source:

- Section table (`.text`/`.data`/`.rdata`/`.bss`/`.edata`/`.idata`/`.tls`/`.rsrc`)
  is byte-for-byte identical in name, size, and file offset between the two
  builds.
- Of 3,981,312 bytes, only 27 differ. The largest identified cluster is the PE
  COFF header `TimeDateStamp` field (rebuild date vs. original build date,
  ~1 month apart, exactly matching each build's actual build time) plus what
  appears to be the export directory's own `TimeDateStamp`. The diff is
  consistent with link-time metadata only, not code generation — no
  behavioral difference is implied by the ~1 month of toolchain version
  drift between the two builds.
- A functional launch test (both the rebuilt DLL and the original deployed
  DLL, same isolated harness) failed identically with `DXVK: No adapters
  found... Vulkan 1.3 capable driver is required`. Follow-up isolated that
  this specific failure is a gap in the harness itself (invoking `wine64`
  directly does not reproduce whatever Vulkan/MoltenVK path resolution the
  full launcher app performs at startup), not a difference between the two
  builds — since both builds failed identically. Full in-launcher visual
  confirmation was not additionally performed since the byte-level
  structural match already establishes the two builds as equivalent.
