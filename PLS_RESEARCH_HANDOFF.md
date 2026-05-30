# PLS Research Handoff — apitrace, Mesa drivers, Vulkan tile memory

> Context digest from a Claude Code (web) research session, written so a **local
> Claude Code session can continue the work**. Drop this file into whatever repo
> you continue in and `@`-reference it, or paste relevant sections.
>
> **Important:** all the source trees referenced below were *ephemeral clones in
> `/tmp` on a remote container* and are gone now. Re-clone locally (commands at
> the end). All `file:line` references are against the upstream sources as of the
> dates noted.

---

## 0. The thread in one paragraph

Goal evolved from "how to extend **apitrace** for `GL_EXT_shader_pixel_local_storage`
(PLS)" → "how **Mesa** added PLS for **Panfrost**, and how to port it to another
gallium driver (**freedreno** / **vc4** / **v3d**)" → testing reality for PLS →
**Vulkan** tile-memory-for-compute (and how it maps to Khronos Vulkan-Docs #2093
and to tone-mapping). Recommendation: for a *useful, tractable* GL PLS port, target
**v3d** (easiest) or **freedreno** (most relevant). For Vulkan compute-on-tile, the
capability exists only as Qualcomm's `VK_QCOM_tile_shading`; Mali has no equivalent.

---

## 1. apitrace PLS support

apitrace = spec-driven codegen (`specs/*.py` → tracer/retracer/dispatch C++).
Source reviewed: `github.com/apitrace/apitrace` @ commit `b5a4ae9`.

**Already present (partial support):**
- PLS1 enums (no functions in PLS1): `specs/glparams.py:2944-2948`
- PLS2 functions: `specs/glapi.py:2605-2608`
- PLS2 enums: `specs/glparams.py:3631-3633`

**The bug (concrete fix):** PLS2's `target` is typed `GLframebuffer` (a `Handle`)
at `specs/glapi.py:2606-2607`, but the spec's `target` is a **framebuffer
binding-target enum** ("the framebuffer bound to `<target>`"), exactly like
`glFramebufferParameteri` which apitrace correctly types `GLenum`
(`specs/glapi.py:1200`). Because retrace rewrites every `Handle` arg through a
name map (`retrace/retrace.py:46` → `_framebuffer_map[value]`, applied at `:207`),
the enum `GL_DRAW_FRAMEBUFFER` gets looked up as a (nonexistent) FBO name →
broken replay. **Fix: `GLframebuffer` → `GLenum` on both functions.**

**Optional polish:** flip the PLS limit tokens' `function` field in
`specs/glparams.py` from `""` to `glGet` so `retrace/glstate_params.py` dumps them
(state inspection). Shader side needs nothing (PLS lives in GLSL source, captured
verbatim).

**Model extension to copy end-to-end:** `GL_EXT_disjoint_timer_query`
(`specs/glapi.py:2369-2381`) — functions + enums + the one custom retrace bit.

---

## 2. Mesa PLS architecture (the Panfrost template)

Source: Mesa mirror `github.com/mirror/mesa` (HEAD ~2026-05-29). PLS shipped in
**Mesa 26.0** by **Collabora** for **Panfrost v6+** (`docs/features.txt:340`).
Exact MR number not recoverable offline (gitlab.freedesktop.org firewalled).

**Two layers. The shared layer is already upstream — any gallium driver gets it free:**
- GLSL frontend parses `__pixel_localEXT` → `nir_var_mem_pixel_local_*`
  (`src/compiler/glsl/glsl_to_nir.cpp:536`).
- `nir_lower_io` on `nir_var_any_pixel_local` emits intrinsics
  `nir_intrinsic_load_pixel_local` / `store_pixel_local`, **carrying the
  layout-qualified format** (`var->data.image.format`), location, offset
  (`src/compiler/nir/nir_lower_io.c:389-452, 628-630`).
- Intrinsic defs: `src/compiler/nir/nir_intrinsics.py:1306, 1357`.
- Helper passes: `nir_downgrade_pls_vars.c` (inout→in/out fixup),
  `nir_lower_io_vars_to_temporaries`.
- **Extension enabled purely by a gallium cap:**
  `src/mesa/state_tracker/st_extensions.c:1120` →
  `EXT_CAP(EXT_shader_pixel_local_storage, shader_pixel_local_storage_size)`.
  Cap fields: `src/gallium/include/pipe/p_defines.h:1147-1148` (default 0).
- GL state/validation all shared: `mtypes.h:3594` (`ctx->PixelLocalStorage`),
  `fbobject.c:3331+`, `draw_validate.c:110+`.
- FB flag to driver: `st_atom_framebuffer.c:149` sets
  `pipe_framebuffer_state::pls_enabled` (`p_state.h:444`).

**Driver layer (all Panfrost added):**
- Advertise cap: `pan_screen.c:728-730` (`= 16` for `arch >= 6`).
- 63-line NIR pass `pan_nir_lower_pls.c`: rewrites load/store_pixel_local →
  `nir_load_tile_pan`/`nir_store_tile_pan` at byte offset `offset*4 + base*4`
  with a format-conversion descriptor.
- Pipeline wiring: `pan_shader.c:516-545` (downgrade_pls_vars →
  lower_io_vars_to_temporaries → lower_io for PLS → re-gather), pass call at
  `pan_shader.c:169-170` gated on `info.fs.accesses_pixel_local_storage`.
- Job/FB: `pan_job.c:479`, `pan_csf.c:105` consume `pls_enabled` (Mali forces
  `sample_count=4` "mega-sample" — Mali-specific, not needed elsewhere).

**Key mental model: PLS ≈ framebuffer-fetch into the tile buffer at a byte
offset.** So porting = map the two PLS intrinsics onto a driver's tile read/write.
Only Panfrost currently sets the cap.

---

## 3. Driver comparison for porting PLS

| | **freedreno** (Adreno) | **v3d** (Pi 4/5) | **vc4** (Pi 0–3) |
|---|---|---|---|
| Relevance | Highest (mobile, real PLS apps) | High (RPi/embedded) | Low (legacy/EOL) |
| GLES/GLSL | 3.2 / 4.6 a6xx+ (`freedreno_screen.c:520-521`) | 3.1 / 3.30 (`v3d_screen.c:309-311`) | 2.0 |
| FB fetch | coherent, via `txf_ms_fb` descriptor (`ir3_compiler_nir.c:3988`) | **coherent, per-RT, format-typed TLB** (`v3d_screen.c:335-336`) | single fixed RGBA8 slot |
| Tile read helper | — | `v3d_nir_get_tlb_color()` (`v3d_nir_lower_load_output.c:7`) | `nir_load_tlb_color_brcm` (`vc4_nir_lower_blend.c:60`) |
| Effort for real PLS | Moderate | **Moderate–low** | High + HW-capped (toy) |

- `vc5` is **not** a real target — it was the codename for **v3d** (only trace:
  `SWVC5-718` erratum in `v3dx_job.c:57`). Drivers are `vc4` and `v3d` (+`v3dv`).
- **Recommendation:** prototype on **v3d** first (closest to Panfrost's clean
  pass; per-RT format-typed coherent TLB + reusable read/write helpers), then port
  to **freedreno** for high-value Adreno reach.

---

## 4. v3d PLS work plan

v3d compiler: NIR → VIR → QPU. FS lowering pipeline entry:
`v3d_nir_lower_fs_early` in `src/broadcom/compiler/vir.c:1211`.

- **A. Caps** (`v3d_screen.c`, near `:335`): set
  `shader_pixel_local_storage_size`/`_fast_size` (version-dependent budget,
  `V3D_MAX_RENDER_TARGETS(ver)`; v3d 71 > v3d 42). Flips the extension on.
- **B. Wire shared pre-lowering** into `v3d_nir_lower_fs_early`, mirroring
  `pan_shader.c:516-545`, gated on `info.fs.accesses_pixel_local_storage`.
- **C. New pass** `src/broadcom/compiler/v3d_nir_lower_pls.c` (model on
  `pan_nir_lower_pls.c`): `rt = io_semantics.location - FRAG_RESULT_DATA0`;
  - load → `v3d_nir_get_tlb_color(b,c,rt,sample)` then `nir_format_unpack_*` to
    the PLS format;
  - store → `nir_format_pack_*` then route to `c->outputs[rt]` (existing color
    write path `ntq_emit_color_write`, `nir_to_vir.c:2881`) → flushed by
    `vir_emit_tlb_color_write` (`:1888`).
- **D. Storage/format model (HARDEST — do as a spike first):** back PLS slots
  with RTs in a **raw integer format** (R32UI/RGBA8UI) so the TLB moves raw bits,
  do all pack/unpack in NIR. TLB format type field: `nir_to_vir.c:1859-1872`.
  `fs_key->color_fmt[]` at `v3d_compiler.h:453`.
- **E. FB/job (RCL) plumbing:** consume `pls_enabled`; configure PLS-backing RTs
  as **transient** (don't store back; PLS is discarded per spec). v3d job tracks
  per-RT load/store masks (`v3d_job.c`, `v3dx_rcl.c`, `v3dx_emit.c`). No
  mega-sample hack needed.
- **F. Coherency/ordering:** `fbfetch_coherent=true` already (`v3d_screen.c:336`);
  TLB scoreboard-locked (`nir_to_vir.c:167-171`). Verify load→store→load RMW order.
- **G. Testing:** see §5 — there is essentially no dEQP/CTS coverage; build unit
  tests + differential vs Panfrost. Run under v3d simulator for **both ver 42 & 71**.

**Estimate:** ~2.5–4 engineer-weeks for first conformant single/multi-slot impl;
packed-float formats + MSAA are the long tail.

**Risks:** format aliasing (D), transient-RT semantics (E), PLS-on/off shader
variant keying, Pi4 vs Pi5 RT-count/TLB diffs.

---

## 5. Testing reality (verified — my earlier claims were WRONG)

Checked VK-GL-CTS (`github.com/KhronosGroup/VK-GL-CTS` @ 2026-05-29, `f94e5f9`):
- **`dEQP-GLES2.functional.shaders.pixel_local_storage.*` DOES NOT EXIST.** I
  fabricated that path.
- **No `KHR-GLES*` PLS functional tests** exist (`external/openglcts/modules` has
  none; PLS is EXT, never given a conformance suite).
- **Only real PLS test:**
  `dEQP-EGL.functional.get_proc_address.extension.gl_ext_shader_pixel_local_storage2`
  (`modules/egl/teglGetProcAddressTests.inl:1434-1438, 2198`) — just checks the
  three **PLS2** function pointers are non-NULL. PLS1 has no functions → zero
  functional coverage.
- Corroboration: Panfrost CI lists PLS only as *advertised*
  (`src/panfrost/ci/panfrost-*-gles2-extensions.txt`), no caselist.

**So validation must be custom**: unit tests (single r32ui slot → each format →
in/out/inout qualifiers → multi-pass RMW) with `glReadPixels` + exact asserts,
plus **differential testing vs Panfrost**. (piglit coverage unverified — check it.)

**Reference workload — ARM "Translucency" sample (real, MIT):**
`github.com/ARM-software/opengl-es-sdk-for-android` (unmaintained),
`samples/advanced_samples/Translucency/`. Deferred shading + subsurface scattering
with the G-buffer entirely in PLS.
- Shaders (`assets/`): `prepass.fs` (writer `__pixel_localEXT`),
  `thickness.fs`/`scattering.fs`/`opaque.fs` (RMW), `resolve.fs`
  (`__pixel_local_inEXT` reader).
- Driver: `jni/app.cpp` (`:102` ext check, `:267`
  `glEnable(GL_SHADER_PIXEL_LOCAL_STORAGE_EXT)`).
- PLS block uses **mixed packed formats** (`layout(rgb10_a2)` + `layout(rg16f)`) —
  exactly the format pack/unpack case that stresses v3d Workstream C/D.
- **Fit: good demo + coarse end-to-end/differential oracle; NOT a granular test**
  (no built-in pass/fail, multi-feature, animated/Android). Harvest its shaders;
  build deterministic unit tests separately.

---

## 6. Vulkan tile memory for compute (the architecture question)

- Tile memory is a **fragment/rasterization-pipeline** resource (implicit
  pixel→tile-slot addressing). A plain `vkCmdDispatch` has no "current tile."
- **Compute's native fast on-chip scratch = workgroup `shared` memory**
  (SPIR-V `Workgroup`). On mobile often the same SRAM as the tile buffer. This —
  not the tile buffer — is the answer for generic "fast local working memory."
- Extensions (verified in `include/vulkan/vulkan_core.h`):
  - `VK_EXT_shader_tile_image` — **fragment-only** PLS-equivalent.
  - `VK_QCOM_tile_shading` (Vulkan **1.4.312**, 2025) — **compute inside a render
    pass, per-tile**, with tile-image load/store/sample + **tile aprons** →
    *compute can touch tile memory*. **Adreno only.**
  - `VK_QCOM_tile_memory_heap`, `VK_QCOM_tile_properties`.
  - **No `VK_ARM_*` equivalent — Mali cannot do compute-on-tile today.**

**Khronos Vulkan-Docs #2093** (opened @DethRaid 2023-03-31): formal request for
**compute access to tile memory like `VK_EXT_shader_tile_image` gives fragment
shaders**. Their cited workaround = "fragment shader with no color writes doing
compute-like work" (the PLS/fragment-as-compute trick, cf. ARM Translucency
resolve). This issue *is* this thread's question; `VK_QCOM_tile_shading` is the
later **vendor-specific** realization (Mali still lacks it). (Couldn't load the
comment thread to confirm whether it cites QCOM.)

**Tone-mapping mapping:**
- *Per-pixel* global tonemap (Reinhard/ACES + uniform exposure) is **already
  solved on-chip by fragment tile access** (PLS / `tile_image` / merge subpass) —
  avoids the full-res HDR DRAM round-trip. No compute needed.
- The compute-on-tile win is the **cooperative** half: **auto-exposure**
  (log-avg luminance / histogram = a reduction, natural for compute + `shared`)
  and **local/spatial tonemapping** (needs neighborhood → tile apron). Doing these
  on tile-resident HDR keeps the HDR buffer off DRAM.
- Caveats: per-tile reductions still need a cross-tile combine; auto-exposure is
  temporal anyway. **Mali fallback:** fragment merge tonemap on-chip + separate
  compute auto-exposure on a *downsampled* luminance buffer.

---

## 7. Open next-steps menu (pick up here locally)

1. apitrace: apply the `GLframebuffer`→`GLenum` fix + (optional) state-dump enums;
   prepare patch against an apitrace fork.
2. v3d: write the Workstream-D **format/storage spike** doc, then skeleton
   `v3d_nir_lower_pls.c` + `v3d_screen.c` caps diff.
3. Extract ARM Translucency's 5 shaders + pass sequence into a portable
   desktop-GLES (EGL/GBM) harness that runs on v3d **and** Panfrost (demo +
   differential oracle).
4. Check piglit for any real PLS tests; design the deterministic unit-test matrix.
5. (Vulkan) check whether `panvk` implements `VK_EXT_shader_tile_image`; sketch
   fused on-chip tonemap+auto-exposure for Adreno (`VK_QCOM_tile_shading`) vs the
   Mali fragment-tile fallback.

---

## 8. Re-clone the sources locally

```bash
# apitrace (PLS specs + the target-type bug)
git clone --depth 1 https://github.com/apitrace/apitrace.git

# Mesa (use a fresh mirror; gitlab.freedesktop.org is the real upstream)
git clone https://gitlab.freedesktop.org/mesa/mesa.git      # preferred locally
# or a GitHub mirror, e.g. https://github.com/mirror/mesa.git

# dEQP / conformance (confirms the testing gap)
git clone --depth 1 --filter=blob:none https://github.com/KhronosGroup/VK-GL-CTS.git

# ARM PLS reference sample (MIT)
git clone --depth 1 https://github.com/ARM-software/opengl-es-sdk-for-android.git
```

Key paths to jump to:
- apitrace: `specs/glapi.py:2605-2608`, `specs/glparams.py:2944-2948`
- Mesa Panfrost: `src/gallium/drivers/panfrost/pan_nir_lower_pls.c`,
  `pan_screen.c:728`, `pan_shader.c:516`
- Mesa v3d: `src/broadcom/compiler/v3d_nir_lower_load_output.c`,
  `src/broadcom/compiler/nir_to_vir.c:1859`, `vir.c:1211`,
  `src/gallium/drivers/v3d/v3d_screen.c:335`
- CTS: `modules/egl/teglGetProcAddressTests.inl:1434`
- ARM: `samples/advanced_samples/Translucency/assets/prepass.fs`
