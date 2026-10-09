# Voxel Engine Decision Log

This is the **non-normative** companion to `voxel-design-doc.md`. It records what changed between versions, why, and which alternatives were rejected. If the two documents disagree, the design document wins.

Entries are newest first. Each entry names the decision, the alternatives considered, and the reason for the choice.

---

## DR-014 — v3 audit: lock caps, phase counts, and the 26-neighbor refresh

**Context.** The v3 audit found that `max_edit_phase_core_chunks = 32` with `max_edit_lock_chunks = 96` could not both hold: the 26-neighbor closure of 32 core chunks is at least 144 (a 4×4×2 block gives 6×6×4), and even a 3×3×3 block (125) exceeds 96. An ordinary explosion therefore became several visibly half-applied phases.

**Decision.** Remove the resident apron cache entirely (DR-011). A phase's lock set becomes exactly its owner-chunk set, so `max_edit_lock_chunks` is deleted and the cap is the phase cap by construction. The dirty set is computed from the window rule (DR-013) rather than from a derived cache refresh.

**Rejected.** Raising the lock cap to 160+ and accepting many-phase explosions. This keeps a large lock set resident, which is the failure mode the cap was introduced to prevent, and it makes an ordinary edit visibly half-applied on screen.

---

## DR-013 — Dirty rule: value change, not sign change

**Context.** v3 marked a chunk dirty only when a derived sample "changed sign." Surface Nets vertex positions interpolate edge crossings from sample *values*, so a value-only change on a boundary sample moves vertices in the neighbor's cells and leaves a crack against the freshly remeshed chunk.

**Decision.** Replace the sign rule with a window rule: **a chunk is dirty if any canonical sample inside its mesh window (`−1..N+1` per axis) changed value.** There is no sign-only exception. The mesh window is the `3³` block of owner chunks centered on the chunk, so a canonical sample written in chunk `C` dirties `C` and its 26 neighbors.

**Consequence.** The dirty set is computed from the write set, not from a cache refresh, which is what makes DR-011 possible.

---

## DR-012 — Quad emission convention

**Context.** v3 never stated which chunk emits quads on a boundary edge, or whether a chunk recomputes the vertex for cell −1. If it did, corner-gradient normals there need sample −2, which makes the 35³ window one short.

**Decision.** A chunk emits geometry **only for the `N³` cells it owns** (local `0..N-1`). It never emits the cell at local index `N`, and it never computes a vertex for a cell it does not own. Therefore no sample at `−2` or `N+2` is ever read, and the `(N+3)³` mesh window is exactly sufficient: cell 0's corners are samples `0..1` and its gradients need `−1` and `+1`; cell `N-1`'s corners are `N-1..N` and its gradients need `N-1` and `N+1`.

Adjacent chunks emit separate quads for adjacent cells. Those quads share an edge, and both chunks compute that edge's position from the same canonical owner blocks, so the shared vertices coincide exactly. The implementation must not introduce per-chunk variation (for example a different summation order) into the vertex computation.

**Also fixed.** The multipart equivalence test is **2×2×2** chunks of 32³ against one 64³ region, not the 2×2 v3 specified, which cannot exercise the Z axis.

---

## DR-011 — No resident apron cache

**Context.** v3 stored a 35³ padded sample window per chunk, of which only 32³ was authoritative; the rest was derived cache that had to be refreshed on every edit, generate, load, and unload across the 26-neighbor closure. The audit noted the doc already conceded the cache was not needed for correctness, since Mesh rebuilds from canonical owners.

**Decision.** Delete the cache. A chunk stores exactly one immutable `N³` core block (DR-010). Every Mesh, Persist, and navigation read builds its own private window from canonical owners.

**Benefits.** Dense memory drops from 83.7 KiB to **64 KiB** per chunk (~24%). Edits lock only canonical owners. The 26-neighbor apron refresh, the 96-lock cap, and the phase-cap contradiction (DR-014) all disappear.

**Cost.** A Mesh job resolves up to 27 owner chunks and copies up to ~1.7 MB to produce a 70 KB window. Tracked as a risk (design doc §8 item 3); the mitigation is a per-chunk neighbor-pointer cache, not a return to the apron.

**Rejected.** Keeping the apron as a write-through cache to reduce Mesh reconstruction cost. It reintroduces the refresh paths and the lock-closure problem for a cost that is bounded and measurable.

---

## DR-010 — Copy-on-write sample blocks

**Context.** v3's Persist version bracket was a data race: copying non-atomic samples while an Edit publishes is UB even if the result is discarded, and the mandatory TSan gate would flag it. The audit's suggested fix was a shared-lock copy of 64 KiB.

**Decision.** Make the authoritative block immutable and swap it atomically. A chunk holds `std::atomic<SampleBlock*>`; a publish is a store plus a version bump under the exclusive write lock. Readers take a reference to the block and read without any lock.

**Protocol.** A reader takes the store's shared lock, resolves the `Chunk*` handles, and increments each block's refcount, then releases the store lock. Because the refcount increment happens under the store lock, a block cannot be freed between resolution and acquisition, and a chunk handle cannot be recycled underneath the reader. A block outlives the chunk that published it.

**Consequences.** The Persist version bracket is deleted rather than fixed — there is nothing to bracket. The Mesh version check remains, but as an optimization to avoid uploading superseded geometry, not as a data-safety mechanism.

**Edit cost.** Each owner chunk's block is copied once per transaction (each owner appears in exactly one phase). That is a 64 KiB `memcpy` per mutated chunk, negligible against the meshing work that follows.

**Rejected.**
* Shared-lock copy (the audit's suggestion). Correct, but it makes Persist contend with Edit on every chunk and still needs a retry path if a publish lands mid-copy.
* Mutable block with a per-chunk read-write lock. Readers would still need to hold a lock across a 64 KiB copy, and a lock held across meshing work is exactly the contention the design is trying to avoid.
* Chunk-level `std::shared_ptr<const SampleBlock>` for the atomic swap. `std::atomic<std::shared_ptr<T>>` is not lock-free in C++17 and is not guaranteed atomic in C++20; a raw pointer with an intrusive refcount is simpler and sufficient.

---

## DR-009 — Sentinel encoding must be reachable

**Context.** v3 clamped the quantized encoding to `±32767` but reserved `INT16_MIN` and `INT16_MAX` as the solid/air markers. `INT16_MIN` could never be produced, so marker-based `SOLID` detection was inconsistent.

**Decision.** Clamp into the full `int16_t` range: `q = clamp(round(sdf / δ * 32767), INT16_MIN, INT16_MAX)`, decode `sdf = q * δ / 32767`. Both markers are reachable, marker detection is exact equality, and a round-trip is exact across the whole range.

---

## DR-008 — Revision-to-readiness watermark

**Context.** v3 had no way to map a `terrain_revision` to the mesh versions a navigation group's source must contain. Concurrent phases could publish revisions out of order, so `desired_terrain_revision` could move backwards, and "group is ready" was undecidable.

**Decision.**
1. Every `EditPhaseCompleted` record carries `(chunk_key, new_version)` pairs for each chunk the phase published — bounded by `kMaxEditPhaseChunks` (32), so the record stays small.
2. The wrapper max-merges each pair into every required group's `required_mesh_version[c]`. Max-merge also applies to `desired_terrain_revision` and `desired_source_epoch`.
3. A group is bake-ready only when every source chunk `c` (owned footprint plus border padding) is resident and `committed_mesh_version[c] >= required_mesh_version[c]`.

---

## DR-007 — Navigation quiescence

**Context.** v3 coupled navigation to chunk commit with no spatial bound. Every Load/Generate mesh commit bumped a group's `source_epoch`, so frontier groups chased their desired state indefinitely while the player moved, `fully_committed()` could never become true, committed mesh buffers were released at commit (so bakes had to read them back from `RenderingServer`, slowly, and outside the memory ledger), and "required group" was undefined.

**Decision.**
1. **Decouple `chunk_committed` from navigation.** It fires when the visual mesh and collision are live. A separate `navigation_group_synced` signal reports the navigation half.
2. **Bound the scope.** A group is *required* only when its footprint intersects `nav_radius_chunks` around a registered agent position. Predicates range over required groups only.
3. **Debounce rebuilds.** `nav_rebuild_debounce_ms` (default 200) of quiet, never more than `nav_rebuild_max_delay_ms` (default 1000). `request_navigation_flush()` submits immediately.
4. **Retain committed mesh for the bake.** A `NavSourceCache` holds the POD arrays of committed chunks in required groups until the group's bake synchronizes, charged to the memory budget.

**Also fixed.** The v3 footprint typo: a 4×4-chunk group covers a **4 × 4 chunk** footprint, not "16×16".

---

## DR-006 — Aging must not flatten priorities

**Context.** v3's aging rule promoted a job one lane every 250 ms "capped at the top lane." At startup, thousands of Generate jobs all became Edit priority after ~1 s, contradicting "Generate is lowest."

**Decision.**
1. Aging is capped at **2 lanes of promotion**: Generate can reach Persist priority, never Load or above.
2. Generate admission is bounded: `kMaxConcurrentGenerateJobs = 2` in flight, `kMaxGenerateQueueDepth = 8` queued. Further requests stay in the streaming request queue.
3. Persist is never aged past Mesh or Load.

---

## DR-005 — Persist under memory pressure

**Context.** v3's eviction policy could wedge: the budget is hard, edited chunks cannot evict until persisted, Persist was guaranteed only about once per second, a failed save pinned memory forever, and the persist unit was a whole-region rewrite.

**Decision.**
1. **Persist pressure boost.** At ≥ 90% budget utilization, Persist is promoted to the Mesh lane until pressure clears. This is the only condition under which Persist outranks Mesh.
2. **Retry with backoff.** Up to `kMaxPersistRetries = 5` attempts (100/200/400/800/1600 ms), each re-acquiring the block reference.
3. **Pin and report.** After the retry budget, the chunk is pinned (exempt from eviction, charged to `edit_pin_reserve`, default 15% of the budget) and `data_loss_warning` is emitted. A transient I/O error clears on the next flush.
4. **Overflow policy.** If pinned bytes exceed the reserve: keep the boost, shrink the active radius to release *unedited* chunks, suppress prefetch. Only then evict an unpersisted edited chunk, with `data_loss_warning`. Silent data loss is not permitted.
5. **Region size 16×16** (was 32×32), because flush rewrites the whole region and rewrite cost is proportional to *edited* chunks in it.

**Also fixed.** The region header now carries the **world seed** and **generator version**; a mismatch rejects the file rather than allowing regenerated unedited neighbors to seam against saved edited chunks after a noise or SIMD change.

**Rejected.** An append-only chunk journal with an atomically replaced index. It moves the crash-safety problem from the region file to the journal index and requires a merge pass on load. Smaller regions bound the rewrite cost with a strictly simpler format. Revisit if flush cost still dominates (design doc §8 item 10).

---

## DR-004 — Commit throughput is a requirement

**Context.** v3 had no commit-throughput target. At roughly 1–2 chunks per frame, a 1000-chunk crater or a cold start takes on the order of 10–15 seconds, and the risky unknowns (commit cost, collision cost, nav bake and API) were validated last.

**Decision.**
1. The default commit path is `RenderingServer` surface creation from worker-built POD arrays, not `ArrayMesh` (kept behind a property for A/B).
2. **Target:** ≥ 8 dense-chunk commits per frame within a 4 ms budget on reference hardware, with vertex upload and collision shape creation measured independently.
3. **Vertical slice in M0:** generate → mesh → commit → collide runs in Godot before the full contract is built, and `kChunkSize` is measured at both 32 and 16.

**Rationale.** The main-thread cost of `ArrayMesh` construction is the largest single commit cost and is not avoidable by tuning. Validating it last is how a design ships a 15-second crater load.

---

## DR-003 — Editor and lifecycle guards

**Context.** GDExtension classes run in the editor unless guarded. v3's `_enter_tree` started the pool and streamed, `save_on_exit` could write region files from the editor, a paused tree stalled `_process` so nothing drained, and reparenting the node triggered a full shutdown/save/recreate.

**Decision.**
1. `_enter_tree`, `_process`, and `_exit_tree` all return early when `Engine::get_singleton()->is_editor_hint()` is true. No pool, no streaming, no saves from the editor; `save_world()` is the only editor save path.
2. `set_process_mode(PROCESS_MODE_ALWAYS)` so the drain loop survives a paused tree.
3. Reparenting is documented as a full shutdown and recreate: correct but expensive, and it discards resident state.

---

## DR-002 — Chunk size is a compile-time constant

**Context.** Open question 1 in v3 asked whether to use 32³ or 16³, but M0 built everything on 32³ regardless.

**Decision.** `kChunkSize` is a compile-time constant (default 32) with every formula in the design doc written as `N = kChunkSize`. The M0 spike builds and measures both 32 and 16; the default changes only on measured evidence. Tests run at both sizes.

---

## DR-001 — Generation homogeneity pre-pass

**Context.** Discovering `EMPTY`/`SOLID` in v3 required evaluating all 32³ × 4 noise samples, which dominates generation cost in an air-heavy world.

**Decision.** A `coarse_bound(chunk)` pre-pass runs before the full evaluation. The default implementation uses a 2D heightfield pass (`N²` evaluations) plus a vertical probe set, and returns `DEFINITELY_AIR` / `DEFINITELY_SOLID` / `UNKNOWN`. It must be **conservative**: only a homogeneous result may be short-circuited, with a margin of at least `2δ` beyond the truncation band; `UNKNOWN` always falls through to the full evaluation. A test compares the pre-pass against full evaluation over a seeded corpus.

---

## Version history

### v3 → v4

| Area | v3 | v4 |
| :--- | :--- | :--- |
| Sample storage | 35³ padded window per chunk, 32³ authoritative + derived cache | One immutable `N³` core block per chunk, no derived cache |
| Reader synchronization | Shared/read locks, version bracket in Persist | Refcounted reference to an immutable block; no reader locks, no bracket |
| Edit lock set | Core owners + 26-neighbor dependency closure, capped at 96 | Owner chunks only, capped at `kMaxEditPhaseChunks` (32) |
| Dirty rule | Derived sample "changed sign" | Any changed canonical sample in the chunk's `−1..N+1` window |
| Dense memory | 83.7 KiB (35³ × 2) | 64 KiB (32³ × 2) |
| Chunk size | Fixed 32³ | Compile-time `kChunkSize`, default 32, measured at 16 |
| Region size | 32×32 chunks | 16×16 chunks |
| Region header | Magic, version, coords, TOC | Plus mandatory world seed and generator version |
| Nav group state | In the Godot wrapper | Pure C++ `NavGroupController` behind `INavServerSync` |
| Nav readiness | Undecidable from reported data | `(chunk, version)` watermark in `EditPhaseCompleted` |
| Nav scope | All live groups, coupled to chunk commit | Required groups within an agent radius, debounced |
| `chunk_committed` | Gated on navigation sync | Emitted at visual/collision commit; `navigation_group_synced` is separate |
| Persist starvation | ~1 job/second, no failure policy | Pressure boost, retry with backoff, pin + `data_loss_warning` |
| Aging | Uncapped, promoted to the top lane | Capped at 2 lanes; Generate admission bounded |
| Commit path | `ArrayMesh` default, `RenderingServer` opt-in | `RenderingServer` default; ≥ 8 chunks/frame target |
| Editor lifecycle | Unbounded editor instantiation | `is_editor_hint` guards, always-process, documented reparent |
| Generation | Full `N³ × 4` evaluation always | Conservative homogeneity pre-pass first |
| Residency math | `(2r+1)³` | `(2r+3)³` active + mesh apron |

### v2.0 → v2.1

| Area | v2.0 | v2.1 |
| :--- | :--- | :--- |
| Concurrency | Job versioning only | Normative concurrency and ownership contract: single writer, snapshot reads, reference-counted unload |
| Coordinate model | 32³ voxels, 34³ samples | 32³ core cells, 32³ canonical samples, 35³ padded volume |
| Sample ownership | Apron and core symmetric | Canonical 32³ sample ownership; 33³ local corner window includes derived high-boundary slots |
| Apron sync | Edit path only | Edit, generate, load, and unload paths all defined |
| Noise density | Implicitly treated as SDF | Density normalized to an SDF with explicit scale and gradient magnitude |
| Residency | 118 KB per chunk always | EMPTY/SOLID chunks use a compact representation; material channel lazily allocated |
| Core linkage | Separate DLL, C++ ABI across boundary | Static linkage into the extension, or a C API if a DLL is required |
| Navigation data | Per-chunk regions, unspecified lifecycle | Region groups with defined lifecycle and border padding |
| Save consistency | Unspecified | Version-bracketed serialization with retry-or-discard; 32³ canonical samples only |
| Shutdown | Persist flushed after join | Single ordered sequence, drain specified by thread |
| Signals | `explosion_applied` ambiguous | `explosion_requested` / `explosion_completed` |
| Readiness | `is_idle()` conflated | `core_idle()` and `fully_committed()` |
| Scheduler | Priority only | Fairness, queue/memory backpressure, edit-phase limits, and dirty-set mesh admission |
| Streaming / residency | `view_distance_chunks` declared but residency policy unspecified | Explicit desired set, prefetch, load/generate routing, 26-neighbor dependency closure, eviction, hard memory budget, and queue interaction |
| Thread start | Constructor | `_enter_tree` |
| Navigation sync | `navigation_groups_synced()` referenced an undefined terrain revision/group state | Explicit global `terrain_revision`, per-group revision/epoch state, group dependency closure, and sync rules |
| Edit locking | One edit could hold one lock set spanning an arbitrarily large explosion | Large edits are partitioned into bounded phases; each phase has a complete lock set and releases all locks before downstream backpressure |
| Completion coupling | Edit directly enqueued Mesh work, so one large edit could flood completion state | Edits mark chunks mesh-dirty; Mesh admission consumes dirty work under queue/memory caps |
| Sample ownership | 33³ "core" samples were all described as authoritative despite shared chunk-boundary coordinates | Canonical 32³ sample ownership with a 33³ local mesh-sample window; boundary/high-face slots are derived |
| Neighbor closure | Six face-neighbors were treated as sufficient for apron/mesh correctness | Full 26-neighbor dependency closure is defined for cached padded samples |
| Load mutation | Load reconstructed authoritative samples even though only Edit/Generate were allowed to mutate them | Load has a dedicated initialization write path before a chunk becomes Resident |
| Edit coalescing | "Covering job" could be read as changing the CSG shape | Coalescing preserves the exact union of sphere CSG terms; it never replaces them with a bounding sphere |
| Navigation source | Chunk completion mixed per-chunk nav data with group-level bake semantics | Mesh jobs produce reusable render/collision source; navigation rebuilds consume committed chunk meshes at region-group scope |

### v1 → v2

| Area | v1 | v2 |
| :--- | :--- | :--- |
| Threading | Explosion path runs synchronously on the main thread | Async job pipeline with an MPSC completion queue and a per-frame GPU upload budget |
| Lifetime | `delete core_engine` in destructor | Explicit ordered teardown with the acting thread named per step |
| Meshing | Transvoxel with no LOD structure | Uniform-resolution Surface Nets for v2.0. Transvoxel deferred to an LOD milestone |
| Chunk layout | 32³ core, no border data | 32³ cells, 33³ core-sample window, 35³ padded volume. Edits propagate to neighbor aprons |
| SDF format | `int8_t` or `float`, unnormalized | Truncated SDF (TSDF) stored as normalized `int16_t` with a fixed truncation band |
| Chunk keys | Signed 21-bit packing, sign extension unspecified | Offset encoding, no sign extension |
| Storage index | "Octree" in diagram, hash map in text | Flat open-addressing hash map. No octree in v2.0 |
| Collision | Not specified | Dedicated collision pipeline |
| Navigation | Standalone `dtTileCache` | Godot `NavigationServer3D` as the single navigation authority |
| Save format | RLE then LZ4 on float SDF | Homogeneous-chunk forms, quantized `int16_t`, byte-plane split, LZ4, crash-safe region files |
| Godot API | `PackedVector3Array::to_writer()` (nonexistent) | `resize()` + `ptrw()`, optional direct `RenderingServer` path |
| Registration | `VoxelAICompanion` not registered | Both classes registered at scene init level |
| Core linkage | Separate core DLL, C++ ABI across boundary | Core statically linked into the extension |
| Library output | Windows, Linux, macOS x86_64 release only | Debug and release entries, arm64 for macOS |

---

## Superseded v3 audit findings and where they landed

The v3 pre-implementation audit raised fourteen findings. Each is resolved as follows; the normative text is in `voxel-design-doc.md`.

| # | Finding | Resolution |
| :-- | :--- | :--- |
| 1 | Dirty rule only tracked sign changes; value-only changes crack seams | DR-013: window rule on value change. §3.2 Dirty Propagation Rule |
| 2 | Quad-emission convention unspecified; 35³ possibly one sample short | DR-012. §3.5 Meshing; test updated to 2×2×2 |
| 3 | Sentinel mismatch: clamp `±32767` vs markers `INT16_MIN/MAX` | DR-009. §3.1 Sample Format |
| 4 | Persist version bracket is a data race | DR-010. §3.2 Read Rule and Save Consistency Rule |
| 5 | Phase caps could not both hold (96-lock vs 144-chunk closure) | DR-011 + DR-014. Lock set = owner set; cap deleted |
| 6 | Navigation footprint typo ("4×4" then "16×16") | DR-007. §4.5 Navigation Group Ownership |
| 7 | Revision-to-readiness gap; out-of-order revisions; no version mapping | DR-008. §4.5 Terrain Revision and Per-Group State |
| 8 | Aging flattens priorities | DR-006. §3.4 Fairness Rules |
| 9 | Eviction can wedge; no persist failure policy; region lacks seed/generator version | DR-005. §3.3 and §3.6 |
| 10 | Navigation never quiescent; required group undefined; source retention | DR-007. §4.5 Navigation Maintenance Radius |
| 11 | Commit throughput has no target; risky costs validated last | DR-004. §4.1, §4.3, §7.3, M0 |
| 12 | Editor and lifecycle guards missing | DR-003. §4.2, §5.2 |
| 13 | Generation cost; residency math off by the dependency ring | DR-001 and §3.3 mesh apron (`(2r+3)³`) |
| 14 | Doc hygiene: interleaved changelogs, typos, stray paragraphs, hardcoded path, overlapping window names | This split: normative spec + decision log. Window names unified to *core* and *mesh window* |

---

## Open log items

These are tracked here rather than in the normative doc, and each has a owner and a milestone.

| Item | Question | Owner | Milestone |
| :--- | :--- | :--- | :--- |
| L-01 | Does the `RenderingServer` surface path meet the 8 chunks/frame target at `N = 32`? If not, which fallback applies (collision decimation, then `N = 16`)? | Core | M0 spike |
| L-02 | Does a lower-level `mesh_add_surface` overload exist at the pinned godot-cpp version, and does it avoid the intermediate packed arrays? | Wrapper | M0 spike |
| L-03 | Can the material channel be passed as a custom attribute at the pinned version, or is a per-chunk shader parameter required? | Wrapper | M0 spike |
| L-04 | Does the Mesh window reconstruction cost (27 owner resolutions) show up in profiling? If so, design the neighbor-pointer cache. | Core | M1 |
| L-05 | Is 16×16 the right region size once flush cost is measured against edited-chunk counts? | Core | M4 |
| L-06 | Does the `NavSourceCache` dominate the memory budget at the default nav radius? If so, fall back to reading triangles from `RenderingServer`. | Wrapper | M4 |
