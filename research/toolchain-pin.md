# Research: Pin the remaining toolchain — compiler, C++ standard, FastNoise2, LZ4

- **Ticket:** [#12 — Pin the remaining toolchain: compiler, C++ standard, FastNoise2, LZ4](https://github.com/acoreyj/voxel/issues/12) (part of map [#1](https://github.com/acoreyj/voxel/issues/1))
- **Date:** 2026-10-09
- **Status:** research complete; recommends a concrete pinned set. **Corrects §6.1 on the Windows runtime** and adds a GCC floor and a FastNoise2 determinism lever that §6.1/§3.5 do not yet mention.
- **Pinned platform** (from [#9](https://github.com/acoreyj/voxel/issues/9)): Godot **4.7** (Forward+), **godot-cpp `10.0.0-stable` = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5`**, built with **`api_version=4.7`**.
- **Spec of record:** `voxel-design-doc.md` §6.1 (Toolchain Specification), §6.2 (Extension Descriptor), §6.3; companion `voxel-decision-log.md` DR-004, DR-010.
- **Related:** `research/nav-api.md` (ticket #11) — the `std::span` signature this note pins the standard around lives in §4.5.

Where a claim is a build-system fact, it was read from the **pinned commit's own build files**, not from prose. Where a claim is a language/library fact, it is cited to cppreference's compiler-support tables. Where a claim is an engine default, it is cited to the `4.7-stable` source. Sources are collected in §9.

## TL;DR

The recommended pinned set:

| Item | Pin | Why |
| :--- | :--- | :--- |
| Godot | `4.7-stable` (from #9) | Pinned by #9 |
| godot-cpp | tag `10.0.0-stable` = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5`, built with `GODOTCPP_API_VERSION=4.7` | #9; option verified in pinned CMake |
| **C++ standard** | **C++20** for `voxel_core` + the wrapper; godot-cpp itself compiles at C++17 | `std::span` in `NavGroupController::note_edit_phase` is C++20 |
| MSVC | **v143+** (Visual Studio 2022, `_MSC_VER` 19.3x+) | Confirmed; matches Godot's recommended Windows toolset |
| Clang | **15+** (Linux/macOS) | Confirmed but **stricter than necessary**; true minimum is far lower (see §2) |
| **GCC** | **11+** (new — the design currently lists no GCC floor) | Covers `std::span` (10) and atomic wait/notify (11) |
| **Windows runtime** | **`/MD` (Release) / `/MDd` (Debug), set globally**; alternative `/MT`/`/MTd` (see §3) | The rule is *one runtime for the whole link*; design says `/MD`/`/MDd` |
| FastNoise2 | **v1.1.1** (conditional: **M1**, pending M0 design ticket #4), built with **`FASTNOISE2_STRICT_FP=ON`** | Latest stable release; strict-FP makes SIMD levels agree bit-for-bit |
| FastSIMD | `16450dae9528727e500e7254f635a671f9c7ee2d` (pulled in by FastNoise2 v1.1.1's CPM) | FastNoise2 v1.1.1 hard-pins this in `src/CMakeLists.txt` |
| LZ4 | **v1.10.0**, vendored (static) | Latest stable release; no transitive deps |
| Build coupling | godot-cpp as a **git submodule**, built by its own CMake via `add_subdirectory`, linked with `godot::cpp` | Enables one `CMAKE_MSVC_RUNTIME_LIBRARY` across all targets |
| CMake | **3.21+** recommended (3.15 absolute minimum; `CMP0091` must be `NEW`) | Required for the runtime-library abstraction |
| Python | **3.8+** at configure/build time | godot-cpp's CMake binding generator |

Key findings beyond the table:

1. **`std::span` forces C++20.** It is the *only* C++20 feature the design needs. The COW refcount, completion queue, and shutdown design use only C++17 `std::atomic` load/store/fetch_add and `std::shared_mutex`; `std::atomic::wait`/`notify` is **not** used anywhere in the current design. See §1.
2. **godot-cpp does not require C++20 and does not cap it.** At the pinned commit godot-cpp sets `CXX_STANDARD 17` on its own target only; it neither propagates a standard requirement nor forbids consumers compiling at C++20. See §1.
3. **godot-cpp's CMake and SCons *default to the static CRT (`/MT`)*, and Godot's own official Windows binaries do too.** The design's `/MD`/`/MDd` therefore must be **explicitly forced before godot-cpp is added**, or it will silently be `/MT`. See §3 — this is the most consequential correction.
4. **FastNoise2's cross-SIMD determinism caveat has a first-party fix.** `FASTNOISE2_STRICT_FP=ON` is documented to "ensure output from different SIMD feature sets match EXACTLY". The design's "accepted for v4" cross-machine variance can be eliminated (at some performance cost) instead of accepted. See §4.
5. **FastNoise2 on Windows should use `clang-cl`, not MSVC.** The project's own README states MSVC "has SIMD compiler bugs, which cause incorrect generation". This interacts with the MSVC v143 decision. See §4.
6. **FastNoise2 pulls FastSIMD over the network at configure time** (`CPMAddPackage`), so a submodule alone is not a hermetic checkout. See §6.
7. **godot-cpp's CMake support is officially "secondary" to SCons.** Recommended anyway, because the design needs one shared runtime library across core + extension. See §6.

---

## 1. C++ standard: use C++20 (forced by `std::span`)

### What the design actually needs

Every standardized-type use in the design and decision log was enumerated:

| Feature | Where | Standard | Needed? |
| :--- | :--- | :--- | :--- |
| `std::span<const …>` | `NavGroupController::note_edit_phase`, `voxel-design-doc.md:1018–1019` | **C++20** | **Yes** |
| `std::atomic<T>` load/store/fetch_add | `SampleBlock::refs`, `Chunk::samples`, `Chunk::version`, `Chunk::refs` (DR-010, §3.1) | C++11 | Yes |
| `std::atomic<bool>` | `shutting_down` (§3.5) | C++11 | Yes |
| `std::shared_mutex` | `Chunk::write_lock`, store lock (§3.1, §3.2) | C++17 | Yes |
| `std::optional`, `std::unique_ptr`, `std::vector` | §6.1 static-link rationale, various interfaces | C++17 | Yes |
| `std::atomic<T>::wait` / `notify_*` | **not present** | C++20 | **No** |
| Designated initializers | **not present** | C++20 | **No** |

`std::span` is C++20: cppreference's C++20 library table lists `<span>` (`P0122R7`) as first available in **libstdc++ GCC 10, libc++ Clang 7, MSVC STL 19.26 (VS 2019 16.6)**. `std::atomic::wait`/`notify_*` (`P1135R6`, "Atomic waiting and notifying") is a *different, later* feature: **GCC 11, Clang 11, MSVC 19.28 (VS 2019 16.9)** — but it is not used, so it does not raise the floor.

**Recommendation: C++20 for `voxel_core` and the extension wrapper. C++17 would only suffice if `std::span` is replaced** (by a small in-repo `Span<T>`, or by `(const T*, size_t)`), which is a normative edit to §4.5. Since the design is the spec of record and uses `std::span`, pin C++20.

### godot-cpp's own standard requirement at the pinned version

At `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5`, `cmake/godotcpp.cmake` sets on its own target only:

```cmake
set_target_properties(godot-cpp PROPERTIES
    CXX_STANDARD 17
    CXX_EXTENSIONS OFF
    ...)
```

There is **no `cxx_std_17` compile feature and no `INTERFACE_CXX_STANDARD`**, so godot-cpp neither requires nor blocks a consumer compiling at C++20. Set the standard per-target in your own CMake:

```cmake
target_compile_features(voxel_core PUBLIC cxx_std_20)   # PUBLIC: wrapper inherits it
target_compile_features(voxel_engine PUBLIC cxx_std_20)
```

`godot-cpp`'s own objects stay C++17; your core + wrapper objects compile at C++20. Mixing C++17 and C++20 translation units is fine on a single toolchain/stdlib; the ABI that matters (Godot's `Ref<T>`/POD marshalling, and the static `voxel_core` link into the extension) is unaffected. MSVC users also get `/Zc:__cplusplus` from godot-cpp (it is a `PUBLIC` compile definition), so `__cplusplus` reports the real standard.

> The design's §6.1 table already says "C++17 minimum; C++20 if `std::atomic`'s wait/notify is used". That condition is wrong on two counts: wait/notify is not used, but **`std::span` is**, and it independently forces C++20. Update the row to: *C++20 (required by `std::span` in `NavGroupController`).*

---

## 2. Compiler floor

### What the design says vs. what is required

The design table pins **MSVC v143+ (Windows), Clang 15+ (Linux/macOS)** and lists **no GCC floor**. Checking each against the features in §1:

| Compiler | Design floor | Minimum for the features used | Verdict |
| :--- | :--- | :--- | :--- |
| MSVC | v143 (VS 2022) | `std::span` needs MSVC STL 19.26 (VS 2019 16.6) | v143 is comfortably above; keep |
| Clang | 15+ | `std::span` needs libc++ 7; atomic wait would need libc++ 11 | **15 is stricter than needed**; safe to keep as a modern baseline |
| GCC | *(absent)* | `std::span` needs libstdc++ 10; atomic wait would need 11 | **Add GCC 11+** |

**MSVC.** `v143` is the Visual Studio 2022 toolset family (`_MSC_VER` 19.30+, MSVC Build Tools 14.3x–14.4x; v142 is VS 2019). Microsoft's compiler-versioning table maps v143 to VS 2022. Godot 4.7's own Windows requirements list **Visual Studio 2019 or later, with Visual Studio 2022 recommended**, and instruct installing "MSVC v143 - VS 2022 C++ build tools". So **MSVC v143+ is confirmed and correct**.

**Clang 15+.** Confirmed safe, but **not the binding constraint**: `std::span` has been in libc++ since Clang 7. Note that on Linux, Clang usually links libstdc++ (GCC's), so `std::span` availability there depends on the *libstdc++* version (GCC 10+), not the Clang version. Keeping Clang 15+ is a reasonable, easily-installed floor; lowering it to the true minimum buys nothing. On macOS, "Clang 15+" corresponds to Xcode 15+ (Godot 4.7 macOS requirements just say "Xcode, or Command Line Tools for Xcode", without a version).

**GCC.** Recommend **GCC 11+**. GCC 10 is technically sufficient for `std::span` alone; GCC 11 adds the atomic wait/notify family and is the safe uniform floor. Godot 4.7's own Linux requirement is only "GCC 9+ / Clang 6+", so the *design's* floor (not Godot's) is the binding one.

### godot-cpp's own compiler constraint

There is none beyond Godot's build prerequisites: godot-cpp's README says to install "the same C++ pre-requisites … required for the `godot` repository". Its CMake requires **CMake 3.10…3.17** and **Python 3.4+** (binding generator); its SCons requires **SCons 4.0+ and Python 3.8+**. In practice the C++20 floor above dominates.

---

## 3. Windows runtime: `/MD` vs `/MDd` — the rule, and a correction

### The rule (why core and extension must match)

Microsoft's `/MD, /MT, /LD` reference states plainly:

> All modules passed to a given invocation of the linker must have been compiled with the same runtime library compiler option (**/MD**, **/MT**, **/LD**).

In this design the **`voxel_core` static library and the extension DLL are linked into one binary** (§6.1: "One `libvoxel_engine.*.dll` per platform, no side-by-side dependencies"). If one is `/MD` and the other `/MT`, the final link fails with `LNK2038: mismatch detected for 'RuntimeLibrary'`. Even where it linked, mixing two CRTs across a DLL boundary is the classic source of cross-boundary `FILE*`/`errno`/`thread_local` bugs. So **core and extension must use the identical runtime-library setting**, and so must any third-party static lib statically linked in (FastNoise2, LZ4).

### How to enforce it in CMake

Use the CMake runtime-library abstraction, not raw flags. `CMAKE_MSVC_RUNTIME_LIBRARY` initializes each target's `MSVC_RUNTIME_LIBRARY` property **as the target is created**, and it has effect only when policy **`CMP0091` is `NEW`**. Set it as a **cache** variable at the top of the project, **before any dependency is added**, with `FORCE` so godot-cpp's own default cannot win:

```cmake
cmake_minimum_required(VERSION 3.21)   # >=3.15 turns CMP0091 ON → runtime abstraction active
project(voxel_engine LANGUAGES CXX)

# One MSVC runtime for core, extension, and every static dep.
# MUST run before add_subdirectory(godot-cpp): godot-cpp's windows.cmake
# does `set(CMAKE_MSVC_RUNTIME_LIBRARY ... CACHE ...)` without FORCE, i.e. it
# only supplies a default and will not overwrite an existing cache entry.
set(CMAKE_MSVC_RUNTIME_LIBRARY
    "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL"   # /MDd in Debug, /MD otherwise
    CACHE STRING "MSVC runtime library" FORCE)
```

CMake documents the default as exactly `MultiThreaded$<$<CONFIG:Debug>:Debug>DLL`, i.e. `/MDd` for Debug and `/MD` otherwise. That matches the design's `/MD` (release) / `/MDd` (debug) wording.

### Correction: the design's `/MD` is *not* what godot-cpp or Godot use by default

At the pinned commit, `cmake/windows.cmake` defaults to the **static** CRT:

```cmake
option(GODOTCPP_USE_STATIC_CPP "Link MinGW/MSVC C++ runtime libraries statically" ON)   # ON
option(GODOTCPP_DEBUG_CRT "Compile with MSVC's debug CRT (/MDd)" OFF)                   # OFF
set(CMAKE_MSVC_RUNTIME_LIBRARY
    "MultiThreaded$<IF:$<BOOL:${GODOTCPP_DEBUG_CRT}>,DebugDLL,$<$<NOT:$<BOOL:${GODOTCPP_USE_STATIC_CPP}>>:DLL>>"
    CACHE STRING "…")
```

With the defaults, the expression reduces to `MultiThreaded` → **`/MT`** in both configs. The SCons path agrees (`tools/windows.py`: `use_static_cpp` default true → `/MT`; `debug_crt` gates `/MDd`). **Godot's own official Windows binaries do the same** — `platform/windows/detect.py` at `4.7-stable` chooses `/MDd` only when `debug_crt` is set, otherwise `/MT` when `use_static_cpp` (default true), else `/MD`.

So the design's `/MD`/`/MDd` is a deliberate divergence from both godot-cpp's default and Godot's shipped binaries. Two internally-consistent options:

- **Option A — honour the design: `/MD` (Release) / `/MDd` (Debug).** Use the snippet above. Consequence: the extension (and any static dep) depends on the **VC++ redistributable** (`vcruntime140.dll`/`msvcp140.dll`).
- **Option B — match the ecosystem: `/MT` (Release) / `/MTd` (Debug).** Set `"MultiThreaded$<$<CONFIG:Debug>:Debug>"` (`/MTd` Debug, `/MT` release) and leave godot-cpp's static default in place. The DLL stays self-contained with no redistributable dependency — which is more consistent with §6.1's "removes the deployment burden entirely" than Option A is.

**Recommendation: pick one explicitly.** The hard requirement is consistency. Given §6.1's stated goal of no side-by-side dependencies, **Option B (`/MT`/`/MTd`) is the better fit for the design's own rationale**, and it is what FastNoise2/LZ4/godot-cpp default to. If the team instead wants to keep the literal `/MD`/`/MDd` text, Option A works but should be recorded as intentionally depending on the VC++ redistributable.

> The design's parenthetical "never mix `/MT` and `/MD` between the core and the extension" is the real invariant and is correct. The specific value is a deployment trade-off, not a correctness one.

Note also that godot-cpp's CMake guidance says: "godot-cpp will set this variable if it isn't already set. So, include it before other dependencies to have the value propagate across the projects." That is the operational rule the snippet implements.

---

## 4. FastNoise2 — conditional (M1), pin v1.1.1, and turn on strict FP

**Status: conditional.** Design ticket **#4 (M0 generation source) is still open** and may keep FastNoise2 out of M0. This section pins it for **M1 readiness** and does not assert that M0 uses it.

### Version

The latest non-prerelease release is **`v1.1.1`** (published 2026-02-24), per the GitHub Releases API. Releases `v1.1.0`, `v1.0.1`, `v1.0.0` precede it; older `v0.x` tags are all marked pre-release. **Pin `v1.1.1`.**

### SIMD dispatch behaviour (the reproducibility caveat)

FastNoise2 uses **FastSIMD** to compile a kernel for each architecture and **select the fastest supported level at runtime**: Scalar, SSE2, SSE4.1, AVX2, AVX512, NEON, WASM SIMD (project README, "Platform Support"). Different SIMD levels can produce slightly different floating-point results. The project's FAQ states it directly:

> FastNoise2 uses SIMD instructions that are automatically selected at runtime based on your CPU's capabilities. Different SIMD levels … can produce slightly different floating point results due to differences in instruction ordering and precision. This output difference is within floating point rounding error.

The design §3.5/§8 currently accepts this ("Cross-machine FastNoise2 determinism. Accepted for v4."). But FastNoise2 v1.1.1 offers **two first-party mitigations**, either of which removes the cross-machine variance:

1. **`FASTNOISE2_STRICT_FP=ON`** (CMake option) — "Enable strict floating point calculations to ensure output from different SIMD feature sets match EXACTLY". In `src/CMakeLists.txt` the OFF path defines `FASTSIMD_RELAXED`, adds `/fp:fast` (MSVC) or `-ffast-math` (GCC/Clang); the ON path skips both. Cost: some throughput.
2. **Lock the dispatch level** when constructing nodes, e.g. `FastNoise::New<FastNoise::Simplex>(FastSIMD::FeatureSet::SSE2)` — fixes a single level across machines.

**Recommendation:** build with **`FASTNOISE2_STRICT_FP=ON`**. It makes generated terrain bit-identical across machines at the cost of performance, which is the right default for a design that already tracks a "generator version" in the region header (§3.6) and lists cross-machine determinism as a risk (§8 item 7). If throughput matters more at M1, keep the option OFF but **record the chosen SIMD FeatureSet in the region header's "generator version"** so regeneration cannot seam against saved edits, and add a test that pins output across at least two SIMD levels.

### Windows correctness caveat

FastNoise2's README warns: *"On Windows using ClangCL is recommended as MSVC has SIMD compiler bugs, which cause incorrect generation. ClangCL also compiles much faster… ClangCL binaries/libraries are fully compatible with MSVC!"* If the project uses **MSVC v143** on Windows (per §6.1), FastNoise2 generation on Windows is a correctness risk. Options: build the whole extension with **`clang-cl`** on Windows (godot-cpp's CMake detects clang-cl and sets the MSVC-compatible flags), or validate MSVC's output against a clang-cl reference and only then trust it.

### CMake integration (submodule + in-tree alias)

FastNoise2 v1.1.1's top-level `CMakeLists.txt` sets `CMAKE_CXX_STANDARD 17`, defaults tools/tests/utility OFF, and **hard-pins FastSIMD** via `CPMAddPackage(... GITHUB_REPOSITORY Auburn/FastSIMD GIT_TAG 16450dae9528727e500e7254f635a671f9c7ee2d)`. In-tree target is `FastNoise`, with `FastNoise2` as an ALIAS; installed consumers get `FastNoise2::FastNoise`.

```cmake
set(FASTNOISE2_STRICT_FP ON  CACHE BOOL "" FORCE)   # cross-SIMD determinism
set(FASTNOISE2_TOOLS     OFF CACHE BOOL "" FORCE)   # no Node Editor executable
set(FASTNOISE2_TESTS     OFF CACHE BOOL "" FORCE)
set(FASTNOISE2_UTILITY   OFF CACHE BOOL "" FORCE)
set(BUILD_SHARED_LIBS    OFF CACHE BOOL "" FORCE)   # static, matches LZ4 too
add_subdirectory(thirdparty/FastNoise2)
target_link_libraries(voxel_core PRIVATE FastNoise2)   # in-tree ALIAS of FastNoise
```

```
git submodule add https://github.com/Auburn/FastNoise2.git thirdparty/FastNoise2
git -C thirdparty/FastNoise2 checkout v1.1.1
```

If consumed via `find_package(FastNoise2 CONFIG)` instead, the imported target is `FastNoise2::FastNoise` (the install export uses namespace `FastNoise2::`). Prefer the submodule to avoid a third-party install step.

**Hermeticity caveat:** `CPMAddPackage` fetches FastSIMD at *configure* time, so the FastNoise2 submodule alone does not make the build offline-reproducible. For a hermetic build, pre-seed the CPM/FetchContent source cache (or vendor FastSIMD at `16450dae…` and point CPM at the local copy). The *pin* is still deterministic; the *fetch* is not offline by default.

---

## 5. LZ4 — pin v1.10.0, vendor it

The latest non-prerelease release is **`v1.10.0`** (published 2024-07-22). The design uses only `LZ4_compress_default` with per-plane compressed lengths in the region TOC (§3.5). LZ4 is a small, dependency-free C library with a stable API/ABI.

**Recommendation: vendor it as a submodule and build it statically** rather than depending on a system `lz4` package. Rationale: (a) reproducible across CI and developer machines with no distro version drift; (b) no `liblz4-dev` requirement; (c) the design already favours static linkage.

LZ4's CMake lives in `build/cmake/`. When added as a subproject it auto-sets `LZ4_BUNDLED_MODE` (no install rules) and forces a static lib; it exposes an interface target `lz4` (which links `lz4_static`). If a system package is preferred instead, `find_package(lz4 CONFIG)` provides the imported target `LZ4::lz4` (export namespace `LZ4::`), or pkg-config's `liblz4`.

```cmake
set(LZ4_BUILD_CLI OFF CACHE BOOL "" FORCE)
add_subdirectory(thirdparty/lz4/build/cmake)
target_link_libraries(voxel_core PRIVATE lz4)   # interface → lz4_static (bundled mode)
```

```
git submodule add https://github.com/lz4/lz4.git thirdparty/lz4
git -C thirdparty/lz4 checkout v1.10.0
```

Build-system note: FastNoise2 and LZ4 both honour the standard `BUILD_SHARED_LIBS` cache variable; setting it OFF once gives static dep libraries for both.

---

## 6. Build-system coupling: submodule + godot-cpp's CMake

**Recommendation: godot-cpp as a git submodule, built by its own CMake via `add_subdirectory`, linked with the `godot::cpp` alias.**

Why CMake and not SCons, despite godot-cpp's docs calling CMake "secondary": the design builds `voxel_core` with CMake (§6.1) and needs the core and the extension compiled with **identical toolchain, flags, and stdlib version** (the whole reason for static linkage). `add_subdirectory(godot-cpp)` puts godot-cpp, the core, and the extension in **one CMake graph** with one `CMAKE_MSVC_RUNTIME_LIBRARY`, which is exactly the invariant §3 requires. An SCons-built godot-cpp would default to `/MT` while a CMake core used `/MD` and the final link would fail `LNK2038`. The trade-off to record: CMake is officially the secondary build system for godot-cpp; if a CMake gap bites, the fallback is SCons for godot-cpp **plus** manually matching the runtime (`use_static_cpp=no` / `debug_crt=yes`) — more fragile.

At the pinned commit, godot-cpp's CMake:

- defines the target `godot-cpp` and alias `godot::cpp` (use the alias);
- takes the API version as the cache option **`GODOTCPP_API_VERSION`** (allowed values `4.3;4.4;4.5;4.6;4.7`), which selects `gdextension/extension_api-4-7.json`;
- runs the binding generator at configure/build time via **Python 3** and `binding_generator.py`;
- has **no install/export and no `find_package(godot-cpp)`** — consumption is `add_subdirectory`/FetchContent only.

**Submodule + `add_subdirectory` (recommended):**

```
git submodule add https://github.com/godotengine/godot-cpp.git thirdparty/godot-cpp
git -C thirdparty/godot-cpp checkout 507ed9d840c01a3c5b2a39af8bb4000bfac30bf5
```

```cmake
set(GODOTCPP_API_VERSION "4.7" CACHE STRING "" FORCE)   # → extension_api-4-7.json
add_subdirectory(thirdparty/godot-cpp)
target_link_libraries(voxel_engine PRIVATE voxel_core godot::cpp)
```

**FetchContent alternative** (pin the same commit; set the API version *before* `MakeAvailable`):

```cmake
include(FetchContent)
set(GODOTCPP_API_VERSION "4.7" CACHE STRING "" FORCE)
FetchContent_Declare(godot-cpp
    GIT_REPOSITORY https://github.com/godotengine/godot-cpp.git
    GIT_TAG        507ed9d840c01a3c5b2a39af8bb4000bfac30bf5)
FetchContent_MakeAvailable(godot-cpp)
target_link_libraries(voxel_engine PRIVATE voxel_core godot::cpp)
```

Submodule is preferred over FetchContent for the design's reproducibility goal: the pinned tree is visible in `git submodule status`, and CI can cache it, whereas FetchContent re-clones at configure time. (With the SCons path the equivalent option is `scons api_version=4.7`, or passing `{"api_version": "4.7"}` to godot-cpp's `SConstruct` as the README recommends.)

**Prerequisites:** CMake ≥ 3.15 (use 3.21+), Python 3.8+, Git, and a platform build tool (MSVC/Ninja/Xcode).

---

## 7. Concrete corrections to the design / decision log

| Where | Current text | Correction |
| :--- | :--- | :--- |
| §6.1 Toolchain table — C++ standard | "C++17 minimum; C++20 if `std::atomic`'s wait/notify is used" | **C++20**, forced by `std::span` in §4.5. wait/notify is not used. |
| §6.1 Toolchain table — Compiler | "MSVC v143 or later (Windows), Clang 15+ (Linux/macOS)" | Add **GCC 11+**. Keep MSVC v143+, Clang 15+ (both confirmed, Clang is stricter than needed). |
| §6.1 Toolchain table — Runtime (Windows) | "`/MD` (release) or `/MDd` (debug) …" | Keep the *consistency* invariant, but note godot-cpp **and** Godot default to `/MT`; choose `/MD`/`/MDd` **or** `/MT`/`/MTd` and force it before godot-cpp. §6.1's "no side-by-side dependencies" goal favours `/MT`/`/MTd`. |
| §6.1 Toolchain table — Godot version | "4.2 or later, matching the godot-cpp branch" | Stale: #9 pinned **4.7** / godot-cpp **10.0.0-stable**. |
| §6.2 / §6.3 | `compatibility_minimum = "4.2"`, "Godot version 4.2" | Stale: #9/#11 pin `compatibility_minimum = "4.7"`. |
| §3.5 / §8 item 7 | "Cross-machine FastNoise2 determinism. Accepted for v4." | Reconsider: `FASTNOISE2_STRICT_FP=ON` removes the variance. Decide M1 and record in the generator-version header field. |
| §6.1 Toolchain table | FastNoise2 "Version pinned; note the SIMD-level behavior in tests" | Pin **v1.1.1**; prefer `clang-cl` on Windows (MSVC SIMD bugs); note CPM fetches FastSIMD `16450dae…`. |
| §6.1 Toolchain table | LZ4 "Version pinned" | Pin **v1.10.0**, vendored static. |
| §3.1 Storage index | "for example `absl::flat_hash_map` or `robin_hood::unordered_flat_map`" | **Not covered by this ticket but not yet pinned.** A third dependency for reproducible builds; `robin_hood` is header-only (easy to vendor), Abseil is larger and C++17. Flag as an open pin. |

---

## 8. What could not be verified from primary sources

- **The design's rationale for `/MD` over `/MT`.** The rule (consistency) is verified; the *value* is not justified in the design. The recommendation to consider `/MT`/`/MTd` is an inference from §6.1's own "no side-by-side dependencies" goal, not a documented requirement.
- **FastNoise2's `FASTNOISE2_STRICT_FP` performance cost.** The option's semantics are primary-sourced (CMake option help + `src/CMakeLists.txt`), but the throughput penalty is not quantified by any primary source; measure at M1.
- **Exact clang-cl behaviour of MSVC's SIMD bug** ("MSVC has SIMD compiler bugs, which cause incorrect generation"). This is the project's own README warning; the specific affected nodes/SIMD levels are not enumerated in the primary sources read. Treat as a caution and validate against a clang-cl reference.
- **Whether CPM honours a local FastSIMD override for fully offline builds.** `CPMAddPackage` uses FetchContent under the hood, so a `FETCHCONTENT_SOURCE_DIR_*`/CPM source cache *should* work, but the exact variable name for this FastNoise2/FastSIMD pair was not verified from source. Verify in M1 if hermetic offline builds are required.
- **A specific minimum Clang/macOS Xcode version.** Godot 4.7 docs give no version; the "Clang 15+" floor is the design's, not a documented dependency. Apple Clang's `std::span` support has existed since Apple Clang 10.
- **Godot's own macOS/Linux official binaries' runtime flags.** Only the Windows `/MT` default was read from `4.7-stable` source; the macOS/Linux runtime-library question is N/A (`libstdc++`/`libc++`/system linker), so no claim is made there.
- **The `absl`/`robin_hood` hash-map pin** (§3.1) is outside this ticket's scope and remains open.

---

## 9. Sources

**godot-cpp — pinned `10.0.0-stable` = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5`:**

- Tag → commit: <https://github.com/godotengine/godot-cpp/releases/tag/10.0.0-stable> (verified via `gh api repos/godotengine/godot-cpp/git/ref/tags/10.0.0-stable`)
- `CMakeLists.txt` (project version 10.0, test default API 4.7): <https://github.com/godotengine/godot-cpp/blob/507ed9d840c01a3c5b2a39af8bb4000bfac30bf5/CMakeLists.txt>
- `cmake/godotcpp.cmake` (`GODOTCPP_API_VERSION`, `CXX_STANDARD 17`, `godot::cpp` alias, `add_library(godot-cpp STATIC)`): <https://github.com/godotengine/godot-cpp/blob/507ed9d840c01a3c5b2a39af8bb4000bfac30bf5/cmake/godotcpp.cmake>
- `cmake/windows.cmake` (MSVC runtime default, clang-cl note): <https://github.com/godotengine/godot-cpp/blob/507ed9d840c01a3c5b2a39af8bb4000bfac30bf5/cmake/windows.cmake>
- `cmake/common_compiler_flags.cmake` (clang-cl detection, `/Zc:__cplusplus`): <https://github.com/godotengine/godot-cpp/blob/507ed9d840c01a3c5b2a39af8bb4000bfac30bf5/cmake/common_compiler_flags.cmake>
- `SConstruct` (SCons 4.0 / Python 3.8), `tools/windows.py` (`use_static_cpp`/`debug_crt` → `/MT`//`/MDd`): <https://github.com/godotengine/godot-cpp/blob/507ed9d840c01a3c5b2a39af8bb4000bfac30bf5/tools/windows.py>
- `README.md` (`api_version`, "same C++ pre-requisites as godot"): <https://github.com/godotengine/godot-cpp/blob/507ed9d840c01a3c5b2a39af8bb4000bfac30bf5/README.md>
- `cmake/GodotCPPModule.cmake` (Python 3 binding generator; no install/export): <https://github.com/godotengine/godot-cpp/blob/507ed9d840c01a3c5b2a39af8bb4000bfac30bf5/cmake/GodotCPPModule.cmake>
- Official CMake guide (secondary build system; "set `CMAKE_MSVC_RUNTIME_LIBRARIES` … before other dependencies"): <https://docs.godotengine.org/en/latest/tutorials/scripting/cpp/build_system/cmake.html>

**Godot engine (`4.7-stable`):**

- `platform/windows/detect.py` (`configure_msvc`: `/MDd` iff `debug_crt`, else `/MT` iff `use_static_cpp` (default true), else `/MD`): <https://github.com/godotengine/godot/blob/4.7-stable/platform/windows/detect.py>
- Compiling for Windows (VS 2019 or later; VS 2022 recommended; MSVC v143): <https://docs.godotengine.org/en/4.7/engine_details/development/compiling/compiling_for_windows.html>
- Compiling for Linux/\*BSD (GCC 9+ / Clang 6+): <https://docs.godotengine.org/en/4.7/engine_details/development/compiling/compiling_for_linuxbsd.html>
- Compiling for macOS (Xcode or CLT, no version): <https://docs.godotengine.org/en/4.7/engine_details/development/compiling/compiling_for_macos.html>

**Microsoft / CMake:**

- `/MD, /MT, /LD` — "All modules passed to a given invocation of the linker must have been compiled with the same runtime library compiler option": <https://learn.microsoft.com/en-us/cpp/build/reference/md-mt-ld-use-run-time-library>
- MSVC compiler versioning (v143 = VS 2022; v142 = VS 2019): <https://learn.microsoft.com/en-us/cpp/overview/compiler-versions>
- CMake `CMAKE_MSVC_RUNTIME_LIBRARY` (allowed values; default `MultiThreaded$<$<CONFIG:Debug>:Debug>DLL`): <https://cmake.org/cmake/help/latest/variable/CMAKE_MSVC_RUNTIME_LIBRARY.html>
- CMake `CMP0091` (runtime-library abstraction; must be `NEW` before `project()`): <https://cmake.org/cmake/help/latest/policy/CMP0091.html>

**cppreference — C++20 compiler support:**

- `std::span` (GCC 10 / Clang 7 / MSVC 19.26), atomic wait/notify-family (GCC 11 / Clang 11 / MSVC 19.28), designated initializers: <https://en.cppreference.com/w/cpp/compiler_support/20>

**FastNoise2:**

- Release `v1.1.1` (latest stable, 2026-02-24): <https://github.com/Auburn/FastNoise2/releases/tag/v1.1.1>
- Top-level `CMakeLists.txt` (v1.1.1, C++17, tools/tests options): <https://github.com/Auburn/FastNoise2/blob/v1.1.1/CMakeLists.txt>
- `src/CMakeLists.txt` (`FastNoise` target + `FastNoise2` alias, `FASTNOISE2_STRICT_FP` → `FASTSIMD_RELAXED`/`-ffast-math`/`/fp:fast`, FastSIMD CPM pin `16450dae…`): <https://github.com/Auburn/FastNoise2/blob/v1.1.1/src/CMakeLists.txt>
- README (FastSIMD runtime dispatch levels; MSVC SIMD bugs → prefer clang-cl): <https://github.com/Auburn/FastNoise2/blob/v1.1.1/README.md>
- FAQ ("My noise outputs vary slightly on different machines"; strict FP / lock SIMD level): <https://github.com/Auburn/FastNoise2/wiki/FAQ>

**LZ4:**

- Release `v1.10.0` (latest stable, 2024-07-22): <https://github.com/lz4/lz4/releases/tag/v1.10.0>
- `build/cmake/CMakeLists.txt` (`lz4` interface target, bundled→static, `LZ4::` export namespace): <https://github.com/lz4/lz4/blob/v1.10.0/build/cmake/CMakeLists.txt>

**Reproducibility:** every version above is a tag or commit resolvable by Git; godot-cpp and FastNoise2/LZ4 facts were read from the pinned trees, and Godot facts from the `4.7-stable` tag.
