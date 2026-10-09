# Research: `NavigationServer3D` async bake + region sync at the pinned Godot 4.7 / godot-cpp 10.0.0-stable

- **Ticket:** [#11 — Verify the NavigationServer3D bake/sync API at the pinned Godot version](https://github.com/acoreyj/voxel/issues/11) (part of map [#1](https://github.com/acoreyj/voxel/issues/1))
- **Date:** 2026-10-09
- **Status:** research complete; **corrects the sync model in `voxel-design-doc.md` §4.5** (see §7)
- **Pinned platform** (from [#9](https://github.com/acoreyj/voxel/issues/9)): Godot **4.7** (Forward+), **godot-cpp `10.0.0-stable` = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5`**, built with **`api_version=4.7`**
- **Spec of record:** `voxel-design-doc.md` §4.5 (Navigation), §7.3 (performance targets); companion `voxel-decision-log.md` DR-007, DR-008

All C++ signatures below were produced by running godot-cpp's own binding generator
(`binding_generator.generate_bindings`) at the pinned commit against
`gdextension/extension_api-4-7.json`, so they are the exact generated headers the wrapper compiles
against. Engine behaviour and the bake pipeline were read from the Godot **`4.7-stable`** source.
Class-reference prose is the 4.7 documentation. Sources are linked inline and collected in §9.

## TL;DR

1. **The async bake exists at 4.7 and is genuinely asynchronous.** `NavigationServer3D.bake_from_source_geometry_data_async(NavigationMesh, NavigationMeshSourceGeometryData3D, Callable)` dispatches to `WorkerThreadPool`. The source geometry is referenced (not copied) on submission; the Recast bake runs on a worker. See §5.
2. **The correct "has my submission gone live?" signal at 4.7 is `region_get_iteration_id(region)` (4.5+), not `map_get_iteration_id` (4.3+).** The design's `submitted_map_iteration` field is too coarse: the map id advances for any region on the map. **`map_force_update` is deprecated at 4.7 and explicitly incompatible with async iterations — the wrapper must not use it to force a bake live.** See §1, §6, §7.
3. **Runtime source geometry is fed through `NavigationMeshSourceGeometryData3D`, not by committing polygons to `NavigationMesh`.** `NavigationMesh.set_vertices()/add_polygon()/create_from_mesh()` are the *post-bake authored* form and **bypass Recast entirely** (no `agent_radius` erosion). Baking needs `bake_from_source_geometry_data[_async]` + a source resource populated with `add_faces()` / `append_arrays()` / `merge()`. See §3.
4. **Border padding is two things.** `NavigationMesh.border_size` is a **bake setting** (`cfg.borderSize` → `rcBuildRegions*`); it is *not* source geometry and does *not* clip the result to a region. The shared world-space padding band (`agent_radius + connection_radius`) must be **authored into the source triangles** (the design's "border source" closure) and must lie **inside the bake AABB** to reach Recast's rasterizer. See §4.
5. **Nothing in this contract requires a `compatibility_minimum` above 4.7.** Per-region sync needs **4.5+**; map-global sync needs **4.3+**. The pin from #9 (`compatibility_minimum = "4.7"`) is satisfied with room to spare. One row of the #2 note is corrected here: `map_set_use_async_iterations` is **4.4**, not 4.5. See §6.
6. **`INavServerSync` needs a small but real change.** The iteration baseline belongs to the wrapper, not the controller; `submit_group_bake` should capture it and `is_submission_synced` should compare per-region iteration and check `is_baking_navigation_mesh`. See §7.

---

## 1. Exact API sequence: submit a bake, then detect it is live

Generated signatures at godot-cpp `10.0.0-stable` / `api_version=4.7`
(`gen/include/godot_cpp/classes/navigation_server3d.hpp`):

```cpp
RID  NavigationServer3D::region_create();
void NavigationServer3D::region_set_navigation_mesh(const RID &p_region, const Ref<NavigationMesh> &p_navigation_mesh);
void NavigationServer3D::region_set_map(const RID &p_region, const RID &p_map);
void NavigationServer3D::region_set_transform(const RID &p_region, const Transform3D &p_transform);
void NavigationServer3D::region_set_enabled(const RID &p_region, bool p_enabled);
void NavigationServer3D::region_set_use_async_iterations(const RID &p_region, bool p_enabled);

void NavigationServer3D::bake_from_source_geometry_data(
        const Ref<NavigationMesh> &p_navigation_mesh,
        const Ref<NavigationMeshSourceGeometryData3D> &p_source_geometry_data,
        const Callable &p_callback = Callable());
void NavigationServer3D::bake_from_source_geometry_data_async(
        const Ref<NavigationMesh> &p_navigation_mesh,
        const Ref<NavigationMeshSourceGeometryData3D> &p_source_geometry_data,
        const Callable &p_callback = Callable());
bool NavigationServer3D::is_baking_navigation_mesh(const Ref<NavigationMesh> &p_navigation_mesh) const;

uint32_t NavigationServer3D::region_get_iteration_id(const RID &p_region) const;
uint32_t NavigationServer3D::map_get_iteration_id(const RID &p_map) const;
void     NavigationServer3D::map_force_update(const RID &p_map);   // DEPRECATED at 4.7
```

**Lifecycle for one group (raw-server path, recommended for the wrapper):**

```cpp
NavigationServer3D *nav = NavigationServer3D::get_singleton();

// --- once, when the group becomes live ---
RID region = nav->region_create();
Ref<NavigationMesh> navmesh; navmesh.instantiate();
// bake settings must be identical across all adjacent groups (see §3, §4):
navmesh->set_cell_size(kNavCellSize);
navmesh->set_cell_height(kNavCellHeight);
navmesh->set_agent_radius(kAgentRadius);
navmesh->set_border_size(kAgentRadius + kConnectionRadius);   // padding, see §4
navmesh->set_filter_baking_aabb(group_bake_aabb);             // see §4 caveat
navmesh->set_filter_baking_aabb_offset(Vector3());
navmesh->set_edge_max_error(1.0f);                            // required for border_size tile alignment
// (also agent_height/max_climb/max_slope/region sizes/detail, same on every group)
nav->region_set_navigation_mesh(region, navmesh);
nav->region_set_map(region, terrain_nav_map);
nav->region_set_transform(region, Transform3D());             // world-space triangles ⇒ identity
nav->region_set_enabled(region, true);
nav->region_set_use_async_iterations(region, true);           // default via project setting, see §5

// --- per rebuild ---
Ref<NavigationMeshSourceGeometryData3D> src; src.instantiate();
// authored owned + border triangles, see §3:
src->add_faces(group_triangles, Transform3D());
src->add_faces(border_triangles, Transform3D());

const uint32_t baseline = nav->region_get_iteration_id(region);   // capture BEFORE submit
nav->bake_from_source_geometry_data_async(navmesh, src, Callable());
// record submitted_terrain_revision / submitted_source_epoch, rebuild_in_flight = true

// --- on later ticks: poll, then re-bind and wait for the region iteration ---
if (!nav->is_baking_navigation_mesh(navmesh)) {
    // Raw-RID regions do NOT observe NavigationMesh.changed; re-bind to mark dirty:
    nav->region_set_navigation_mesh(region, navmesh);
    // then, once the region rebuilds (possibly on a worker):
    if (nav->region_get_iteration_id(region) != baseline) {   // uint32 wrap-safe with !=
        // synchronised: copy submitted_* -> synced_*, clear rebuild_in_flight, release NavSourceCache
    }
}
```

Notes that matter:

- **`map_force_update` is deprecated at 4.7.** The 4.7 class reference states: *"Deprecated: This method is no longer supported, as it is incompatible with asynchronous updates. It can only be used in a single-threaded context, at your own risk."* (It still exists and calls `map->sync()` synchronously, but it will not block for an in-flight async bake, so it cannot be used to force a submission live.) Do not put it in the sync loop.
- **`is_baking_navigation_mesh` is the "bake still in flight" query.** *"Returns true when the provided navigation mesh is being baked on a background thread."* Submission itself **errors** if the same `NavigationMesh` is already baking (`"NavigationMesh is already baking. Wait for current bake to finish."`). One in-flight bake per resource; the wrapper's `rebuild_in_flight` must serialize submissions.
- **The async callback fires on the main thread.** It is invoked from `NavMeshGenerator3D::sync()`, which runs from `GodotNavigationServer3D::process()` — i.e. once per rendered frame (not per physics tick). `NavigationRegion3D::_bake_finished` still defensively `call_deferred`s if it is somehow not on the main thread.
- The bake writes the result **into the same `NavigationMesh` resource** (`emit_changed()`, then the node re-binds the region). Godot keeps only the newest result; the design's "results are versioned by `(terrain_revision, source_epoch)`" is a **wrapper-side** concept, not an engine one.
- Failure/invalid path: if the source has no data, `bake_from_source_geometry_data_async` clears the navmesh and fires the callback immediately.

---

## 2. `NavigationRegion3D` / `region_*` lifecycle

The design's §4.5 says each group owns one `NavigationRegion3D` and one `NavigationMesh` on the terrain map. The node wraps the server RID. Either the node or the raw RID path works; the facts:

**Node path (`scene/3d/navigation/navigation_region_3d.cpp`):**

- The node owns a server region created with `region_create()`, assigned an owner id, enter/travel costs and layers in its constructor.
- `NavigationRegion3D::set_navigation_mesh(Ref<NavigationMesh>)` connects `NavigationMesh.changed` to `_navigation_mesh_changed()`, which calls `NavigationServer3D::region_set_navigation_mesh(region, navigation_mesh)`. So **when an async bake calls `emit_changed()` on the navmesh, the node automatically re-binds it and marks the region dirty.**
- `set_navigation_map(RID)` → `region_set_map()`; entering the tree uses the `World3D` default map unless overridden.
- `bake_navigation_mesh(on_thread = true)` is the node's own convenience: it instantiates source data, calls `parse_source_geometry_data()` to parse the scene subtree, then `bake_from_source_geometry_data[_async]()`. It then emits `bake_finished`. **This parses scene nodes; it is not the path for the design's cached runtime triangles**, which are built directly (§3).
- `is_baking()` → `is_baking_navigation_mesh(navigation_mesh)`; `is_baking()` is the node-level form.
- `NavigationRegion3D` is a `Node3D`, so using it from GDExtension means instantiating and adding a node to the tree; `region_set_map()` can bypass the tree requirement by passing the map RID explicitly.

**Raw-RID path (`modules/navigation_3d/nav_region_3d.cpp`):**

- `NavRegion3D::set_navigation_mesh()` sets `navmesh`, `iteration_dirty = true`, `request_sync()`. **It does not connect to `NavigationMesh.changed`.** A raw region therefore stays stale after an in-place async bake unless the wrapper calls `region_set_navigation_mesh(region, navmesh)` again after the bake finishes. The node path avoids this.
- `region_set_transform()` sets the region→world transform. If the wrapper authors triangles in world space, keep it identity.
- `region_set_enabled(false/true)` also marks the region dirty and (eventually) advances its iteration id.
- Free with `NavigationServer3D.free_rid(region)` when a group dies.

**Region sync (`modules/navigation_3d/nav_region_3d.cpp`):**

- `_build_iteration()` reads the navmesh data (`navmesh->get_data(...)`) on the main thread, then, if `use_async_iterations` (default **true**, project setting `navigation/world/region_use_async_iterations`), dispatches `NavRegionBuilder3D::build_iteration` to `WorkerThreadPool`. `_sync_iteration()` swaps the result and increments `iteration_id = iteration_id % UINT32_MAX + 1`. `region_get_iteration_id()` returns it. `0` means "never synchronised".
- `region_get_iteration_id` docs: *"Every time the navigation region changes and synchronizes, the iteration ID increases… wraps around to 1."*

---

## 3. Building the runtime source geometry (triangles)

Two completely different forms must not be confused:

| Form | API | What it does | Erosion / padding |
| :--- | :--- | :--- | :--- |
| **Authored navmesh** | `NavigationMesh.set_vertices(PackedVector3Array)`, `NavigationMesh.add_polygon(PackedInt32Array)`, `NavigationMesh.create_from_mesh(Mesh)` | Commits final walkable polygons directly | **None.** No `agent_radius`, no `border_size`, no Recast. `create_from_mesh` copies mesh triangles verbatim into polygons. |
| **Bake source** | `NavigationMeshSourceGeometryData3D` + `NavigationServer3D.bake_from_source_geometry_data[_async]` | Runs the Recast pipeline (rasterize → erode → partition → contours → polymesh → detail) into the `NavigationMesh` | Full bake settings apply (`agent_radius`, `border_size`, `cell_size`, …) |

The design needs erosion (agents must not clip walls) and border padding, so it must use the **bake source** form. `NavigationMeshSourceGeometryData3D` is a `Resource` and can be instantiated in godot-cpp (`Ref<NavigationMeshSourceGeometryData3D> src; src.instantiate();`). It accepts runtime triangles **without parsing any scene node**:

```cpp
void NavigationMeshSourceGeometryData3D::add_faces(const PackedVector3Array &p_faces, const Transform3D &p_xform);
void NavigationMeshSourceGeometryData3D::append_arrays(const PackedFloat32Array &p_vertices, const PackedInt32Array &p_indices);
void NavigationMeshSourceGeometryData3D::set_vertices(const PackedFloat32Array &p_vertices);
void NavigationMeshSourceGeometryData3D::set_indices(const PackedInt32Array &p_indices);
void NavigationMeshSourceGeometryData3D::merge(const Ref<NavigationMeshSourceGeometryData3D> &p_other_geometry);
bool NavigationMeshSourceGeometryData3D::has_data();
void NavigationMeshSourceGeometryData3D::clear();
```

- **Winding / layout.** `add_faces` docs: *"For each face the array must have three vertex positions in clockwise winding order."* `PackedVector3Array`, `size() % 3 == 0`. Internally it emits indices `(0, 2, 1)` — i.e. it reverses the supplied winding to Godot's CCW convention; supply exactly what the docs ask.
- **`append_arrays`** takes a **flat** `PackedFloat32Array` (`x,y,z,…`) and `PackedInt32Array`; it offsets appended indices by `existing_vertex_count` (internally `vertices.size()/3`). This is the cheap way to append a cached chunk triangle block without building per-face `Vector3`s.
- **`set_vertices`/`set_indices` replace** the arrays; `Warning: Inappropriate data can crash the baking process` (third-party Recast). Validate before committing.
- **`parse_source_geometry_data`** is only needed when parsing scene nodes; not for the design's `NavSourceCache`.
- **`source_geometry_parser_create()`** (4.3+) is an optional hook to inject custom source geometry during node parsing. Not needed for the cached-triangle path.
- **Main-thread cost of source assembly is the wrapper's.** `bake_from_source_geometry_data_async` stores a `Ref` to the source and the worker reads it via `get_data()` under an `RWLock`; the arrays are not deep-copied on submission. Copying `NavSourceCache` triangles into the source resource (`append_arrays`) is O(triangles) and runs on the main thread.

---

## 4. Border padding / seams

The design (§4.5) pads a group's outer border by `agent_radius + connection_radius` and wants adjacent groups to share the same world-space border geometry. At 4.7 this decomposes into **two mechanisms**, and both are needed:

1. **`NavigationMesh.border_size` is a bake setting.** In `nav_mesh_generator_3d.cpp` it becomes `cfg.borderSize = ceil(border_size / cell_size)` and is passed to `rcBuildRegions`/`rcBuildRegionsMonotone`/`rcBuildLayerRegions`. 4.7 docs: *"The size of the non-navigable border around the bake bounding area. In conjunction with the `filter_baking_aabb` and an `edge_max_error` value at 1.0 or below the border size can be used to bake tile aligned navigation meshes without the tile edges being shrunk by `agent_radius`. Note: If this value is not 0.0, it will be rounded up to the nearest multiple of `cell_size` during baking."* So set `border_size = agent_radius + connection_radius` (or the chosen shared width) and `edge_max_error <= 1.0`. It does **not** add source geometry and does **not** clip the output to a region.
2. **The world-space padding band must be authored into the source triangles.** Recast rasterises the supplied triangles into the heightfield bounded by `cfg.bmin/bmax` (set from `filter_baking_aabb` + offset, else from the source geometry bounds) via `rcRasterizeTriangles`, which clips to that box. Geometry outside the bake AABB never reaches the heightfield. So the design's "border source" — triangles of resident chunks intersecting the `agent_radius + connection_radius` band, *including adjacent groups' chunks* — must be appended to the same `NavigationMeshSourceGeometryData3D`. Padding is therefore **authored into the source and expressed as `border_size`**, not either/or.

**Caveat / correction to §4.5 as written.** The design says *"Padding geometry is input to the bake only; it does not become owned playable area outside the group's actual region bounds."* The engine does **not** enforce that. Whatever is inside the bake AABB and survives erosion becomes the region's navmesh; if the padded band is inside the AABB it can appear as walkable polygons outside the group footprint. To honour "input-only", the wrapper must choose the interaction deliberately — e.g. set `filter_baking_aabb`/offset so the grid is the group footprint and accept that band triangles are clipped (losing their erosion influence), or bake over the padded box and clip the resulting polygons to the group. **This is a design decision the engine will not make for you and should be settled with a runtime test in M4.** (The availability answer — that all required APIs exist — is unaffected.)

Practical recipe to satisfy "same width, shared samples":

- Every group's `NavigationMesh` uses the same `cell_size`, `cell_height`, `agent_radius`, `border_size`, `agent_max_climb`, `agent_max_slope`, `edge_max_error`, sample partition type, and detail settings.
- A shared chunk's committed triangles are byte-identical inputs to whichever group(s) depend on it.
- `agent_radius` is ceiled to `cell_size`; `border_size` is ceiled to `cell_size`. Choose `cell_size` so the padding quantisation is acceptable (the engine emits `WARN_PRINT` when either is not an exact multiple).

---

## 5. Is the bake truly async, and what is the main-thread cost?

**Yes — worker-side, gated by a project setting.**

- `bake_from_source_geometry_data_async` calls `NavMeshGenerator3D::bake_from_source_geometry_data_async`, which (after a `has_data()` check and an `is_baking()` check) allocates a task and calls
  `WorkerThreadPool::get_singleton()->add_native_task(&NavMeshGenerator3D::generator_thread_bake, task, high_priority, SNAME("NavMeshGeneratorBake3D"))`.
  The Recast pipeline then runs in `generator_bake_from_source_geometry_data` on the worker.
- **Fallback to synchronous** when `use_threads` is false. `use_threads = GLOBAL_GET("navigation/baking/thread_model/baking_use_multiple_threads")`, **default `true`** (`core/config/project_settings.cpp`). Platforms/export presets without threads (e.g. Web without thread support) fall back, and then the callback is invoked before the async call returns; `is_baking` is immediately false.
- **Completion** is polled once per **rendered frame** in `NavMeshGenerator3D::sync()` (from `GodotNavigationServer3D::process()`): it checks `is_task_completed()`, `wait_for_task_completion()`, fires the callback, calls `navigation_mesh->emit_changed()`, and frees the task.

**Main-thread cost of a submission** is small and bounded:

- `has_data()` (RWLock read) and `is_baking()` (mutex + hash lookup);
- `memnew(NavMeshGeneratorTask3D)` + two hashmap inserts + one worker-pool enqueue (with its own lock). No vertex/Recast work, no source copy (a `Ref` is stored).
- The **dominant** main-thread cost is the wrapper assembling the `NavigationMeshSourceGeometryData3D` from `NavSourceCache` (appending owned + border triangles, §3), plus, on the completion frame, the callback and `emit_changed()` bookkeeping. The §7.3 target — *"< 10 ms submission overhead on main thread; actual bake asynchronous and measured separately"* — is therefore structurally achievable, but **cannot be confirmed from source alone**; it must be measured (§7.3). Likewise, region iteration reads `navmesh->get_data()` on the main thread before dispatching to a worker; `region_set_use_async_iterations(true)` keeps only that read on main.

---

## 6. Version gating (godot-cpp v10 dumps, `api_version = 4.3 … 4.7`)

Presence (`Y`) / absence (`-`) in the generated bindings:

| API | 4.3 | 4.4 | 4.5 | 4.6 | 4.7 |
| :--- | :-: | :-: | :-: | :-: | :-: |
| `bake_from_source_geometry_data` | Y | Y | Y | Y | Y |
| `bake_from_source_geometry_data_async` | Y | Y | Y | Y | Y |
| `parse_source_geometry_data` | Y | Y | Y | Y | Y |
| `source_geometry_parser_create` | Y | Y | Y | Y | Y |
| `map_get_iteration_id` | Y | Y | Y | Y | Y |
| `map_force_update` | Y | Y | Y | Y | Y |
| `is_baking_navigation_mesh` | Y | Y | Y | Y | Y |
| `map_set_use_async_iterations` | – | **Y** | Y | Y | Y |
| `region_get_iteration_id` | – | – | Y | Y | Y |
| `region_set_use_async_iterations` | – | – | Y | Y | Y |
| `region_create` / `region_set_navigation_mesh` | Y | Y | Y | Y | Y |
| `NavigationMesh.border_size` | Y | Y | Y | Y | Y |
| `NavigationMesh.create_from_mesh` / `add_polygon` | Y | Y | Y | Y | Y |
| `NavigationMeshSourceGeometryData3D.add_faces` / `append_arrays` | Y | Y | Y | Y | Y |

Readings:

- **Pinned 4.7 satisfies the whole contract**, including per-region iteration and async iterations. The full model needs **4.5+**; a map-global-only sync model needs **4.3+**. Nothing here forces `compatibility_minimum` above 4.7.
- **Correction to [#2](https://github.com/acoreyj/voxel/issues/2):** its §2.2 table listed `map_set_use_async_iterations` as 4.5+. The pinned godot-cpp v10 dumps say it is present in **4.4** (`version_full_name = Godot Engine v4.4.stable.official`). This does not change the #9 decision (4.7); it only corrects the version table.

---

## 7. Does `INavServerSync` map cleanly, and what must change?

The §4.5 interface:

```cpp
class INavServerSync {
public:
    virtual void submit_group_bake(const NavigationGroupKey& p_key,
                                   uint64_t p_terrain_revision,
                                   uint64_t p_source_epoch,
                                   uint64_t p_map_iteration) = 0;
    virtual bool is_submission_synced(const NavigationGroupKey& p_key,
                                      uint64_t p_map_iteration) = 0;
};
```

It **maps onto the 4.7 API, but its shape is wrong in one place: it lets the pure-C++ controller own a Godot server iteration number.** Corrections:

1. **Per-region, not per-map.** The precise "this submission is live" signal is `region_get_iteration_id(region)`. `map_get_iteration_id(map)` only changes when the *map* finishes an iteration and can advance because of other regions. Rename the state field `NavigationGroupState::submitted_map_iteration` → `submitted_region_iteration` (or keep the name but document it as the region's id) and the interface parameter accordingly.
2. **The controller must not supply the baseline.** `p_map_iteration` is an *output* of the server, not an input the pure-C++ controller can know. Either:
   - change `submit_group_bake` to `virtual uint64_t submit_group_bake(const NavigationGroupKey&, uint64_t terrain_revision, uint64_t source_epoch) = 0;` returning the baseline `region_get_iteration_id(region)` captured at submission; or
   - keep a `void` signature and have the wrapper store the per-key baseline internally, with `is_submission_synced(const NavigationGroupKey&)` taking **only the key**.
   Option (b) keeps `INavServerSync` minimal and the controller ignorant of Godot iteration values; prefer it unless the controller needs to assert the baseline.
3. **`is_submission_synced` semantics:** return true when
   `!NavigationServer3D::is_baking_navigation_mesh(navmesh_for_key)` **and** `region_get_iteration_id(region_for_key) != submitted_baseline_region_iteration` (uint32; use `!=`, not `>=`, because the id wraps at `UINT32_MAX` back to `1`).
4. **Never use `map_force_update`.** Deprecated at 4.7 and incompatible with async iteration. Poll.
5. **One in-flight bake per `NavigationMesh`.** `bake_from_source_geometry_data_async` errors if the resource is already baking. The design's "a new rebuild does not overwrite an in-flight result; results are versioned by `(terrain_revision, source_epoch)`" is implemented by the wrapper's own queue/`rebuild_in_flight`, **not** by Godot, which keeps only the latest bake result in the resource.
6. **Include the re-bind step in the wrapper.** If the design uses raw region RIDs, the wrapper must call `region_set_navigation_mesh(region, navmesh)` after `is_baking_navigation_mesh` goes false (raw regions do not consume `NavigationMesh.changed`). If it uses `NavigationRegion3D` nodes, the node re-binds automatically on `emit_changed()`.
7. **Callback vs poll.** A `Callable` passed to `bake_from_source_geometry_data_async` is the cleanest completion edge (delivered on the main thread in `NavigationServer3D.process()`). It can be used to trigger the re-bind; the wrapper still polls the region iteration to know the region is *live*. With `use_threads == false`, the callback fires inline and `is_baking` is immediately false — handle that path.

**Net:** `INavServerSync` remains implementable and the "pure C++ controller, Godot adapter" split holds. The only required change is to stop threading a Godot map-iteration value through the controller and to key synchronization on **region iteration + is-baking**, with the wrapper owning the baseline.

### Concrete correction to the design text

- §4.5 "Update Lifecycle" step 3: *"records … the map iteration observed at submission"* and step 5 *"polls the pinned Godot navigation synchronization state"* → should read **region iteration** (`region_get_iteration_id`), captured at submission, and the wrapper must re-bind the region mesh after the bake completes (unless it uses `NavigationRegion3D`, which re-binds via `changed`). §4.5's closing sentence ("the exact Godot API sequence is pinned and verified at M0") is now answered by this note.
- `NavigationGroupState::submitted_map_iteration` → `submitted_region_iteration`.
- `INavServerSync::submit_group_bake`/`is_submission_synced` signatures per §7.2–7.3.
- §4.5 seam claim "padding … does not become owned playable area" is **not automatic** and needs the `filter_baking_aabb` decision in §4 above.

---

## 8. What could not be verified from primary sources

- **Absolute main-thread timing** of submission (`< 10 ms`, §7.3). Only the qualitative cost structure is verified from source; the number needs an M0/M4 measurement on the reference hardware.
- **Exact cross-group edge-connection behaviour** when adjacent regions bake overlapping padded navmeshes (how region ownership/merge resolves the band). This is a runtime property that needs a test scene; it is not decidable by reading the bindings.
- **Whether the design's "padding is input-only, not owned area" holds** under a chosen `(filter_baking_aabb, border_size)` pair. Engine semantics are documented above; the wrapper must pick and test.
- The **`navigation/world/region_use_async_iterations` / `baking_use_multiple_threads` platform matrix** (which export presets fall back to synchronous). Only engine defaults and the `use_threads` fallback are verified; platform-specific behaviour is out of scope here.

---

## 9. Sources

**godot-cpp (pinned `10.0.0-stable` = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5`, generated with `api_version=4.7`):**

- Generated bindings (this note): `gen/include/godot_cpp/classes/navigation_server3d.hpp`, `navigation_mesh.hpp`, `navigation_mesh_source_geometry_data3d.hpp`, `navigation_region3d.hpp` — produced locally by `binding_generator.generate_bindings` against `gdextension/extension_api-4-7.json`.
- API dump used: <https://github.com/godotengine/godot-cpp/blob/507ed9d840c01a3c5b2a39af8bb4000bfac30bf5/gdextension/extension_api-4-7.json>
- Version-presence table: `gdextension/extension_api-4-{3,4,5,6,7}.json` at the same commit.

**Godot class reference (4.7):**

- `NavigationServer3D`: <https://docs.godotengine.org/en/4.7/classes/class_navigationserver3d.html>
- `NavigationMesh`: <https://docs.godotengine.org/en/4.7/classes/class_navigationmesh.html>
- `NavigationRegion3D`: <https://docs.godotengine.org/en/4.7/classes/class_navigationregion3d.html>
- `NavigationMeshSourceGeometryData3D`: <https://docs.godotengine.org/en/4.7/classes/class_navigationmeshsourcegeometrydata3d.html>

**Godot engine source (`4.7-stable`):**

- `modules/navigation_3d/3d/nav_mesh_generator_3d.cpp` (async dispatch, worker bake, `border_size`/`agent_radius` → Recast config, completion in `sync()`): <https://github.com/godotengine/godot/blob/4.7-stable/modules/navigation_3d/3d/nav_mesh_generator_3d.cpp>
- `modules/navigation_3d/3d/godot_navigation_server_3d.cpp` (`bake_*`, `is_baking_navigation_mesh`, `map_force_update`, `map_get_iteration_id`, `region_get_iteration_id`, `process()`/`physics_process()`): <https://github.com/godotengine/godot/blob/4.7-stable/modules/navigation_3d/3d/godot_navigation_server_3d.cpp>
- `modules/navigation_3d/nav_region_3d.cpp` (region lifecycle, iteration bump, async iteration): <https://github.com/godotengine/godot/blob/4.7-stable/modules/navigation_3d/nav_region_3d.cpp>
- `modules/navigation_3d/nav_map_3d.cpp` (map iteration bump, `sync()`): <https://github.com/godotengine/godot/blob/4.7-stable/modules/navigation_3d/nav_map_3d.cpp>
- `scene/3d/navigation/navigation_region_3d.cpp` (node lifecycle, `changed` → re-bind, `bake_navigation_mesh`): <https://github.com/godotengine/godot/blob/4.7-stable/scene/3d/navigation/navigation_region_3d.cpp>
- `scene/resources/3d/navigation_mesh_source_geometry_data_3d.cpp` (`add_faces`, `append_arrays`, `merge`, winding/offset): <https://github.com/godotengine/godot/blob/4.7-stable/scene/resources/3d/navigation_mesh_source_geometry_data_3d.cpp>
- `scene/resources/navigation_mesh.cpp` (`create_from_mesh`, `border_size`/`agent_radius` properties, `set_vertices`/`add_polygon`): <https://github.com/godotengine/godot/blob/4.7-stable/scene/resources/navigation_mesh.cpp>
- `core/config/project_settings.cpp` (async-iteration and threaded-baking defaults): <https://github.com/godotengine/godot/blob/4.7-stable/core/config/project_settings.cpp>

**Reproducibility:** the pinned API dump and the godot-cpp generator are the exact inputs used to produce the signatures above; the engine files are at the `4.7-stable` tag.
