# Research: RenderingServer surface path and material channel at the pinned godot-cpp

- **Ticket:** [#10 — Verify the RenderingServer surface and material-channel API at the pinned godot-cpp](https://github.com/acoreyj/voxel/issues/10) (part of map [#1](https://github.com/acoreyj/voxel/issues/1))
- **Date:** 2026-10-08
- **Status:** research complete; resolves decision-log `L-02` and `L-03`
- **Pinned platform** (from [#9](https://github.com/acoreyj/voxel/issues/9)): Godot **4.7** (Forward+), **godot-cpp `10.0.0-stable` = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5`**, built with **`api_version=4.7`**
- **Spec of record:** `voxel-design-doc.md` §4.1 (memory/marshalling), §4.3 (commit loop), §4.4 (collision); companion `voxel-decision-log.md` `L-02`, `L-03`

All C++ signatures below were produced by running godot-cpp's own binding generator
(`binding_generator.py`, `precision=single`) against
`gdextension/extension_api-4-7.json` at the pinned commit, so they are the exact generated
headers the wrapper compiles against — not the JSON read by eye and not a docs paraphrase.

## TL;DR

1. **There is no raw-attribute-view overload, at 4.7 or any version.** godot-cpp exposes exactly
   two surface-add methods: `mesh_add_surface_from_arrays(mesh, primitive, Array arrays, ...)` and
   `mesh_add_surface(mesh, Dictionary surface)`. The C++ `RenderingServer::mesh_add_surface(RID, const SurfaceData &)`
   is a pure virtual that is **not** bound to GDExtension; the bound form is
   `_mesh_add_surface(RID, Dictionary)`, which calls `_dict_to_surf`. So the "lower-level overload that
   accepts raw attribute views" the design hopes for **does not exist**. `mesh_add_surface` takes a
   **serialized `Dictionary`** of already-packed byte buffers. (→ `L-02`)
2. **The material channel fits a custom vertex attribute at 4.7 — and since 4.2.** Use
   `ARRAY_CUSTOM0` with `ARRAY_CUSTOM_RGBA8_UNORM` (4 bytes/vertex, format type `0`). The array-path
   value is a `PackedByteArray` of exactly `4 * vertex_count` bytes; the shader reads `in vec4 CUSTOM0`.
   A per-chunk shader parameter is only a fallback and **cannot represent a mixed-material chunk**
   (it is per-instance, not per-vertex). (→ `L-03`)
3. **Mesh RID lifecycle:** `mesh_create()` → add surface → `mesh_surface_set_material()`. Replace by
   `mesh_surface_remove()` (added **4.4**) or `mesh_clear()` (all versions) + re-add; same-topology
   updates can use the `mesh_surface_update_*_region` calls (`vertex`/`attribute`/`skin` since 4.2,
   `index` since **4.5**). Free with `free_rid()`; RIDs are **not** ref-counted.
4. **Collision:** `ConcavePolygonShape3D::set_faces(const PackedVector3Array &)` accepts a flat
   `float` triangle soup **directly via one memcpy** because `PackedVector3Array` is a contiguous
   `Vector3[]` and `Vector3` is exactly 3 `real_t` (`precision=single` ⇒ 12 bytes). The **only**
   cost-relevant setting is `backface_collision` (default `false`); **there is no "fast parse"
   setting** — `_setup()` always copies the soup and rebuilds the BVH.

---

## 1. The surface-add API (decision-log `L-02`)

### 1.1 Exact generated signatures at the pin

From the generated `gen/include/godot_cpp/classes/rendering_server.hpp` at `10.0.0-stable`
(`api_version=4.7`, `precision=single`):

```cpp
RID  mesh_create();
RID  mesh_create_from_surfaces(const TypedArray<Dictionary> &p_surfaces, int32_t p_blend_shape_count = 0);

uint32_t mesh_surface_get_format_offset(BitField<ArrayFormat> p_format, int32_t p_vertex_count, int32_t p_array_index) const;
uint32_t mesh_surface_get_format_vertex_stride(BitField<ArrayFormat> p_format, int32_t p_vertex_count) const;
uint32_t mesh_surface_get_format_normal_tangent_stride(BitField<ArrayFormat> p_format, int32_t p_vertex_count) const;
uint32_t mesh_surface_get_format_attribute_stride(BitField<ArrayFormat> p_format, int32_t p_vertex_count) const;
uint32_t mesh_surface_get_format_skin_stride(BitField<ArrayFormat> p_format, int32_t p_vertex_count) const;
uint32_t mesh_surface_get_format_index_stride(BitField<ArrayFormat> p_format, int32_t p_vertex_count) const;

void  mesh_add_surface(const RID &p_mesh, const Dictionary &p_surface);
void  mesh_add_surface_from_arrays(const RID &p_mesh, PrimitiveType p_primitive, const Array &p_arrays,
                                   const Array &p_blend_shapes = Array(), const Dictionary &p_lods = Dictionary(),
                                   BitField<ArrayFormat> p_compress_format = (BitField<ArrayFormat>)0);

void  mesh_surface_set_material(const RID &p_mesh, int32_t p_surface, const RID &p_material);
RID   mesh_surface_get_material(const RID &p_mesh, int32_t p_surface) const;
Dictionary mesh_get_surface(const RID &p_mesh, int32_t p_surface);

void  mesh_surface_remove(const RID &p_mesh, int32_t p_surface);
void  mesh_clear(const RID &p_mesh);
void  mesh_surface_update_vertex_region(const RID &p_mesh, int32_t p_surface, int32_t p_offset, const PackedByteArray &p_data);
void  mesh_surface_update_attribute_region(const RID &p_mesh, int32_t p_surface, int32_t p_offset, const PackedByteArray &p_data);
void  mesh_surface_update_skin_region(const RID &p_mesh, int32_t p_surface, int32_t p_offset, const PackedByteArray &p_data);
void  mesh_surface_update_index_region(const RID &p_mesh, int32_t p_surface, int32_t p_offset, const PackedByteArray &p_data);

void  free_rid(const RID &p_rid);
```

The C++ side confirms the bind split. In Godot 4.7 `servers/rendering/rendering_server.h`:

```cpp
virtual void mesh_add_surface(RID p_mesh, const RenderingServerTypes::SurfaceData &p_surface) = 0;   // line 214, NOT bound
virtual void mesh_add_surface_from_arrays(RID p_mesh, RSE::PrimitiveType p_primitive, const Array &p_arrays,
        const Array &p_blend_shapes = Array(), const Dictionary &p_lods = Dictionary(),
        BitField<RSE::ArrayFormat> p_compress_format = 0);                                            // line 213
```

and in `rendering_server.cpp`:

```cpp
void RenderingServer::ClassDB... bind_method(D_METHOD("mesh_add_surface", "mesh", "surface"),
                                             &RenderingServer::_mesh_add_surface);     // line 2367
void RenderingServer::_mesh_add_surface(RID p_mesh, const Dictionary &p_surface) {  // line 2003
    mesh_add_surface(p_mesh, _dict_to_surf(p_surface));
}
```

`_dict_to_surf` (line 1933) reads a fixed key set: required `primitive`, `format`, `vertex_data`,
`vertex_count`, `aabb`; optional `attribute_data`, `skin_data`, `index_data`, `index_count`,
`uv_scale`, `lods`, `bone_aabbs`, `blend_shape_data`, `material`. So **yes, it takes a serialized
`Dictionary` of packed byte buffers** — but the `SurfaceData &` view that would accept a raw C++
attribute buffer is not reachable from a GDExtension.

### 1.2 Why "the dictionary form avoids the packed arrays" is only half true

`mesh_add_surface(RID, Dictionary)` does avoid the three `Packed*Array` objects, but it does **not**
avoid the work or the copy, and it moves the hard part onto the caller:

- The caller must own Godot's **internal packed layout** and the correct `format` bitfield.
  `mesh_surface_make_offsets_from_format` (`rendering_server.cpp` line 1055) lays out
  `vertex_data` as a **position block followed by a normal/tangent block**, and **normals are
  octahedral-compressed to 4 bytes** (`ARRAY_NORMAL` element size is always `4`). The design's
  worker output — interleaved `{Vector3 position, Vector3 normal, uint32_t material}` — therefore
  **cannot be handed over verbatim**; the normal half must be re-packed/compressed either by the
  worker (dictionary path) or by Godot (`_surface_set_data`, array path).
- `_dict_to_surf` copies the caller's `PackedByteArray` into `SurfaceData::vertex_data`
  (`Vector<uint8_t>`), so the dictionary path still performs one full copy.
- The dictionary path must also set the **surface format version bits** itself.
  `MeshStorage::mesh_add_surface` hard-fails unless
  `format & (VERSION_MASK << VERSION_SHIFT) == ARRAY_FLAG_FORMAT_CURRENT_VERSION`
  (`mesh_storage.cpp` lines 354–368). A zero version is v1, which `fix_surface_compatibility`
  would then reinterpret as the old *interleaved* layout — silently corrupting a v2 buffer.
  The array path sets this automatically (`rendering_server.cpp` lines 1275–1276), which is a
  strong correctness argument for using it in the spike.

**Verdict for `L-02`:** the design's hoped-for raw-view overload does not exist at the pinned
version (or any Godot 4.x). The dictionary form is the closest thing to a lower-level path, but it
is a serialized-buffer API, and adopting it means the worker (not the main thread) must reproduce
Godot's packed layout including normal compression and index width. That *is* aligned with §4.1's
"main thread only copies" goal, but it is an optimisation to be measured in the `L-01`
(`#12`/`#13`) throughput spike — not a version question. The minimal, correct default is the
array path; if the array path's per-vertex repack in `_surface_set_data` shows up in the spike,
switch the worker to emit Godot-layout buffers for the dictionary path.

### 1.3 Which methods are post-4.2

Presence checked against the generated API dumps (`godot-4.2-stable` and `extension_api-4-{3..7}.json`):

| Method | 4.2 | 4.3 | 4.4 | 4.5 | 4.6 | 4.7 |
| :--- | :-: | :-: | :-: | :-: | :-: | :-: |
| `mesh_add_surface` (Dictionary) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_add_surface_from_arrays` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_create_from_surfaces` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_get_surface` / `mesh_get_surface_count` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_set_material` / `mesh_get_surface_material` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_update_vertex_region` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_update_attribute_region` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_update_skin_region` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_clear` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_remove` | – | – | ✓ | ✓ | ✓ | ✓ |
| `mesh_surface_update_index_region` | – | – | – | ✓ | ✓ | ✓ |

Practical reading at the pin: **both** `mesh_clear` and per-surface `mesh_surface_remove` are
available, and all four `mesh_surface_update_*_region` calls are available. The design's commit
loop can therefore replace exactly one surface without touching the others, and can partially
re-upload.

---

## 2. The material channel (decision-log `L-03`)

### 2.1 It can be a custom vertex attribute — at the pinned version and since 4.2

`ARRAY_CUSTOM0..3` and `ARRAY_CUSTOM_RGBA8_UNORM` exist in every dump from 4.2 to 4.7
(`RenderingServer.ArrayType` / `ArrayCustomFormat`). The engine already implements the
encode/decode path in 4.2, so this is a shader/format choice, not a version gate.

- **What the channel needs:** the design stores one `uint8_t` material per canonical sample
  (`voxel-design-doc.md`, "Material Channel"); the ticket allows `uint8_t`/`uint32_t` per vertex.
  Godot's smallest custom format is **4 bytes/vertex**, so the slot is a `uint32_t`-sized value.
- **Array constant:** `RenderingServer::ARRAY_CUSTOM0`.
- **Format:** `RenderingServer::ARRAY_CUSTOM_RGBA8_UNORM` (value `0`). The array value must be a
  `PackedByteArray` of **exactly `4 * vertex_count` bytes**
  (`rendering_server.cpp` `_surface_set_data`, custom case, lines 779–800). You can memcpy a worker
  `uint32_t[]` straight in.
- **Compress-format argument:** for `RGBA8_UNORM` the custom-format bits are `0`, so the default
  `compress_format = 0` selects it. For `ARRAY_CUSTOM_R_FLOAT` (also 4 bytes, a single float) the
  argument is `ARRAY_CUSTOM_R_FLOAT << ARRAY_FORMAT_CUSTOM0_SHIFT` (i.e. `4 << 13`).
- **Shader:** a spatial shader declares `in vec4 CUSTOM0` in `vertex()` and passes it to
  `fragment()` via a varying. `RGBA8_UNORM` arrives normalised to `[0,1]`; decode a material id with
  `int(CUSTOM0.r * 255.0 + 0.5)`. Other custom formats exist (`RGBA8_SNORM`, `RG_HALF`,
  `RGBA_HALF`, `R_FLOAT`, `RG_FLOAT`, `RGB_FLOAT`, `RGBA_FLOAT`).

### 2.2 The per-chunk shader-parameter fallback is coarse

If a fallback were needed, the natural mechanism with the RenderingServer-direct path is the
**per-instance** shader parameter `instance_geometry_set_shader_parameter(instance, "material_id", value)`.
But that is one value for the whole chunk. Since the material channel is **per sample** (a chunk
can contain several materials), the per-chunk parameter **cannot reproduce the design's mixed-material
chunks**; it only works if a chunk is guaranteed single-material. The custom attribute is the only
path that preserves the per-vertex material channel.

**Verdict for `L-03`:** use `ARRAY_CUSTOM0` + `ARRAY_CUSTOM_RGBA8_UNORM`, per-vertex. It is
supported at the pin. The per-chunk shader parameter is a lossy fallback, not an equivalent.

---

## 3. Mesh RID lifecycle: create / populate / replace / free

### 3.1 Ownership rules

- RIDs returned by `mesh_create()`, `instance_create*()`, `material_create()`,
  `concave_polygon_shape_create()` etc. are **not reference-counted**. `free_rid()` is mandatory;
  the docs are explicit: *"To avoid memory leaks, this should be called after using an object as
  memory management does not occur automatically when using RenderingServer directly."*
- `mesh_surface_set_material(mesh, surface, material_rid)` stores the material RID on the surface;
  it does **not** take ownership. If the RID came from `material_create()` the caller frees it; if
  it came from a `ShaderMaterial` resource (`material->get_rid()`) the resource owns it — do not
  `free_rid` it.
- `mesh_surface_remove`/`mesh_clear` drop surface data and update any attached instances
  (`MeshStorage::mesh_surface_remove` calls `_mesh_instance_remove_surface`; `mesh_clear` calls
  `_mesh_instance_clear`). They do not free the mesh RID itself.
- Reusing one persistent mesh RID per resident chunk (replace the surface on recommit) is preferable
  to create/free per commit — it keeps the instance->base binding stable.

### 3.2 Minimal rendering call sequence (array path, recommended default)

```cpp
using RS = RenderingServer;
RS *rs = RS::get_singleton();

// Worker POD, precision=single: float positions[3N], float normals[3N],
//                              uint32_t materials[N], int32_t indices[M]
RID mesh = rs->mesh_create();                       // own it; free_rid() on eviction

PackedVector3Array positions; positions.resize(N);
memcpy(positions.ptrw(), pos, N * sizeof(Vector3)); // Vector3 == 3 floats

PackedVector3Array normals; normals.resize(N);
memcpy(normals.ptrw(), nrm, N * sizeof(Vector3));

PackedInt32Array indices; indices.resize(M);        // ARRAY_INDEX is PackedInt32Array
memcpy(indices.ptrw(), idx, M * sizeof(int32_t));

PackedByteArray materials; materials.resize(N * 4); // ARRAY_CUSTOM0, RGBA8_UNORM
memcpy(materials.ptrw(), mat, N * 4);               // 4 bytes per vertex

Array arrays;
arrays.resize(RS::ARRAY_MAX);
arrays[RS::ARRAY_VERTEX]  = positions;   // 0
arrays[RS::ARRAY_NORMAL]  = normals;     // 1
arrays[RS::ARRAY_CUSTOM0] = materials;   // 6
arrays[RS::ARRAY_INDEX]   = indices;     // 12

rs->mesh_add_surface_from_arrays(mesh, RS::PRIMITIVE_TRIANGLES, arrays); // surface index 0
rs->mesh_surface_set_material(mesh, 0, material_rid);                    // shader reads CUSTOM0

// Display: one instance per chunk, bound to the world's scenario.
RID scenario = get_world_3d()->get_scenario();
RID instance = rs->instance_create2(mesh, scenario);   // or instance_create()+set_base()+set_scenario()
rs->instance_set_transform(instance, chunk_xform);

// --- replace, same topology (counts/format unchanged): in-place, no realloc ---
// off = byte offset; use mesh_surface_get_format_* to compute strides if needed.
rs->mesh_surface_update_vertex_region(mesh, 0, off, positions_as_bytes);
rs->mesh_surface_update_attribute_region(mesh, 0, off, materials_bytes);
rs->mesh_surface_update_index_region(mesh, 0, off, indices_bytes);   // 4.5+

// --- replace, changed topology (vertex/index counts differ) ---
rs->mesh_surface_remove(mesh, 0);                   // 4.4+; or rs->mesh_clear(mesh)
rs->mesh_add_surface_from_arrays(mesh, RS::PRIMITIVE_TRIANGLES, arrays);
rs->mesh_surface_set_material(mesh, 0, material_rid);   // re-set after remove

// --- evict a chunk ---
rs->free_rid(instance);   // detach/free instance first
rs->free_rid(mesh);       // then the mesh and all its surfaces
```

Notes:

- `mesh_surface_update_*_region` calls `RD::buffer_update` on the existing buffer
  (`mesh_storage.cpp` lines 571–625); they do **not** resize. They require the surface to already
  exist and the buffer to be non-null. `ARRAY_FLAG_USE_DYNAMIC_UPDATE` is documented as a
  **GLES-only** hint (`GL_DYNAMIC_DRAW`) and *"Unused on Vulkan"* — the Forward+ target does not
  need it.
- `ARRAY_INDEX` must be a `PackedInt32Array`; the engine narrows to 16-bit internally when
  `vertex_count <= 65536`.
- The array path computes the surface `AABB` itself from the vertex array
  (`_surface_set_data` → `surface_data.aabb`), so the caller supplies no `aabb`.

### 3.3 Minimal dictionary variant (only if workers emit Godot's packed layout)

```cpp
// format = VER | ARRAY_FORMAT_VERTEX | ARRAY_FORMAT_NORMAL | ARRAY_FORMAT_CUSTOM0 | ARRAY_FORMAT_INDEX
// VER    = RenderingServer::ARRAY_FLAG_FORMAT_CURRENT_VERSION   (required; see §1.2)
// custom-format bits for RGBA8_UNORM are 0.
Dictionary surf;
surf["primitive"]    = RS::PRIMITIVE_TRIANGLES;
surf["format"]       = format;
surf["vertex_data"]  = vertex_data;    // PackedByteArray: positions block + 4-byte compressed normals block
surf["vertex_count"] = N;
surf["attribute_data"] = attribute_data; // PackedByteArray: custom0 block, 4*N
surf["index_data"]   = index_data;       // PackedByteArray: uint16 or uint32
surf["index_count"]  = M;
surf["aabb"]         = aabb;             // required
// surf["material"] = material_rid;      // optional; see caveat
rs->mesh_add_surface(mesh, surf);
```

Caveat on the `"material"` key: the RenderingServer docs describe it as a `Material`
(`class_renderingserver.html`, `mesh_add_surface`), but `_dict_to_surf` assigns it to
`SurfaceData::material`, which is an **`RID`** (`rendering_server_types.h`, line 110). Prefer
`mesh_surface_set_material(mesh, 0, material_rid)` and leave the key out.

---

## 4. Collision: `ConcavePolygonShape3D` from a flat float soup

### 4.1 It consumes a flat soup with a single memcpy

Generated signature (godot-cpp 10.0.0-stable, `concave_polygon_shape3d.hpp`):

```cpp
void set_faces(const PackedVector3Array &p_faces);
PackedVector3Array get_faces() const;
void set_backface_collision_enabled(bool p_enabled);
bool is_backface_collision_enabled() const;
```

`PackedVector3Array` exposes `Vector3 *ptrw()` (contiguous), and `Vector3` is exactly three
consecutive `real_t` (`include/godot_cpp/variant/vector3.hpp`: `real_t x, y, z`). With
`precision=single` (the godot-cpp default) `real_t == float` and `sizeof(Vector3) == 12`, so a
worker-built flat `float[3*V]` triangle soup is **layout-identical** to the target array:

```cpp
PackedVector3Array faces; faces.resize(3 * triangle_count);   // 3 vertices/triangle
memcpy(faces.ptrw(), soup, 3 * triangle_count * sizeof(Vector3)); // == float[9*tri_count]
Ref<ConcavePolygonShape3D> shape;
shape.instantiate();
shape->set_faces(faces);
shape->set_backface_collision_enabled(false);
```

No re-indexing and no per-triangle loop is required on the main thread. **Precision caveat:** this
holds only for `precision=single`. A `precision=double` build (`real_t == double`) would make
`Vector3` 24 bytes and a `float` soup would be misread; the C++ wrapper's precision must match the
Godot build (this is already pinned to the default `single`).

### 4.2 Settings and update cost

- **There is no fast-parse setting.** The 4.7 `ConcavePolygonShape3D` class reference exposes only
  `faces` (the data) and `backface_collision`; the method list is just `set_faces`/`get_faces` and
  `set_backface_collision_enabled`/`is_backface_collision_enabled`.
- **`backface_collision`** (default `false`) is the one relevant cost knob: when enabled,
  collisions occur on both sides of each face (`GodotConcavePolygonShape3D::cull` sets
  `face.backface_collision = backface_collision`), i.e. back-face queries traverse the same BVH.
  Leave it `false` for terrain unless one-sided queries are insufficient.
- **Cost is always a copy plus a BVH rebuild.** `ConcavePolygonShape3D::set_faces` first copies the
  array into its member `faces`, then `_update_shape` passes a `Dictionary` to the physics server;
  `GodotConcavePolygonShape3D::_setup` (`modules/godot_physics_3d/godot_shape_3d.cpp`, line 1597)
  copies every vertex again into its own `vertices` vector and builds a per-face `_Volume_BVH`.
  There is **no zero-copy** path. Recommitting a chunk re-runs the whole build, so this is exactly
  the §4.3/§7.3 cost to measure; the design's decimation escape hatch is the lever if it misses.
- **Server-direct equivalent** (if the wrapper manages `PhysicsServer3D` RIDs instead of nodes):
  `shape_set_data(shape, Dictionary{"faces": PackedVector3Array, "backface_collision": bool})` is
  the documented and implemented contract (`class_physicsserver3d.html`; `GodotConcavePolygonShape3D::set_data`
  requires a `Dictionary` with `"faces"`). `ConcavePolygonShape3D::set_faces` builds that same
  dictionary internally.

---

## 5. What the wrapper should implement

- **Default surface path:** `mesh_add_surface_from_arrays(mesh, PRIMITIVE_TRIANGLES, arrays)` with
  `ARRAY_VERTEX` (`PackedVector3Array`, single memcpy), `ARRAY_NORMAL` (`PackedVector3Array`, single
  memcpy), `ARRAY_INDEX` (`PackedInt32Array`, single memcpy), and `ARRAY_CUSTOM0`
  (`PackedByteArray`, `4 * N`, single memcpy). Keep `ArrayMesh` behind the existing `use_array_mesh`
  property for A/B.
- **Material channel:** `ARRAY_CUSTOM0` + `ARRAY_CUSTOM_RGBA8_UNORM`; spatial shader reads
  `in vec4 CUSTOM0`. Do not use the per-chunk shader-parameter fallback except for
  guaranteed-single-material chunks.
- **Chunk RID policy:** one persistent `mesh_create()` RID per resident chunk, one
  `instance_create2(mesh, scenario)`; replace same-topology edits with
  `mesh_surface_update_vertex_region`/`_attribute_region`/`_index_region`, and changed-topology
  recommits with `mesh_surface_remove` + `mesh_add_surface_from_arrays` + `mesh_surface_set_material`.
  On eviction `free_rid(instance)` then `free_rid(mesh)`. Free materials/shapes only when their
  resource/RID owner is done.
- **Spike follow-up:** if the `L-01` throughput target (8 dense chunks/frame) misses and the
  main thread is spending time in Godot's per-vertex repack (`_surface_set_data`), move the pack into
  the worker and switch to the `mesh_add_surface(mesh, Dictionary)` path — but the worker must then
  emit Godot's v2 layout (planar positions + 4-byte compressed normals + custom block + index width)
  and set `ARRAY_FLAG_FORMAT_CURRENT_VERSION`.
- **Collision:** worker emits a flat `float[9 * triangle_count]` soup; main thread memcpy's it into
  a resized `PackedVector3Array` and calls `set_faces` with `backface_collision = false`.
- **Precision:** build godot-cpp with `precision=single` (default) so the memcpy layouts hold.

## 6. Verification status

Everything above is drawn from primary sources at the pinned refs. One documentation inconsistency
is called out rather than smoothed over: the RenderingServer 4.7 reference advertises the
`mesh_add_surface` `"material"` key as a `Material`, while the implementation and the `SurfaceData`
type use an `RID`; the recommendation avoids the ambiguity by using `mesh_surface_set_material`.
No signature in this note had to be guessed.

## 7. Sources

**godot-cpp @ `10.0.0-stable` = `507ed9d840c01a3c5b2a39af8bb4000bfac30bf5`**

- Generated bindings (produced locally with `binding_generator.generate_bindings` from
  `gdextension/extension_api-4-7.json`, `gdextension_interface.json`, `precision=single`):
  `gen/include/godot_cpp/classes/rendering_server.hpp` (lines 898–925, 1221–1231, 1361),
  `gen/include/godot_cpp/classes/concave_polygon_shape3d.hpp` (lines 45–52),
  `gen/include/godot_cpp/variant/packed_vector3_array.hpp` (lines 145–149).
- `include/godot_cpp/variant/vector3.hpp` (`real_t x, y, z`), `include/godot_cpp/core/math_defs.hpp`
  (`real_t` = `float` unless `precision=double`).
- `gdextension/extension_api-4-7.json` (all signatures and enum values) and
  `gdextension/extension_api-4-{3..6}.json` (version matrix); `godot-4.2-stable/gdextension/extension_api.json`
  (4.2 absence checks).
- `tools/godotcpp.py` (`precision` default `"single"`, `supported_api_versions`), `binding_generator.py`.
- <https://github.com/godotengine/godot-cpp/tree/10.0.0-stable>

**Godot `4.7-stable`**

- `servers/rendering/rendering_server.cpp`: `_surface_set_data` custom handling (lines 779–830),
  `mesh_surface_make_offsets_from_format` (line 1055), `mesh_add_surface_from_arrays` (line 1395),
  `_dict_to_surf` (line 1933), `_mesh_add_surface` (line 2003), `fix_surface_compatibility` (line 2161).
- `servers/rendering/rendering_server.h`: bound vs virtual surface API (lines 193–248, 1068–1069).
- `servers/rendering/rendering_server_types.h`: `struct SurfaceData` (line 83), `RID material` (line 110).
- `servers/rendering/renderer_rd/storage_rd/mesh_storage.cpp`: `mesh_surface_update_*_region` (571–625),
  `mesh_clear` (864), `mesh_surface_remove` (894), version check (354–368).
- `scene/resources/3d/concave_polygon_shape_3d.{h,cpp}`: `set_faces`/`_update_shape`/`backface_collision`.
- `modules/godot_physics_3d/godot_shape_3d.cpp`: `GodotConcavePolygonShape3D::_setup` (1597),
  `set_data` (1657).
- <https://github.com/godotengine/godot/tree/4.7-stable>

**Official docs (4.7)**

- RenderingServer: `mesh_add_surface`, `mesh_add_surface_from_arrays`, `mesh_create`,
  `mesh_surface_remove`, `mesh_surface_update_*_region`, `instance_create2`, `free_rid`:
  <https://docs.godotengine.org/en/4.7/classes/class_renderingserver.html>
- ConcavePolygonShape3D: <https://docs.godotengine.org/en/4.7/classes/class_concavepolygonshape3d.html>
- PhysicsServer3D `shape_set_data` (`SHAPE_CONCAVE_POLYGON` data shape):
  <https://docs.godotengine.org/en/4.7/classes/class_physicsserver3d.html>
- Spatial shader `CUSTOM0`–`CUSTOM3`:
  <https://docs.godotengine.org/en/4.7/tutorials/shaders/shader_reference/spatial_shader.html>
