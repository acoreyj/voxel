# Research: Godot 4.2 vs 4.7 for the engine's GDExtension target

- **Ticket:** [#2 — Survey Godot 4.2 vs 4.7 for the engine's needs](https://github.com/acoreyj/voxel/issues/2) (part of map [#1](https://github.com/acoreyj/voxel/issues/1))
- **Date:** 2026-10-08
- **Status:** research complete; decision belongs to [#9](https://github.com/acoreyj/voxel/issues/9)
- **Spec of record:** `voxel-design-doc.md` (v4), §4.1, §4.5, §6.1–6.3, §8.6; companion `voxel-decision-log.md` (L-02, L-03)

## TL;DR

1. **Godot 4.2 is not viable and `compatibility_minimum = "4.2"` must be raised.** The design's own §4.5 synchronization contract stores a `submitted_map_iteration` and polls "the map synchronization/iteration condition". The API that exposes that value, `NavigationServer3D.map_get_iteration_id()`, **does not exist in 4.2**; it first appears in **4.3**. `is_baking_navigation_mesh()` is also 4.3+. This is exactly the M0 verification the design anticipated in §6.2 and §8.6, and it fails at 4.2.
2. **godot-cpp no longer has a per-Godot-version branch after 4.5.** Since v10, godot-cpp is versioned independently and one checkout targets Godot 4.3–4.7 through an `api_version` build variable. Godot 4.6/4.7 are supported by **godot-cpp v10**, not by a `4.6`/`4.7` branch.
3. **The Jolt, D3D12, and `.NET` settings in `project.godot` are all explainable**, and only the first two bear on the version pin:
   - Jolt is built into Godot from **4.4** (experimental) and is the **default from 4.6**; the project setting value `"Jolt Physics"` is valid from 4.4 onward.
   - D3D12 support predates 4.3 but only became painless when the DXIL validator was open-sourced in **4.3**; it is the **default on Windows from 4.6**.
   - `[dotnet] project/assembly_name` is a **C# assembly name**, unrelated to a C++ GDExtension; it is noise and should be removed.
4. **Recommendation (primary):** pin **Godot 4.7.x stable (4.7.2)** with **godot-cpp `10.0.0-stable` (`507ed9d8`) built with `api_version=4.7`**, and set **`compatibility_minimum = "4.7"`**.
   **Recommendation (conservative alternative, if a wider runtime range matters):** pin **Godot 4.5.x stable (4.5.2)** with **godot-cpp `godot-4.5-stable` (`e83fd090`)**, `compatibility_minimum = "4.5"`. GDExtension compatibility is one-directional, so a 4.5-built extension also runs on 4.6/4.7, whereas a 4.7-built extension does not run on 4.5/4.6.

Either way, **4.2 is out**. The rest of this note is the evidence.

---

## 1. Current stable Godot versions and their godot-cpp counterparts

Godot 4.x stable lines and the latest patch in each (from the Godot git tags; `4.7.2-stable` is the newest overall):

| Godot line | Latest stable patch | godot-cpp ref | `compatibility_minimum` the build can declare |
| :--- | :--- | :--- | :--- |
| 4.0 | 4.0.4 | `godot-cpp` tag `godot-4.0.4-stable` (branch `4.0`) | 4.1+ (Godot rejects `< 4.1.0`; see §3.4) |
| 4.1 | 4.1.4 | tag `godot-4.1.4-stable` (branch `4.1`) | 4.1+ |
| 4.2 | 4.2.2 | tag `godot-4.2.2-stable` (branch `4.2`) | 4.1 – 4.2 |
| 4.3 | 4.3 | tag `godot-4.3-stable` (branch `4.3`) | 4.1 – 4.3 |
| 4.4 | 4.4.1 | tag `godot-4.4.1-stable` (branch `4.4`) | 4.1 – 4.4 |
| 4.5 | 4.5.2 | tag `godot-4.5-stable` (branch `4.5`) | 4.1 – 4.5 |
| 4.6 | 4.6.3 | **no dedicated ref** — godot-cpp v10 with `api_version=4.6` | 4.1 – 4.6 |
| 4.7 | **4.7.2** | **no dedicated ref** — godot-cpp v10 with `api_version=4.7` | 4.1 – 4.7 |

Sources:
- Godot tags: <https://github.com/godotengine/godot/tags> — `4.7.2-stable` = `ed1daf0b`, `4.7-stable` = `5b4e0cb0`, `4.5.2-stable` = `6ce3de25` (verified with `git ls-remote https://github.com/godotengine/godot`).
- godot-cpp branches end at `4.5`: <https://github.com/godotengine/godot-cpp/branches>. The branch `4.5` is at `27d9dd23`.
- godot-cpp per-Godot tags end at `godot-4.5-stable`: <https://github.com/godotengine/godot-cpp/tags>. `godot-4.5-stable` = `e83fd090`.

### 1.1 godot-cpp changed versioning at v10 (the single most important plumbing fact)

From the godot-cpp README, *Versioning* and *Compatibility* sections (<https://github.com/godotengine/godot-cpp/blob/master/README.md>):

> Starting with version 10.x, godot-cpp is versioned independently from Godot. Using the `api_version` parameter (see below), godot-cpp v10 can target Godot 4.3 or later.
>
> GDExtensions targeting an earlier version of Godot should work in later minor versions, but not vice-versa. For example, a GDExtension targeting Godot 4.3 should work just fine in Godot 4.4, but one targeting Godot 4.4 won't work in Godot 4.3.

The v10 release ships one API dump per supported Godot minor — `gdextension/extension_api-4-3.json` … `extension_api-4-7.json` — and refuses any other value (`tools/godotcpp.py`):
<https://github.com/godotengine/godot-cpp/blob/10.0.0-stable/tools/godotcpp.py>

```python
supported_api_versions = ["4.3", "4.4", "4.5", "4.6", "4.7"]
...
filename = "extension_api-%s.json" % api_version.replace(".", "-")
```

The v10 release is `10.0.0-stable` = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5` (published 2026-09-15; the release notes explicitly say it is no longer beta):
- <https://github.com/godotengine/godot-cpp/releases/tag/10.0.0-stable>
- <https://github.com/godotengine/godot-cpp/blob/10.0.0-stable/gdextension/README.md>

Consequence for the design's §6.1 table ("godot-cpp | Exact commit or tag recorded in the submodule"): for a 4.7 target the pinned ref is the **tag `10.0.0-stable`** plus the build variable **`api_version=4.7`** (or `custom_api_file`), not a `4.7` branch.

---

## 2. GDExtension API differences 4.2 → 4.7 that matter to this design

All signatures below are read from the GDExtension API dumps that godot-cpp generates from Godot and commits to the repo:
- 4.2: <https://raw.githubusercontent.com/godotengine/godot-cpp/godot-4.2-stable/gdextension/extension_api.json>
- 4.3–4.7: <https://github.com/godotengine/godot-cpp/tree/master/gdextension> (`extension_api-4-3.json` … `extension_api-4-7.json`)

### 2.1 `RenderingServer::mesh_add_surface` vs `mesh_add_surface_from_arrays`, and custom vertex attributes

**There is no raw-attribute-view overload in the GDExtension API at either version.** 4.2 and 4.7 expose exactly two ways to add a surface:

```
mesh_add_surface_from_arrays(mesh: RID, primitive: PrimitiveType, arrays: Array,
                             blend_shapes: Array = [], lods: Dictionary = {},
                             compress_format: BitField[ArrayFormat] = 0)
mesh_add_surface(mesh: RID, surface: Dictionary)
```

The second form is the design's hoped-for "lower level" path: internally it takes a serialized `RS::SurfaceData` and is converted by `_dict_to_surf`. But in GDExtension it is a **`Dictionary`**, not a C++ `SurfaceData&` view — the bound C++ `mesh_add_surface(RID, const RS::SurfaceData &)` is not exposed. See:

- 4.2: `servers/rendering_server.cpp` — `_mesh_add_surface` at line 1986 and `_dict_to_surf` at line 1916, <https://github.com/godotengine/godot/blob/4.2-stable/servers/rendering_server.cpp>
- 4.7: `servers/rendering/rendering_server.cpp` — `_mesh_add_surface` at line 2003 and `_dict_to_surf` at line 1933, <https://github.com/godotengine/godot/blob/4.7-stable/servers/rendering/rendering_server.cpp>

The `_dict_to_surf` key set is **byte-for-byte the same in 4.2 and 4.7**: `primitive`, `format`, `vertex_data`, `attribute_data`, `skin_data`, `vertex_count`, `index_data`, `index_count`, `aabb`, `uv_scale`, `lods`, `bone_aabbs`, `blend_shape_data`, `material`. So the dictionary form carries the already-packed vertex/attribute byte buffers and a `format` bitfield, avoiding the three intermediate `Packed*Array`s — at the cost of the caller owning the interleaved byte layout and the format bits. The design's §4.1 note is correct that this is the lower-level escape hatch, and it exists at **both** 4.2 and 4.7.

**Per-vertex material channel as a custom attribute is supported at both versions.** `Mesh.ArrayCustomFormat` and the `ARRAY_CUSTOM0..3` slots exist in 4.2 and 4.7 (values `ARRAY_CUSTOM_RGBA8_UNORM` … `ARRAY_CUSTOM_RGBA_FLOAT`, plus `ARRAY_FORMAT_CUSTOM0..3` and `ARRAY_FORMAT_CUSTOM_BASE`/`BITS`/`MASK`). The format is packed into the surface `format` bitfield. The 4.2 source already implements the `ARRAY_CUSTOM*` encode/decode path (`rendering_server.cpp` lines 780–825, 1094–1121), so the design's "pass the material channel as a custom attribute" option does **not** require 4.7. It is decided by shader/format choice, not by Godot version.

Surface-lifecycle differences that do matter:

| Method | 4.2 | 4.3 | 4.4 | 4.5 | 4.6 | 4.7 |
| :--- | :-: | :-: | :-: | :-: | :-: | :-: |
| `mesh_add_surface` (Dictionary) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_add_surface_from_arrays` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_get_surface` / `mesh_get_surface_count` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_set_material` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_update_vertex_region` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_remove` | – | – | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_update_index_region` | – | – | – | ✓ | ✓ | ✓ |

Practical reading: at 4.2/4.3, replacing a chunk's surface requires `mesh_clear` (clears all surfaces) or recreating the mesh `RID`. `mesh_surface_remove` (4.4+) and `mesh_surface_update_index_region` (4.5+) make per-surface replacement and partial re-upload possible. None of these are listed in the design, so they are gains, not blockers — but they are relevant to the M0 commit-loop spike (ticket [#10](https://github.com/acoreyj/voxel/issues/10)).

### 2.2 `NavigationServer3D` async bake, synchronization, and map iteration

This is where 4.2 fails the design. Presence of the relevant methods:

| Method | 4.2 | 4.3 | 4.4 | 4.5 | 4.6 | 4.7 |
| :--- | :-: | :-: | :-: | :-: | :-: | :-: |
| `bake_from_source_geometry_data` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `bake_from_source_geometry_data_async` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `parse_source_geometry_data` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `map_get_iteration_id` | **–** | ✓ | ✓ | ✓ | ✓ | ✓ |
| `is_baking_navigation_mesh` | **–** | ✓ | ✓ | ✓ | ✓ | ✓ |
| `source_geometry_parser_create` | – | ✓ | ✓ | ✓ | ✓ | ✓ |
| `region_get_iteration_id` | – | – | – | ✓ | ✓ | ✓ |
| `map_set_use_async_iterations` | – | – | – | ✓ | ✓ | ✓ |
| `region_set_use_async_iterations` | – | – | – | ✓ | ✓ | ✓ |

Key findings:

- **Async bake itself is available at 4.2.** Both `bake_from_source_geometry_data` and `bake_from_source_geometry_data_async` are present in the 4.2 API dump. So the design's phrase "rely on asynchronous `NavigationServer3D` APIs" is true even at 4.2. The blocker is **not** the bake; it is the *synchronization query*.
- **`map_get_iteration_id` is the missing 4.2 piece.** The design's `NavigationGroupState` stores `submitted_map_iteration` and its §4.5 text says "the wrapper polls the pinned Godot navigation synchronization state". The official 4.7 class reference documents exactly the needed semantics:
  > `int map_get_iteration_id(map: RID) const` — Returns the current iteration id of the navigation map. Every time the navigation map changes and synchronizes the iteration id increases. An iteration id of 0 means the navigation map has never synchronized. Note: The iteration id will wrap back to 1 after reaching its range limit.
  > — <https://docs.godotengine.org/en/stable/classes/class_navigationserver3d.html>
  It is absent from the 4.2 dump, so the design's sync contract cannot be implemented as written at 4.2. `is_baking_navigation_mesh` (also 4.3+) is the complementary "is a bake still in flight" query.
- **`region_get_iteration_id` first appears in 4.5.** This gives per-region synchronization instead of map-global iteration, which is a better fit for the design's per-group `synced_*` watermark than the map-global id the design currently names. `map_set_use_async_iterations` / `region_set_use_async_iterations` (4.5) expose the background-thread iteration mode described in the 4.5 release notes ("Enabling async iterations asks the navigation servers to delegate the navigation process to a background thread" — <https://godotengine.org/releases/4.5/>).
- `source_geometry_parser_create` (4.3+) allows a custom runtime source-geometry parser; `parse_source_geometry_data` exists at 4.2. Relevant to whether `navigation_mesh` can be fed a runtime triangle source (ticket [#11](https://github.com/acoreyj/voxel/issues/11)).

**Minimum version implied by the design:** 4.3 for the sync contract as written (map iteration + is-baking); **4.5** if the stronger per-region iteration sync is desired.

### 2.3 Jolt physics "class names"

There are **no `Jolt*` classes in the GDExtension API in any of 4.2–4.7** (verified by scanning all class names for `Jolt` in every dump). Jolt is not a set of GDExtension classes; it is a `PhysicsServer3D` backend selected by the project setting `physics/3d/physics_engine`. The official Jolt page (<https://docs.godotengine.org/en/stable/tutorials/physics/using_jolt_physics.html>) says:

> The Jolt physics engine was added as an alternative to the existing Godot Physics physics engine in **4.4**. … Previously it was available as an extension but is now built into Godot. By default, new projects will use it as the physics engine.
>
> To change the 3D physics engine to be Jolt Physics, set **Project Settings > Physics > 3D > Physics Engine** to **Jolt Physics**.

So:
- The design's collision path — `ConcavePolygonShape3D` on a `StaticBody3D`, driven through `PhysicsServer3D` — is **backend-agnostic** and unaffected by choosing Jolt. There is no Jolt-specific class to bind.
- `project.godot`'s `physics/3d/physics_engine="Jolt Physics"` is valid **only from 4.4**. At 4.2/4.3 that string has no meaning (the external `godot-jolt` GDExtension was the only route). This alone pushes the floor to 4.4 if the existing project setting is to be kept.
- Jolt became the default in 4.6 (release notes: "Remember when we integrated Jolt Physics as an experimental option in 4.4? … Jolt Physics by default" — <https://godotengine.org/releases/4.6/>).

### 2.4 `.gdextension` descriptor format

The design's §6.2 descriptor uses `[configuration] entry_symbol`, `compatibility_minimum`, and `[libraries]`, with no `[dependencies]`. That shape is valid across 4.2–4.7; only optional keys were added. From the official `.gdextension file` page (<https://docs.godotengine.org/en/4.6/tutorials/scripting/gdextension/gdextension_file.html>) and the loader source:

| Key | Availability | Notes |
| :--- | :--- | :--- |
| `entry_symbol` | all | required (`GDExtension configuration file must contain a "configuration/entry_symbol" key`) |
| `compatibility_minimum` | 4.1+ | required; **must be ≥ 4.1.0**, or the engine refuses to load |
| `compatibility_maximum` | 4.3+ | optional; "Only supported in Godot 4.3 or later" |
| `reloadable` | 4.2+ | "Reloading is supported for the godot-cpp binding in Godot 4.2 or later" |
| `android_aar_plugin` | 4.4-ish | Android packaging |
| `[libraries]` | all | per-platform/feature library paths |
| `[dependencies]` | 4.3+ docs section | for dependent shared libraries; **not used** here (core is statically linked) |

Source: `core/extension/gdextension_library_loader.cpp` at 4.7 (config parsing moved out of `gdextension.cpp` in 4.4), <https://github.com/godotengine/godot/blob/4.7-stable/core/extension/gdextension_library_loader.cpp>. The 4.2 equivalent is `core/extension/gdextension.cpp`, <https://github.com/godotengine/godot/blob/4.2-stable/core/extension/gdextension.cpp> (same `entry_symbol`/`compatibility_minimum` parsing, minimum 4.1.0).

No descriptor change is needed to move from 4.2 to 4.7 other than raising the `compatibility_minimum` value itself.

---

## 3. Does targeting 4.7 break the design's "Godot 4.2 or later" assumptions?

**No.** 4.7 satisfies "4.2 or later", and §1.1 of the design already says "Godot 4.x". But three spec statements become stale and must change:

- §6.2: `compatibility_minimum = "4.2"` → the pinned version.
- §6.3: "Godot version | 4.2 stable" → the pinned version.
- §8.6 / §4.5: the asynchronous-navigation-API caution is answered — the APIs exist, but only from 4.3 (map iteration) / 4.5 (per-region).

**What 4.7 gains for this design:**

- All navigation sync APIs (`map_get_iteration_id`, `is_baking_navigation_mesh`, `region_get_iteration_id`, async iterations).
- Jolt built in and default; `project.godot`'s value becomes meaningful.
- D3D12 mature and default on Windows (4.6), vs the pre-4.3 version that required shipping the proprietary `DXIL.dll` (4.3 release notes: "Until now, you had to build a custom Godot version yourself and include the DXIL.dll library in your project in order to use Direct3D 12" — <https://godotengine.org/releases/4.3/>).
- `mesh_surface_remove` (4.4) and `mesh_surface_update_index_region` (4.5) for replace/partial-update surface commits.
- GDExtensions listed in Project Settings (4.7) — quality of life only.

**Costs/risks of 4.7:**

- Raises the minimum runtime to 4.7 if built against the 4.7 API; older 4.x users cannot load the extension.
- godot-cpp v10 must be built with `api_version=4.7` (a build-system variable to pin and record, replacing the old "branch = Godot minor" mental model).
- The design's M0 nav verification (ticket #11) still has to prove the exact submission/sync sequence at the chosen version; the version bump does not by itself implement it.

---

## 4. Is the `.NET`/Mono setup relevant to a pure C++ GDExtension?

**No — remove it.** Findings:

- godot-cpp is "the C++ bindings for the Godot Engine's GDExtensions API" (<https://github.com/godotengine/godot-cpp/blob/master/README.md>). It builds with SCons or CMake and has no .NET dependency; the `[libraries]` it produces are native `.dll`/`.so`/`.dylib`.
- `dotnet/project/assembly_name` is a **C# project setting**. The Godot docs list it under the `.NET` properties (`dotnet/project/assembly_name`, `dotnet/project/assembly_reload_attempts`, `dotnet/project/solution_directory`) — <https://docs.godotengine.org/en/4.5/classes/class_projectsettings.html>. It names the managed assembly, which only exists for a C# project.
- C# support is not even in the standard editor: "The standard Godot executable does not contain C# support out of the box. Instead, to enable C# support for your project you need to download a .NET version of the editor from the Godot website." (<https://docs.godotengine.org/en/4.6/tutorials/scripting/c_sharp/index.html>).
- A C++ GDExtension loads in the standard (non-.NET) editor and in exports built from it. Keeping the `[dotnet]` section neither enables nor disables anything for the extension; it just signals an unused C# toolchain.

**Conclusion:** the `[dotnet] project/assembly_name="voxel"` block in `project.godot` is leftover noise from `config/features=("4.7", ...)`. Delete it. (The design's §6.3 greenfield project table does not include it.)

---

## 5. Recommendation and required changes

### 5.1 Primary recommendation

| Item | Value |
| :--- | :--- |
| **Godot** | **4.7.x stable — pin 4.7.2** (`4.7.2-stable` = `ed1daf0b`) |
| **godot-cpp** | **v10 tag `10.0.0-stable`** = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5` |
| **godot-cpp build target** | **`api_version=4.7`** (selects `gdextension/extension_api-4-7.json`) |
| **`compatibility_minimum`** | **`"4.7"`** |
| **`compatibility_maximum`** | omit (or `"4.x"`-equivalent; optional, 4.3+) |
| **`project.godot` `config/features`** | `PackedStringArray("4.7", "Forward Plus")` — keep |
| **Jolt** | keep `physics/3d/physics_engine="Jolt Physics"` (valid 4.4+) |
| **D3D12** | keep `rendering_device/driver.windows="d3d12"` (default on Windows from 4.6) |
| **`[dotnet]`** | remove |

Why 4.7: it is the current stable line; the project already declares 4.7; godot-cpp v10 is the current bindings release and explicitly ships and supports `extension_api-4-7.json`; every navigation/rendering API the design names exists there; and Jolt/D3D12 are at their most mature.

### 5.2 Conservative alternative (if a wider supported Godot range is a goal)

Pin **Godot 4.5.2** (`4.5.2-stable` = `6ce3de25`) with **godot-cpp `godot-4.5-stable`** = `e83fd090`, and `compatibility_minimum = "4.5"`. This is the **lowest version that satisfies all hard requirements**:

- `map_get_iteration_id` (4.3), `is_baking_navigation_mesh` (4.3), `region_get_iteration_id` (4.5), async nav iterations (4.5);
- built-in Jolt selectable as `"Jolt Physics"` (4.4);
- D3D12 without the DXIL headache (4.3+);
- identical `RenderingServer` surface API and custom attributes.

Because a 4.5-built extension also runs on 4.6/4.7, this maximizes the runtime range. It uses the last dedicated godot-cpp per-Godot branch, so no `api_version` variable is required at all. The cost is that the build is *tested* against 4.5 APIs, not 4.7; if a 4.6/4.7-only API is later needed, the floor moves up.

### 5.3 Out of scope / left to the M0 spikes

- The exact `RenderingServer` surface call sequence (array form vs dictionary form) and RID lifecycle: ticket [#10](https://github.com/acoreyj/voxel/issues/10).
- The exact `NavigationServer3D` submission/sync sequence and whether `region_get_iteration_id` replaces the map-level iteration in the design's `NavigationGroupState`: ticket [#11](https://github.com/acoreyj/voxel/issues/11).
- These are unchanged by the version choice beyond the availability facts above.

### 5.4 Verification status

Everything above is from primary sources (the Godot/godot-cpp repositories and generated API dumps, plus official Godot docs and release notes). Two items could not be confirmed from source and are called out rather than guessed:

- The **4.7 documentation page** for the `.gdextension` file is not published under the stable docs URL in this environment (it 404s); the 4.6 page plus the 4.7 loader source are used instead. The format itself is stable across 4.2–4.7, so this does not affect the recommendation.
- **When exactly `compatibility_maximum` entered the parser** is documented by Godot as "Godot 4.3 or later"; the source inspection confirms it is absent in 4.2 and present by 4.4. It is optional and unused by the design, so this is immaterial.

---

## 6. Sources

Primary (repository / generated API / official docs):

- Godot tags & releases: <https://github.com/godotengine/godot/tags> · <https://github.com/godotengine/godot/releases>
- godot-cpp README (versioning/compatibility): <https://github.com/godotengine/godot-cpp/blob/master/README.md>
- godot-cpp release `10.0.0-stable`: <https://github.com/godotengine/godot-cpp/releases/tag/10.0.0-stable>
- godot-cpp supported API versions & build variable: <https://github.com/godotengine/godot-cpp/blob/10.0.0-stable/tools/godotcpp.py>
- GDExtension API dumps used for all signature matrices:
  - 4.2: `godot-4.2-stable/gdextension/extension_api.json`
  - 4.3–4.7: `master/gdextension/extension_api-4-{3..7}.json` (and the same files under tag `10.0.0-stable`)
- Godot source:
  - 4.2 `servers/rendering_server.cpp` (`_dict_to_surf`, `_mesh_add_surface`): <https://github.com/godotengine/godot/blob/4.2-stable/servers/rendering_server.cpp>
  - 4.7 `servers/rendering/rendering_server.cpp`: <https://github.com/godotengine/godot/blob/4.7-stable/servers/rendering/rendering_server.cpp>
  - 4.7 `core/extension/gdextension_library_loader.cpp`: <https://github.com/godotengine/godot/blob/4.7-stable/core/extension/gdextension_library_loader.cpp>
  - 4.2 `core/extension/gdextension.cpp`: <https://github.com/godotengine/godot/blob/4.2-stable/core/extension/gdextension.cpp>
- Official docs:
  - `.gdextension file` (4.6): <https://docs.godotengine.org/en/4.6/tutorials/scripting/gdextension/gdextension_file.html>
  - `NavigationServer3D` class reference (4.7): <https://docs.godotengine.org/en/stable/classes/class_navigationserver3d.html>
  - Jolt Physics (4.7): <https://docs.godotengine.org/en/stable/tutorials/physics/using_jolt_physics.html>
  - C#/.NET (4.6): <https://docs.godotengine.org/en/4.6/tutorials/scripting/c_sharp/index.html>
  - ProjectSettings (4.5), `dotnet/*` properties: <https://docs.godotengine.org/en/4.5/classes/class_projectsettings.html>
- Release notes:
  - 4.3 (DXIL/D3D12): <https://godotengine.org/releases/4.3/>
  - 4.4 (Jolt integrated, experimental): <https://godotengine.org/releases/4.4/>
  - 4.5 (async navigation iterations): <https://godotengine.org/releases/4.5/>
  - 4.6 (Jolt default, D3D12 default on Windows): <https://godotengine.org/releases/4.6/>
  - 4.7 (GDExtensions in Project Settings): <https://godotengine.org/releases/4.7/>
