# High-Level Design Document (HLD) v4
## Project Core Architecture: Voxel Engine Library with Godot GDExtension Integration

**Status:** Draft v4.0
**Supersedes:** Voxel Design Doc v3
**Review input:** v3 pre-implementation audit (14 findings, reproduced in the decision log)

This document is the **normative specification**. The full version history, the audit findings, and the alternatives that were rejected are in `voxel-decision-log.md`. If this document and the decision log disagree, this document wins.

Normative words are used with their usual RFC 2119 weight: **must** is a correctness invariant, **should** is a default that can be overridden with a recorded reason.

---

## 1. Executive Summary

### 1.1 Objective

Design a cross-platform, high-performance, **modular voxel engine written in C++** as a Godot-independent core library that is statically linked into the GDExtension for v4.x. The core has no dependency on Godot. The first integration wrapper targets **Godot 4.x** through the **GDExtension API** using **godot-cpp**.

### 1.2 Core Capabilities

* **Explosions:** Spherical CSG subtraction on the TSDF, applied by worker threads in bounded, lock-limited phases. Results reach the screen within a bounded number of frames.
* **AI Companion Navigation:** Navigation updates for modified regions are asynchronous and use Godot's `NavigationServer3D`.
* **Persistent Saves:** Chunk serialization with homogeneous-chunk elision, quantized SDF, and LZ4 block compression, stored in region files.
* **Procedural Generation:** Multi-threaded, deterministic terrain generation with **FastNoise2**, with a cheap homogeneity pre-pass.
* **Physics:** Static collision generated from the same meshes as the visuals, updated asynchronously when chunks are remeshed.

### 1.3 Non-Goals

* LOD and Transvoxel transitions (planned for milestone M5).
* Multiplayer replication of voxel edits.
* Non-voxel material blending beyond a single material ID per voxel. A material ID channel is reserved in the chunk format (see 3.1).
* Copy-on-write blocks spanning material edits differently from TSDF edits. Material lives in the same immutable block.

### 1.4 What changed in v4

The v4 model removes two pieces of state that v3 carried and that v3's own audit showed to be the source of most of its complexity and its worst races:

1. **No resident apron cache.** A chunk stores exactly one immutable block of 32³ canonical samples. Every mesh, persist, and navigation read reconstructs its own private window from canonical owners. The 26-neighbor *apron refresh* job disappears entirely.
2. **Copy-on-write sample blocks swapped atomically.** Readers (Mesh, Persist, navigation source) take a reference to an immutable block and never wait, never copy under a lock, and never share a live buffer with a writer. This deletes the v3 version-bracket in Persist, which was a data race (non-atomic copy of samples under concurrent mutation) and is also unnecessary once blocks are immutable.

Consequences that the rest of this document relies on:

* An edit phase locks **only** the canonical owner chunks it writes. There is no dependency closure to lock. The `max_edit_lock_chunks` cap from v3 is gone.
* The dirty rule is a *window* rule: any changed sample inside a chunk's `−1..33` window dirties that chunk (3.2, Dirty Propagation Rule).
* Chunks are smaller in memory: 32³ × 2 B = **64 KiB** dense, not 35³ × 2 B = 83.7 KiB.
* Nav groups have a concrete per-chunk committed-version watermark, so "group is ready" is decidable from data the core actually reports (4.5).

---

## 2. System Architecture Overview

The architecture enforces a strict boundary: the **Core** owns state, simulation, meshing, and persistence. The **Godot layer** owns presentation, physics bodies, navigation registration, and editor integration.

```
+-----------------------------------------------------------------------+
|                            GODOT ENGINE 4.x                           |
|  +---------------------------+       +-----------------------------+  |
|  |    VoxelTerrain3D Node    |       |      VoxelAICompanion       |  |
|  +-------------+-------------+       +--------------+--------------+  |
|                |  RenderingServer / PhysicsServer3D  |  NavigationAgent3D |
+----------------|-------------------------------------|------------------+
                 |                                     |
+----------------|-------------------------------------|------------------+
|                GDEXTENSION WRAPPER LAYER (godot-cpp)                  |
|  - Marshals worker-built POD buffers into RenderingServer              |
|  - Owns Godot node lifetime, main-thread commit, budget enforcement    |
|  - Owns the pure-C++ nav group state machine + server-sync adapter     |
+----------------------------------|-------------------------------------+
                                   | Pure C++ API (no Godot types)
+----------------------------------|-------------------------------------+
|                        CORE VOXEL ENGINE (libvoxel_core)               |
|  ChunkStore     WorkerPool       MeshGen          SaveSystem           |
|  (hash map,     (5 priority      (Surface Nets   (quantize + LZ4,     |
|   COW blocks)   lanes, aged)     from canonical   region files)        |
|                                  owner blocks)                         |
|                                                                        |
|  ProcGen (FastNoise2)     Edit (CSG on TSDF)     Completion queue      |
+-----------------------------------------------------------------------+
```

Section 3.2 (**Concurrency and Data Ownership Contract**) is normative for every box above. It is not a detail to resolve during implementation.

---

## 3. Core Engine Pipeline (Pure C++)

The core operates on raw memory arrays, math structs, and explicit task scheduling. It has no knowledge of Godot.

### 3.1 Data Layout and Chunk Store

#### Chunk Size is a Compile-Time Constant

`kChunkSize` is a compile-time constant, default **32**. Every formula below is written with `N = kChunkSize` so the implementation is parameterized; v4 tests run at both `N = 32` and `N = 16`.

| Constant | Default | Meaning |
| :--- | :--- | :--- |
| `kChunkSize` | 32 | Cells (and canonical samples) per chunk axis |
| `kChunkMask` | `N - 1` | For fast local indexing |
| `kChunkLog2` | `log2(N)` | For fast division where a power of two is required |

#### Coordinate Conventions

All positions below are **world sample coordinates**: continuous real-valued positions. Sample grid points are integers. The world origin sits at sample `(0, 0, 0)`.

A chunk with chunk coordinate `C = (Cx, Cy, Cz)` owns a `N³` block of **cells**. Cell `(i, j, k)` with `i, j, k ∈ [0, N)` has minimum corner at world sample:

$$p_{\min} = N \cdot C + (i, j, k)$$

The SDF sample `S` used by that cell at corner index `(a, b, c)` is at world sample `N \cdot C + (i + a, j + b, k + c)`, with `a, b, c ∈ {0, 1}`.

Consequences that implementations must honor:

* **Cells, not samples, are the unit of ownership.** A chunk owns `N³` cells along each axis, indexed `0..N-1`.
* **Voxel indices are ambiguity traps.** "`N × N × N` voxels" is not used in this document. Use the terms **cell** and **sample**.
* **Ownership is inclusive of the low face, exclusive of the high face.** Cell `N-1` is the last cell a chunk owns. The neighbor chunk's cell 0 abuts it. No cell is owned twice and none is skipped.

#### Window Names

v3 used three overlapping names for its sample windows (`33³ core`, `35³ padded`, "canonical"). v4 uses exactly two, and they are not interchangeable:

| Name | Size | Where it lives | Who reads it |
| :--- | :--- | :--- | :--- |
| **core** (canonical block) | `N³` = 32³ | Resident per chunk, immutable, atomically swapped | Everyone; it *is* the authority |
| **mesh window** | `(N+3)³` = 35³ | Private to one Mesh job, built on a worker | Surface Nets, then discarded |

There is **no third window**. v3's "33³ local corner window with derived high-boundary slots" is gone: the only stored sample data per chunk is its `N³` core. Anything outside it is either another chunk's core or an absent sample.

#### Sample Format

Truncated SDF stored as normalized `int16_t`.

* Truncation band: ±δ, with δ = 2.0 voxel units (configurable at build time).
* Encoding: `q = clamp(round(sdf / δ * 32767), INT16_MIN, INT16_MAX)`. Decoding: `sdf = q * δ / 32767`.
* Sign convention: **negative = solid, positive = air.** The surface is the `sdf = 0` isosurface.
* `INT16_MIN` (−32768) and `INT16_MAX` (32767) are the clamped solid and clamped air markers. They mean "far beyond the band," not an exact distance. Because the encoder clamps into the full `int16_t` range, both markers are **reachable** values, and marker detection is exact equality. v3 clamped to `±32767` while reserving `INT16_MIN`, which made `SOLID` detection inconsistent; that clamp is corrected here.
* Quantization step is `δ / 32767`. A round-trip through encode/decode is exact for `q ∈ [INT16_MIN, INT16_MAX]`.

**Absent samples.** A world sample whose canonical owner chunk is not resident (and is not known-absent-outside-world) is *not* defined. The mesh window is only ever built after the owner chunks are resident or known absent (3.3), so every sample in a mesh window is either a real core sample or resolved by this rule:

| Owner location | Substitute value |
| :--- | :--- |
| Resident chunk | Its core sample |
| Outside the world on any horizontal side, or above the top of the world | `INT16_MAX` (air) |
| Below `y = 0` | `INT16_MIN` (solid) |

The solid floor rule prevents a visible surface at the bottom of the world.

#### Compact Residency Representation

A chunk's sample data is stored in one of three forms, and the form is updated as the chunk changes:

| Form | Memory | Meaning |
| :--- | :--- | :--- |
| `EMPTY` | 0 bytes of sample data | Every **canonical owned sample** is the clamped air marker |
| `SOLID` | 0 bytes of sample data | Every **canonical owned sample** is the clamped solid marker |
| `DENSE` | `N³ × 2` = **64 KiB** | Full `int16_t` core sample array |

* A chunk that generates as `EMPTY` or `SOLID` alloc no sample array. It must still record its form so it can be promoted to `DENSE` when an edit or a neighbor edit touches it.
* Promotion to `DENSE` **allocates a new block filled with the marker of the previous form**, then applies the edit to that block. Under copy-on-write this is the same operation as an edit on a `DENSE` chunk.
* This matters because v2/v3 allocated a full padded array for every resident chunk, making an air-heavy world cost far more memory than its actual voxel content justifies.

#### Copy-On-Write Sample Blocks

This is the central data structure of v4. The full ownership contract is in 3.2; this section defines the object.

```cpp
struct SampleBlock {
    // Immutable after publish. One contiguous allocation:
    //   [SampleBlock header][int16_t tsdf[N^3]][uint8_t material[N^3] (optional)]
    std::atomic<int32_t> refs;
    uint32_t             magic;        // kSampleBlockMagic, sanity check
    const int16_t*       tsdf;         // N^3, never null
    const uint8_t*       material;     // N^3 or null; allocated iff any material != 0
};
```

* A `SampleBlock` is **immutable once published**. There is no in-place mutation of a published block.
* A chunk publishes a new block by `samples.store(block, memory_order_release)` while holding its exclusive write lock. The load-side must be `memory_order_acquire`.
* `EMPTY` and `SOLID` are represented by two process-wide immutable singletons (`kSolidBlock`, `kAirBlock`) with a refcount that is never allowed to reach zero. Chunks in those forms store the singleton pointer and allocate nothing. This keeps the reader path uniform: `samples.load()` is never null.
* A published block with no remaining references is freed by the last releaser (refcount reached zero) and its bytes are released to the memory ledger. **A block therefore outlives the chunk that published it**, which is what makes lock-free reads safe.

#### Material Channel

* Stored as a parallel `uint8_t` per **canonical owned sample**, inside the same immutable block.
* No material array exists while every material in a chunk is 0; the pointer is null and the allocation omits the 32 KiB plane. The first write of a non-zero material allocates a **new block** with a material plane.
* Derived boundary material values do not exist: a mesh window reads material from the same canonical owners it reads TSDF from.
* Chunk memory per resident chunk is therefore **64 KiB for the core TSDF block**, plus **32 KiB** when materials are present. v3's 83.7 KiB padded-window cost is removed.

#### Chunk Key

64-bit, offset-encoded. Each axis is stored as `coord + 2^20` in 21 bits, giving a range of [-1,048,576, 1,048,575] chunk coordinates per axis.

```cpp
struct ChunkKey {
    uint64_t raw;

    static constexpr int64_t kBias = 1LL << 20;
    static constexpr uint64_t kMask = (1ULL << 21) - 1;

    static ChunkKey from_coords(int32_t x, int32_t y, int32_t z) {
        uint64_t ux = static_cast<uint64_t>(static_cast<int64_t>(x) + kBias) & kMask;
        uint64_t uy = static_cast<uint64_t>(static_cast<int64_t>(y) + kBias) & kMask;
        uint64_t uz = static_cast<uint64_t>(static_cast<int64_t>(z) + kBias) & kMask;
        return ChunkKey{ ux | (uy << 21) | (uz << 42) };
    }

    int32_t x() const { return static_cast<int32_t>(raw & kMask) - static_cast<int32_t>(kBias); }
    int32_t y() const { return static_cast<int32_t>((raw >> 21) & kMask) - static_cast<int32_t>(kBias); }
    int32_t z() const { return static_cast<int32_t>((raw >> 42) & kMask) - static_cast<int32_t>(kBias); }
};
```

Offset encoding is used rather than packing two's-complement coordinates, because `int21_t` does not exist in C++ and manual sign extension on unpack is a recurring bug source.

#### Storage Index

* Flat open-addressing hash map (for example `absl::flat_hash_map` or `robin_hood::unordered_flat_map`). Values are **pool handles, not `std::unique_ptr`**. Node-based `std::unordered_map` is not used.
* The store holds one **shared mutex** (`store_lock`). It is held for the duration of a handle resolution or a chunk insert/erase, and released before any block work. It is never held across meshing, noise, or disk IO.
* Chunks are **not freed directly**. Unload decrements a reference count; a chunk is only destroyed when its refcount reaches zero. A chunk with queued or in-flight jobs holding a reference cannot be freed (see 3.2).

#### Chunk Object

```cpp
struct Chunk {
    ChunkKey              key;

    // Authoritative, immutable, lock-free to read (see 3.2 Read Rule).
    std::atomic<SampleBlock*> samples;   // never null; singletons for EMPTY/SOLID
    std::atomic<uint64_t>    version;    // bumped on every authoritative publish

    std::shared_mutex         write_lock;  // Edit and initialization only
    std::atomic<int32_t>     refs;        // residency + job references

    ResidencyState            residency;   // see 3.3
    MeshState                 mesh_state;  // see below
    bool                      edited = false;
    PersistState              persist;     // see 3.5
};
```

#### Mesh/Visual State Machine

```
Resident -> Clean <-> Dirty -> Meshing -> MeshReady -> Committed
                             ^                                  |
                             +======= (version mismatch) =======+
                             +======= (unload: evict) ==========+
```

Each chunk has a monotonically increasing `version` counter (see 3.2). A mesh result is committed only if its captured version equals the chunk's current version. Otherwise it is discarded and the chunk returns to `Dirty`. This check is an **optimization to avoid uploading superseded geometry**, not a data-safety mechanism: under copy-on-write a snapshot can never be torn.

### 3.2 Concurrency and Data Ownership Contract

This section is **normative**. It defines exactly which state is authoritative, which jobs may write it, how large edit operations are partitioned, and how downstream backpressure interacts with editing.

#### Authoritative State

Per chunk, the authoritative state is:

* its **`SampleBlock*`** — the immutable `N³` core TSDF array plus its optional material plane;
* its **`version`** counter;
* its **residency state** and **reference count**.

The Core also owns a global `uint64_t terrain_revision`. It is defined precisely in section 4.5 and is used only to identify successful terrain-edit commits; it is not a replacement for per-chunk `version`.

#### Derived State

The following are caches derived from authoritative state and carry no weight in a dispute:

* Private mesh windows (`(N+3)³` snapshots) held by in-flight Mesh jobs.
* Meshed vertex and index buffers.
* Collision triangle buffers and Godot collision shapes.
* Navigation source data, navigation region resources, and navigation server state.
* Dirty flags and scheduler bookkeeping.

#### Mutation Rule

* **Resident chunks:** only **Edit** jobs may publish a new authoritative `SampleBlock` after a chunk is `Resident`, and only while holding that chunk's exclusive write lock. There is at most one writer per chunk at any time.
* **Initialization:** **Load** and **Generate** jobs may publish the initial authoritative block of a newly admitted chunk while that chunk is still in `Requested` / `Loading` / `Generating` state and before it becomes `Resident`. They must hold that chunk's exclusive write lock for the initialization and publish the `Resident` state only after the authoritative data is complete.
* Initialization is not a terrain edit and therefore does **not** increment `terrain_revision`.
* A publish is the pair `(samples.store(block), version.fetch_add(1))`, both performed while the exclusive write lock is held. The version bump must be visible to any thread that observes the new block pointer.

#### Read Rule

**Mesh, Persist, and navigation-source jobs never hold a chunk write lock and never read a block that can mutate.**

1. **Handle resolution.** A reader takes the store's shared lock, resolves the `Chunk*` for each required owner key, and increments the `SampleBlock` refcount for each block it will read. It then releases the store lock. Because the store lock is held across the refcount increment, a block cannot be freed between resolution and acquisition, and a chunk handle cannot be recycled underneath the reader.
2. **Reading.** Once a reader holds a block reference, it reads the block without any lock. The block is immutable, so this is race-free by construction. A reader may hold a block reference for the entire duration of a mesh, persist, or navigation-source operation.
3. **Mesh snapshot.** The job builds a private `(N+3)³` mesh window by copying the `−1..N+1` range of each of the 27 owner chunks' core blocks (or the absent-sample rule from 3.1). The snapshot is then immutable and decoupled from unload and from further edits.
4. **Persist snapshot.** The job serializes directly from the block it holds. It does **not** copy the block and does **not** take a version bracket. The v3 version bracket is deleted: it was a data race (a non-atomic copy of samples while an Edit published a new block) and it is unnecessary once blocks are immutable.
5. **Version capture.** A Mesh job records the `version` of every chunk contributing authoritative samples to its snapshot. A result is accepted only if all captured versions still match when the result is committed. This is an optimization, as described in 3.1.

#### Edit Phases and Lock-Set Bound

An edit request may span many chunks, but it is **never permitted to hold one lock set for the entire radius**.

* The edit planner first computes the complete set of canonical sample-owner chunks whose authoritative samples may change.
* The planner partitions that work into deterministic **edit phases**. The safety limits are:

| Constant | Default | Meaning |
| :--- | :--- | :--- |
| `kMaxEditPhaseChunks` | 32 | Maximum canonical owner chunks per phase |
| `kMinEditPhaseChunks` | 4 | Smallest phase the planner will produce when subdividing under memory pressure |

* **A phase locks exactly its owner-chunk set and nothing else.** v3's `max_edit_lock_chunks = 96` existed to bound the 26-neighbor apron-refresh closure. There is no apron cache in v4, so there is no closure to lock, and the lock cap is the phase chunk cap by construction. A test asserts this invariant (7.1).
* **Each phase computes its complete lock set before taking any lock.** All locks in that phase are acquired in ascending `ChunkKey::raw` order and released in descending order.
* There is **no partial lock acquisition inside a phase**, but a large edit is explicitly allowed to execute multiple such phases. This is the v4 resolution to unbounded lock residency in a single explosion.
* The phase must not wait for completion-queue capacity, memory admission, navigation synchronization, or main-thread work while any phase locks are held. Those checks occur **before the next phase begins or after all phase locks are released**.
* The current v4 edit workload is subtractive CSG, so phases are allowed to yield between one another without changing the intended result: each sphere contributes a monotonic `max(current, sphere_sdf)` operation. Future non-commutative edit types must define their own transaction semantics before using phased execution.
* Each owner chunk appears in **exactly one phase** of a transaction, so its block is copied and published at most once per transaction. The copy cost is `N³ × 2` = 64 KiB per mutated chunk, which is negligible against the noise and meshing work that follows.

#### Dirty Propagation Rule

v3 marked a chunk dirty only when a derived sample "changed sign." That rule is wrong for Surface Nets: vertex positions interpolate edge crossings from sample *values*, so a value-only change on a boundary sample moves vertices in the neighbor's cells and leaves a crack against the freshly remeshed chunk. v4 replaces it with a window rule:

1. A chunk's **mesh window** is the world-sample range `N·C − 1 .. N·C + N + 1` on each axis, i.e. the `3³` block of owner chunks centered on `C`.
2. **A chunk is dirty if any canonical sample inside its mesh window changed value.** There is no sign-only exception: any value change dirties the chunk.
3. After a phase publishes new blocks, the phase marks dirty: every owner chunk it wrote, plus the 26 neighbor chunk coordinates whose mesh windows include any of those chunks' canonical samples. Only resident chunks are marked; the rest are ignored (they will be meshed from canonical owners when they are admitted).
4. A chunk becoming `Resident` (Load or Generate) marks itself and its 26 neighbors dirty.
5. A chunk being unloaded marks its 26 neighbors dirty, because their mesh windows referenced it.
6. Dirty marking is scheduler bookkeeping. It is performed **after** the phase's locks are released, into a deduplicated dirty set protected by its own mutex. It never requires a chunk lock.

The `3³` rule is required because a corner sample participates in cells across face-, edge-, and corner-adjacent chunks. A six-neighbor-only rule is insufficient for 3D Surface Nets windows.

#### Version Rule

* Every authoritative publish to a **resident** chunk (TSDF or material) increments that chunk's `version` while its write lock is held.
* Load/Generate initialization publishes the initial block before the chunk becomes `Resident`; it establishes the starting version and does not count as an edit revision.
* A Mesh job captures the versions of all contributing chunks at snapshot time.
* A result is accepted only if every captured version equals the corresponding current version at completion time.

#### Save Consistency Rule

A Persist job serializing chunk `C` holds a reference to `C`'s current `SampleBlock` and serializes from it. Because the block is immutable:

* the serialized bytes are always a complete, self-consistent chunk state;
* there is no `version_before` / `version_after` bracket, no discard-and-retry, and no re-enqueue;
* a concurrent Edit publishing a new block does not affect the save in progress — the save writes the old state, which is a valid prior state of the chunk.

The v3 bracket is deleted rather than fixed. Its replacement (a shared-lock copy) was the review's suggestion; copy-on-write makes even that unnecessary.

#### Unload Rule

* A resident chunk owns one **residency reference**. Every dequeued job acquires an additional **job reference**; each job releases it when complete or discarded.
* `Unload` removes the residency reference and marks the chunk `EvictPending`. Actual destruction occurs only when the total reference count reaches zero.
* A queued job that resolves to a non-resident/evict-pending chunk before dequeue is dropped without execution.
* No chunk memory is freed while any job or completion result still holds a reference to it. A `SampleBlock` is freed by its own refcount, independently of the chunk that published it.

#### Completion Rule

* A heavy worker result owns its buffers (vertex data, index data, collision triangles/source data) until the main thread commits it.
* Worker code never touches Godot objects, RIDs, `Object`s, or Godot-owned allocator state.
* The MPSC completion queue is the single worker-to-main completion channel. It may contain both heavy chunk results and small edit-phase control events.
* Completion entries are charged against the same engine-managed memory budget used for streaming.

#### Edit-to-Mesh Handoff

Edit workers **do not enqueue one Mesh job per changed chunk directly**.

* A completed edit phase marks affected resident chunks `mesh_dirty` and inserts their keys into a deduplicated dirty set.
* The Mesh scheduler pulls dirty chunks only when completion-queue depth and completion-buffer bytes are below their admission caps.
* A chunk is removed from the dirty set only when its Mesh job is successfully admitted. If the job becomes stale, the chunk is marked dirty again.
* This decoupling means a single large explosion can mark thousands of chunks dirty without being able to overrun the MPSC completion queue in one phase.

#### Testing of This Contract

Every rule above maps to a unit or integration test in section 7.1/7.2. The TSan/ASan runs are mandatory CI gates.

### 3.3 Chunk Streaming and Residency Management

`view_distance_chunks` is a **residency target**, not a promise that the engine will allocate every chunk in that geometric range. The Streaming Manager maintains a bounded set of resident chunks around a moving focus point while honoring the hard memory budget below.

#### Streaming Inputs and Policy

* **Focus:** The Godot wrapper supplies the world-space focus position once per frame (normally the player or camera position) through a pure C++ `update_stream_focus()` call. The core converts it to the containing chunk coordinate and computes the desired residency set.
* **View distance:** `view_distance_chunks` is a 3D **Chebyshev radius** in chunk coordinates. A target radius `r` therefore contains at most `(2r + 1)^3` chunk coordinates before world-height and memory limits are applied.
* **Mesh dependency apron:** meshing any chunk in the active set requires the `3³` owner-chunk closure around it (3.2, Dirty Propagation Rule). The active set therefore requires `(2r + 3)^3` chunk positions to be resident or known absent. **This apron is mandatory, not prefetch.** v3's `(2r + 1)^3` budget math is corrected here.
* **Prefetch:** `prefetch_distance_chunks` (default `2`) extends the desired radius beyond the view radius. Prefetch is **soft**: it is admitted only when memory headroom exists after the active view and its mesh apron are resident. The prefetch shell therefore requires `(2r + 2p + 3)^3` positions in the worst case.
* **Motion bias:** Within the prefetch shell, chunks in the current movement direction are ranked ahead of lateral and rear chunks. The wrapper supplies an optional focus velocity; zero velocity falls back to distance-only ordering.
* **World bounds:** Y coordinates are clamped to `[0, world_height_chunks)`. X/Z have no additional streaming bound beyond the `ChunkKey` range.

#### Desired-Set Construction and Load Order

Each streaming update produces three sets:

1. **Active set:** chunks within `view_distance_chunks` of the focus.
2. **Mesh apron:** the 1-chunk shell around the active set, required to mesh it. Missing apron chunks outside the world are treated as absent and do not consume a residency slot.
3. **Prefetch set:** chunks within `view_distance_chunks + prefetch_distance_chunks`, ranked by predicted usefulness. Prefetch never displaces the active set or its apron.

Missing chunks are admitted in this order:

1. Missing **mesh-apron chunks** needed to unblock a visible/dirty active chunk.
2. Missing **active chunks**, ordered by Chebyshev distance to the focus, then movement-direction score, then `ChunkKey::raw` as a stable tie-breaker.
3. Missing **prefetch chunks**, using the same ordering but lower priority.

For each requested chunk, the Streaming Manager first checks the region-file TOC for an `EDITED` record:

* If an edited record exists, enqueue a **Load** job.
* If no edited record exists, enqueue a **Generate** job.
* Load jobs therefore consume saved state when available; Generate is the fallback for unedited procedural terrain.

A streaming request reserves the chunk's worst-case **dense core block** before the job enters a worker queue. The reservation also includes a material allocation if needed. Mesh snapshot, mesh-result, collision-source, and other transient buffers are reserved at their ownership transition and released when ownership moves or the result is discarded. A byte is never reserved twice: the memory ledger has one owner for each live allocation.

If a reservation would exceed the memory budget, the request remains queued but is not admitted. Prefetch requests are the first to yield under pressure.

Edit-dependent phases may request only the canonical owners for the current phase; a huge explosion therefore does not require the entire explosion footprint to be resident simultaneously. The phase scheduler may stream the next phase after the current phase releases its locks.

Every later allocation that is owned by the core/wrapper is charged against the same budget; an allocation that cannot be reserved causes that job to defer rather than exceed the hard cap.

#### Residency State

Residency is tracked independently from the mesh state machine in 3.1:

```
Absent
  -> Requested
  -> Loading / Generating
  -> Resident
  -> EvictPending
  -> Unloading
  -> Absent
```

* `Requested` means the chunk has a streaming demand but no worker has started it yet.
* `Loading` reads an edited chunk payload from disk and publishes its authoritative `N³` core block. Nothing outside the core is loaded from disk.
* `Generating` publishes the authoritative `N³` core block procedurally, then participates in the same dirty-propagation path as a loaded chunk.
* `Resident` means the authoritative core exists in the chunk store. A resident chunk may independently be `Clean`, `Dirty`, `Meshing`, `MeshReady`, or `Committed` in the mesh state machine.
* `EvictPending` means the chunk is no longer in the active/prefetch demand set and is a candidate for removal once persistence and in-flight references are clear.
* `Unloading` removes the chunk from the desired streaming set and marks its 26 neighbors dirty. Reference counting in 3.2 still controls the actual memory release.

#### Prefetch and Eviction Hysteresis

A chunk is not evicted merely because it crosses the active view radius. Chunks remain resident while they are inside the **prefetch radius**, which prevents immediate unload/reload thrashing as the focus moves near a chunk boundary.

Under normal operation, a chunk becomes an eviction candidate when:

* it is outside `view_distance_chunks + prefetch_distance_chunks`, and
* it has no edit, mesh, load, generate, persist, or completion work that requires residency.

Eviction candidates are ordered **farthest-first** from the current focus. Under memory pressure, the farthest chunks in the **prefetch set are allowed to evict even before they leave the prefetch radius**; this is the hysteresis exception used to recover memory without touching the active set. If additional space is required after all prefetch chunks are gone, the engine **reduces effective prefetch to zero** before considering active-set degradation.

#### Edited Chunks, Persist Failure, and the Eviction Wedge

v3's eviction policy could wedge: the budget is hard, edited chunks cannot evict until persisted, Persist was guaranteed only about once per second, and a failed save pinned memory forever. v4 makes the failure path explicit.

* **Persist priority boost.** When `resident_bytes + reserved_bytes >= 0.9 * budget`, Persist is promoted to the Mesh lane for the duration of the pressure. This is the only condition under which Persist outranks Mesh.
* **Retry with backoff.** A failed save is retried up to `kMaxPersistRetries = 5` times with exponential backoff (100 ms, 200 ms, 400 ms, 800 ms, 1600 ms). Each retry re-acquires the block reference, so it always serializes the latest state.
* **Pin and report.** After the retry budget is exhausted, the chunk is marked `persist_failed` and **pinned**: it is exempt from eviction and its bytes are charged against a dedicated `edit_pin_reserve` (default 15% of the budget). The failure is reported through telemetry and the `data_loss_warning` signal (4.2). A pinned chunk is retried on the next flush interval, so a transient I/O error clears itself.
* **Overflow policy.** If pinned edited bytes exceed `edit_pin_reserve`, the engine, in order: (a) keeps the Persist boost active, (b) reduces the effective active radius to release *unedited* chunks, (c) suppresses prefetch entirely. Only if all three are exhausted does it evict an unpersisted edited chunk, and it emits `data_loss_warning` with the chunk key and the number of unsaved samples. Silent data loss is not permitted.
* **Flush interval.** The default flush interval is 1 s, matching the Persist reservation in 3.4. The maximum data-loss window is therefore one flush interval plus the retry backoff, and it is recorded in telemetry.

#### Memory Budget

`resident_memory_budget_mb` is a hard ceiling for **engine-managed CPU memory associated with chunk streaming**, including resident core blocks and material planes, streaming reservations, worker mesh windows, dirty-set bookkeeping, queued completion buffers, and the navigation source cache. It is configurable and defaults to **512 MiB**.

The dense sample payload is:

* TSDF core: `N³ × 2 = 65,536` bytes = **64.0 KiB** per dense chunk.
* Material: `N³ = 32,768` bytes = **32.0 KiB** when allocated.
* Conservative dense-with-material planning cost: **96.0 KiB per chunk**, before mesh/collision/navigation resources.

The streaming controller therefore treats the memory limit as an admission constraint, not as an after-the-fact warning:

$$\text{resident\_bytes} + \text{reserved\_bytes} \le \text{resident\_memory\_budget\_mb} \times 2^{20}$$

`view_distance_chunks` is a **target** and is satisfied whenever the memory budget permits. When the requested radius cannot fit, the engine does not allocate past the hard limit: it first suppresses prefetch, then reduces the effective active radius to the largest affordable value. The effective radius is exposed in telemetry so the game can detect memory-constrained streaming.

For reference, a fully dense 16-chunk Chebyshev radius with its mesh apron contains `35^3 = 42,875` chunk positions and would require roughly **2.66 GiB of TSDF storage alone** before materials or derived resources. The design therefore does **not** treat `view_distance_chunks = 16` as a memory guarantee; the memory budget is authoritative.

Godot/driver allocations that are not controlled by the extension allocator (for example GPU residency) cannot be made a strict process-level hard bound by the core. Their CPU-side estimates are tracked separately in performance telemetry, while the streaming budget above is the hard bound for memory owned by this engine pipeline.

#### Interaction with Load, Generate, and the Completion Queue

* **Load vs. Generate:** Load is a dedicated worker-lane queue for edited chunks; Generate remains the fallback for unedited chunks. Both reserve memory before admission and both produce a resident authoritative chunk that marks itself and its neighbors mesh-dirty for later Mesh admission.
* **Priority:** Edit and Mesh work outrank Load; Load outranks Persist and Generate for latency-sensitive streaming. Generate is lowest priority and is the first work throttled when the active view is incomplete due to memory pressure. Edit phases may yield between phases so that this priority can take effect.
* **Neighbor readiness:** A Mesh job for an active chunk is not admitted until every chunk in its `3³` owner-chunk closure is either resident or known absent. This prevents a visible chunk from stalling on diagonal boundary samples.
* **Completion queue:** Load and Generate jobs do **not** enter the MPSC completion queue as heavy results because they initialize core state. Their successful completion marks the chunk mesh-dirty. Edit-phase control records and Mesh chunk results enter the completion queue; navigation is handled at group scope.
* **Completion backpressure:** The completion depth caps in 3.4 remain in force, and admission is additionally blocked when queued completion buffers consume the configured streaming memory budget. Backpressure therefore limits both queue length and bytes.
* **Eviction under load:** Eviction is a low-priority streaming action. It is never allowed to interrupt an Edit job, and it waits for the chunk's references to reach zero as required by 3.2.

#### Large-Edit Interaction

A large explosion may touch far more chunks than can be safely locked or remeshed in one step.

* The Streaming Manager does **not** reserve the explosion's whole affected set up front.
* The Edit lane uses the bounded phases from 3.2. Each phase acquires at most `kMaxEditPhaseChunk` locks on its owner chunks only, then releases them before any scheduler, queue, memory, or navigation wait.
* Edit phases only mark `mesh_dirty`; they do not push one Mesh completion per changed chunk. The Mesh scheduler admits dirty chunks according to the completion-depth and memory limits in 3.4.
* If the completion queue is at its hard cap when an edit phase finishes, the edit can yield before starting the next phase. No phase lock is retained during that wait.
* This explicitly couples large-edit behavior to downstream backpressure without allowing one explosion to hold a giant lock set or burst the completion queue.

#### Streaming Update Frequency

The Streaming Manager recomputes the desired set at most once per frame and only when the focus crosses a chunk boundary or the velocity direction changes enough to alter prefetch ranking. This avoids repeatedly rebuilding the same request set while the player moves within one chunk.

The stream planner itself runs on the main thread because it operates on small chunk-coordinate sets and only enqueues core jobs. Disk IO, decompression, generation, and meshing remain worker-side.

### 3.4 Worker Pool and Scheduling

#### Lanes (highest to lowest priority)

1. **Edit:** explosions and other density modifications.
2. **Mesh:** proximity-driven meshing and collision-source generation for dirty chunks.
3. **Load:** disk reads and decompression for edited chunks requested by the Streaming Manager.
4. **Persist:** serialization, compression, and disk writes.
5. **Generate:** procedural generation of new chunks.

#### Workers

`max(1, hardware_concurrency() - 2)` threads. The main thread is excluded, and one core is left for the OS and the Godot render thread.

Workers pull from lanes in priority order. Strict priority starvation is a real failure mode: sustained editing can starve Persist and Generate indefinitely. The rules below bound that while also bounding the size of any single Edit lock round.

#### Fairness Rules

| Rule | Value | Purpose |
| :--- | :--- | :--- |
| Aging | A job's effective priority increases by 1 lane every 500 ms in queue, **capped at 2 lanes of promotion** | Guarantees lower-priority work eventually runs without flattening the lane order. Generate can reach Persist priority; it can never reach Load or above. |
| Edit starvation bound | At most 16 consecutive **Edit phases/jobs** before the scheduler must yield to Mesh or Load when such work is waiting | Prevents streaming/meshing from never running |
| Persist reservation | Persist jobs are never aged past Mesh or Load priority, but at least 1 Persist job runs per second while the queue is non-empty and the worker is not inside a required phase | Prevents unbounded save starvation |
| Persist pressure boost | When `resident_bytes + reserved_bytes >= 0.9 * budget`, Persist is promoted to the Mesh lane until pressure clears | Drains edited chunks before the eviction wedge in 3.3 can form |
| Load reservation | Load jobs outrank Generate and are admitted whenever an active chunk has a saved edit and memory headroom exists | Saved edits should not wait behind procedural generation |
| Generate preemption | Generate may run only after active Load work is admitted and only when the active chunk set still has memory headroom | Protects active-view latency |
| Generate admission cap | At most `kMaxConcurrentGenerateJobs = 2` Generate jobs in flight and at most `kMaxGenerateQueueDepth = 8` queued at any time; further requests stay in the streaming request queue | Bounds how much of the world Generate can materialize at once, so a cold start cannot flood the budget |
| Edit coalescing | Intersecting explosion requests may be batched up to `max_edit_batch_spheres = 16`; batching evaluates the exact max of every sphere SDF term and **never** replaces the union with a bounding sphere | Reduces queue pressure without changing CSG semantics |
| Edit queue cap | `max_pending_edit_transactions = 64` by default | Prevents unbounded user-side edit backlog; enqueue fails cleanly when full |
| Edit phase cap | `kMaxEditPhaseChunks = 32` by default | Bounds mutation work per lock round; also the lock cap, by construction |
| Phase yield | After every Edit phase, the transaction may yield before planning/acquiring the next phase | Prevents one huge explosion from monopolizing workers while retaining bounded lock hold time |
| Lane budget | A worker checks `shutting_down` between jobs and after every Edit phase | Bounds teardown latency |

The v3 aging rule ("capped at the top lane") is corrected: uncapped aging meant that at startup, thousands of Generate jobs all became Edit priority after 1 s, contradicting "Generate is lowest." The 2-lane cap preserves the ordering.

#### Edit Phase Scheduling

An Edit transaction is a stateful job with a deterministic list of phase descriptors.

* A phase is the **only** unit allowed to acquire Edit locks.
* Before starting a phase, the scheduler checks `shutting_down`, completion-queue hard backpressure, and required memory reservations.
* Before a phase can lock anything, all canonical owner chunks needed by that phase must be **Resident**. Missing owners are requested through the streaming manager at Edit-dependency priority. The phase waits for residency with **no chunk locks held**. If the current phase cannot fit the memory budget, the planner subdivides it into smaller phases down to `kMinEditPhaseChunks`; the edit is deferred only when even the minimum viable phase cannot be admitted.
* A phase reserves **one completion-queue entry before acquiring any lock** for its mandatory `EditPhaseCompleted` control event. That reservation remains held until the phase completion record is published or the phase is cancelled. This prevents a phase from reaching completion and then being unable to report the terrain revision because other workers filled the queue meanwhile.
* Once the phase begins, it must acquire its precomputed complete lock set and finish or reach a defined cancellation safe point; it does not wait for downstream queues.
* After releasing all phase locks, the transaction updates its progress and returns to the scheduler. If backpressure is active, it remains queued without holding any chunk lock.
* Mesh jobs are not created eagerly for every changed chunk. The dirty-set scheduler below performs downstream admission.

#### Backpressure Rules

| Condition | Response |
| :--- | :--- |
| Completion queue depth >= 8 entries | Stop admitting new Mesh results; continue Edit/Load work and honor the Persist reservation |
| Completion queue depth >= 32 entries | Hard backpressure: do not start another Edit phase until the main thread drains the queue. Load may continue only when it cannot add a completion result that would exceed the hard cap |
| Completion-buffer bytes at budget limit | Stop admitting heavy Mesh/collision results; run only work whose allocations already fit the budget |
| Streaming memory budget exhausted | Stop admitting prefetch and Generate work; continue only work required to keep the active mesh apron resident or persist edited chunks |
| Pending unload of a chunk with in-flight jobs | Unload is deferred and retried; reference counting controls actual memory release |
| Dirty-set backlog large | Keep dirty keys deduplicated; Mesh admission continues in priority order as completion capacity becomes available |

#### Completion Queue Contract

The queue has two explicit limits:

* **Entry-depth limit:** 32 hard entries, with 8 as the soft stop threshold for new Mesh admissions.
* **Byte limit:** all completion-owned buffers are charged to `resident_memory_budget_mb`; a result that would cross the budget is not allocated.

To make the hard limits race-free, producers use a small **completion-admission reservation** before allocating/queuing a result:

* A worker atomically reserves one completion entry before starting a Mesh result that will be published.
* It reserves the expected completion-buffer bytes before allocating those buffers. Actual allocations must also succeed through `try_reserve(bytes)`; a failed reservation aborts or defers the result before exceeding the cap.
* An `EditPhaseCompleted` control event reserves one completion entry before its phase acquires any lock; the reservation is consumed when that control event is published and released on cancellation.
* A reservation is released when the result is committed, discarded, or otherwise removed from the queue.

These are downstream limits. **An Edit transaction cannot bypass them** because an edit phase only mutates authoritative state and marks dirty keys. Mesh work is admitted later, one chunk at a time, under the same reservations.

The completion-queue cap exists because unbounded accumulation would let a large edit produce unbounded Mesh/collision buffers before the main thread can commit them. v4 prevents that by moving queue admission out of the Edit operation itself.

#### Cancellation

Every job checks a shared `std::atomic<bool> shutting_down` at safe points: between ordinary jobs and between Edit phases. Edits already applied to authoritative state are never rolled back by cancellation.

#### Thread Start

The worker pool is created by `VoxelTerrain3D::_enter_tree()`, not in the constructor, and only when the node is not in the editor (4.2). Godot instantiates nodes freely in the editor, in the resource loader, and in the scene-instantiation path before they enter the tree. Spawning a pool in a constructor leaks a pool per preview instantiation.

### 3.5 System Implementations

#### Explosion CSG (TSDF subtraction)

An explosion payload contains center `c` and radius `R`. The core:

1. Validates `R > 0` and computes the axis-aligned sphere bounds in world sample coordinates.
2. Maps every potentially changed world sample to its **canonical owner chunk** using `floor_div(sample, N)`.
3. Computes the complete affected-owner set, then partitions it into the bounded Edit phases defined in 3.2. Each phase locks exactly its owner set.
4. For each phase, acquires the complete lock set in ascending `ChunkKey::raw` order. While the locks are held, for every canonical sample `p` owned by the phase:

$$\text{NewSDF}(p) = \max\big(\text{CurrentSDF}(p),\; R - \lVert p - c\rVert\big)$$

The result is clamped to the truncation band ±δ. Under the negative-solid convention, samples inside the sphere become air.

5. **Only canonical samples are written.** Each owner chunk's new block is produced by copying its current block and applying the writes to the copy; the copy is published atomically. There is no derived state to refresh.
6. Each changed resident owner chunk increments its `version` while its write lock is held and is marked `mesh_dirty` along with its 26 neighbors (3.2, Dirty Propagation Rule).
7. After all phase locks are released, the phase increments the global `terrain_revision` exactly once if at least one canonical sample changed. The phase completion record carries that revision, the affected navigation-group keys, and the `(chunk_key, new_version)` pairs for every chunk it published (bounded by `kMaxEditPhaseChunks`).
8. The phase then yields back to the scheduler. It does not wait for the completion queue while locks are held.
9. After the final phase, the logical explosion request is complete. `explosion_completed` is emitted only after its final edit completion record reaches the main thread, not merely when its first phase finishes.

The Edit worker **does not enqueue a Mesh job for every changed chunk**. Dirty chunks are deduplicated and consumed by the Mesh scheduler under the completion/memory caps in 3.4.

This phased design is intentionally different from v2/v3's single giant lock round: a very large explosion may touch hundreds or thousands of chunks, but no worker ever holds locks for that entire set at once, and no worker ever holds a lock on a chunk it does not write.

#### Procedural Generation (FastNoise2)

* Generation runs on the Generate lane, one chunk per job.
* **FastNoise2 produces a density field, not a true SDF.** v4 therefore treats the generated field as an **SDF approximation** obtained from a first-order local distance estimate:

$$\text{sdf}_{approx} = \frac{\text{density} - \text{iso}}{\lVert \nabla \text{density} \rVert \cdot s}$$

where `iso` is the iso-level of the noise type, `∇density` is the gradient magnitude in voxel units, and `s` is the voxel scale. This normalization reduces local distance-scale distortion near the isosurface, but it is not an exact signed-distance transform for arbitrary noise fields. The implementation must validate **surface-position error, near-surface gradient magnitude, and repeated CSG stability** before treating the approximation as sufficient. The gradient is evaluated by finite difference of the noise, so a full evaluation costs roughly 4 noise evaluations per sample.

**Homogeneity pre-pass.** Evaluating all `N³ × 4` noise samples to discover that a chunk is entirely air or entirely solid is the dominant generation cost in an air-heavy world. v4 therefore requires a cheap pre-pass before the full evaluation:

1. The generator exposes `coarse_bound(chunk) -> { DEFINITELY_AIR, DEFINITELY_SOLID, UNKNOWN }`.
2. The default implementation evaluates a 2D heightfield pass (`N²` noise evaluations, one column per `(x, z)`) plus a small vertical probe set at the chunk's corners and center. If the highest surface estimate plus a margin is below the chunk's lowest sample, the chunk is `DEFINITELY_AIR`; if the lowest surface estimate minus a margin is above the chunk's highest sample, it is `DEFINITELY_SOLID`.
3. The pre-pass must be **conservative**: it may only return `DEFINITELY_AIR` or `DEFINITELY_SOLID` when the full evaluation is guaranteed to produce a homogeneous chunk. A margin of at least `2δ` beyond the truncation band is required. `UNKNOWN` always falls through to the full evaluation.
4. A `DEFINITELY_AIR` or `DEFINITELY_SOLID` result publishes the corresponding singleton block and allocates nothing. The pre-pass is never allowed to produce a wrong homogeneous classification; a test compares it against the full evaluation over a seeded corpus (7.1).

* The generator publishes the chunk's **`N³` canonical core block**. There is no local corner window and no padded window to reconstruct.
* Output is deterministic for a given seed, noise graph, and SIMD level. Cross-machine determinism is **not** guaranteed by FastNoise2's SIMD dispatch. Saved edits are stored as full canonical chunk data, so a change in generation output does not corrupt existing saves. Re-generation of unedited chunks may differ across machines; accepted for v4, and listed as a risk in section 8.
* Generation uses the initialization write path from 3.2. It does not increment `terrain_revision`.
* Marks the chunk and its 26 neighbors `Dirty` for meshing.

#### Dirty Propagation Across Lifecycle

There is no derived sample cache to maintain. The only lifecycle obligation is dirty marking, defined by the window rule in 3.2:

| Lifecycle event | Dirty propagation |
| :--- | :--- |
| **Edit** | Every owner chunk written, plus the 26 neighbor coordinates whose mesh windows include any of those chunks' canonical samples |
| **Generate** | The new chunk, plus its 26 neighbors |
| **Load** | The loaded chunk, plus its 26 neighbors |
| **Unload** | The 26 neighbors of the removed chunk |

A chunk does not have to be fully surrounded by resident neighbors before its authoritative data exists. However, Mesh admission waits until every chunk in the required `3³` owner-chunk closure is either resident or known absent, so the snapshot is deterministic.

#### Meshing (Surface Nets for v4.0)

* **Algorithm:** Naive Surface Nets over a private `(N+3)³` mesh window.
* **Snapshot source:** the job resolves the 27 owner chunks, holds a `SampleBlock` reference on each, and copies the `−1..N+1` range of each into the window. Absent owners are filled from the absent-sample rule in 3.1. The snapshot is independent of later edits and unload.
* **Cell ownership and quad emission (normative convention).** A chunk emits geometry **only for the `N³` cells it owns** (local `0..N-1`). Consequences:
  * A chunk never emits the cell at local index `N`; that cell is the neighbor's cell 0.
  * A chunk never computes a vertex for a cell it does not own, so it never needs a sample at `−2` or `N+2`. The `(N+3)³` window is exactly sufficient: cell 0's corners are samples `0..1` and its gradients need `−1` and `+1`; cell `N-1`'s corners are `N-1..N` and its gradients need `N-1` and `N+1`. **No sample outside `−1..N+1` is ever required**, which resolves the v3 concern that 35³ was one sample short.
  * Adjacent chunks emit separate quads for adjacent cells. Those quads share an edge, and both chunks compute that edge's position from the same canonical owner blocks, so the shared vertices coincide exactly and no crack appears. Bit-identical inputs give bit-identical outputs; the implementation must not introduce per-chunk variation (for example different summation order) into the vertex computation.
* **Gradient normals:** computed by central difference of the mesh window.
* **Output:** interleaved vertex buffer (position, normal, material), `uint32_t` index buffer, per-chunk bounds, and collision triangle source data. Ownership moves to the completion queue.
* **Navigation:** Mesh does **not** produce a separate per-chunk navigation result. The committed Surface Nets mesh is reused as source geometry for region-group navigation rebuilds defined in 4.5.
* **Why not Transvoxel for v4:** Transvoxel's transition cells exist to bridge chunks of different resolution. v4 has one resolution, so those tables add cost with no benefit. LOD milestone M5 adopts Transvoxel with an octree or clipmap structure and transition masks.

#### Save and Load Pipeline

Per chunk, in order:

1. **Homogeneous form:** If every **canonical owned sample** in the `N³` core block is the clamped air marker, store `CHUNK_EMPTY`. If every canonical owned sample is the clamped solid marker, store `CHUNK_SOLID`. Neither carries a sample payload.
2. **Serialize the core block:** Otherwise the chunk's `N³` canonical `int16_t` sample array is written as two byte planes (low byte, high byte). The material plane, when present, is a third plane containing the `N³` canonical material samples.
3. **Compress:** `LZ4_compress_default` per plane. Compressed length is recorded in the TOC.
4. **No version bracket.** The Persist job serializes from an immutable block reference (3.2, Save Consistency Rule).
5. **Write:** Into a region file (see 3.6).

RLE is not in the pipeline. Only **edited** chunks are written; an unedited chunk is regenerated on load, so a save file contains the delta from generation, not the world. Chunks carry an `edited` flag, set by the first authoritative mutation and cleared only by an explicit regeneration.

Load is the inverse of this process: it reconstructs the canonical `N³` core block, publishes it under the Load initialization write path, and marks the chunk and its 26 neighbors dirty.

### 3.6 Region File Format

* Chunks are grouped into **regions of 16 × 16 chunks** in X and Z, with Y as a vertical stack bounded by `world_height_chunks`. v3 used 32 × 32; 16 × 16 is chosen because a flush rewrites the whole region, and rewrite cost is proportional to the number of *edited* chunks in the region. A smaller region bounds the amplification when a region is sparsely edited.
* Each region file starts with a fixed header and a **table of contents (TOC)**:

| Field | Notes |
| :--- | :--- |
| Magic, format version | Rejects foreign or truncated files |
| Region coordinates | `floor_div(Cx, 16)`, `floor_div(Cz, 16)` |
| **World seed** | Required. A save whose seed does not match the current world must not be loaded. |
| **Generator version** | Required. Identifies the noise graph and SIMD level that produced unedited chunks. If it does not match the current generator, unedited chunks are regenerated rather than trusted, so a noise or SIMD change cannot seam against saved edited chunks. |
| Chunk count | Number of TOC entries |
| Per-chunk entry | Byte offset, compressed length, uncompressed length, flags (`EMPTY`, `SOLID`, `RAW`, `EDITED`), save timestamp |

The v3 header lacked the seed and generator version, which allowed regenerated unedited neighbors to seam against saved edited chunks after a noise or SIMD change. Both fields are mandatory in v4.

#### Crash Safety

v2.0 described two conflicting approaches: "append and update the TOC," which requires an in-place TOC write, and "write a temp file and rename atomically," which rewrites the whole region. An in-place TOC update is not crash-safe, so the region format is defined as follows, with one consistent rule:

* **Durability unit is the region file, not the chunk.**
* A flush writes the complete region state to `region.tmp`, `fsync`s it, then `rename`s over the live file. `rename` is atomic on Windows and POSIX, so the live file is always a complete region.
* **There is no in-place TOC update.** The TOC lives in RAM and is only committed to disk on flush. Frequent small edits therefore batch, and the flush is triggered by a dirty-chunk count or a time interval, not per edit.
* A crash therefore loses at most the chunks written since the last flush. That is the accepted loss window, and it is why the Persist lane must not be starved (3.4) and why `save_on_exit` exists.

An append-only chunk journal with an atomically replaced index was considered and rejected: it moves the crash-safety problem from the region file to the journal index, and it requires a merge pass on load. Smaller regions achieve the same bound on rewrite cost with a strictly simpler format. The decision and its rationale are recorded in the decision log.

#### Compaction

* During normal operation, each flush rewrites the region file in full, so there is no append-log growth and no need for the separate compaction pass from v2.0.
* If a region exceeds the configured size limit, it is split into child regions at flush time and the parent TOC redirects. This is a maintenance pass triggered manually, not a runtime path.

---

## 4. GDExtension Integration Strategy (Godot Wrapper)

Godot 4 uses **GDExtension** rather than GDNative. The wrapper compiles into an extension that links against the core library.

### 4.1 Memory Allocation and Marshalling

Large buffers are produced by workers as plain POD and copied once, from core memory into Godot-owned memory, on the main thread during commit. The copy is bounded by the upload budget (see 4.3).

**Default commit path is `RenderingServer`, not `ArrayMesh`.** v3 made `RenderingServer` an opt-in optimization pending profiling. v4 inverts that: the default is the direct server path, because the main-thread cost of `ArrayMesh` construction is the single largest commit cost and it is not avoidable by tuning. The `ArrayMesh` path remains available behind a property for A/B comparison.

Workers build the surface as three POD arrays — interleaved vertex data (`Vector3` position, `Vector3` normal, `uint32_t` material), `uint32_t` indices, and de-indexed collision triangles — and hand them to the completion queue. The main thread wraps them:

```cpp
PackedVector3Array positions;
positions.resize(vertex_count);
memcpy(positions.ptrw(), core_mesh.positions, vertex_count * sizeof(Vector3));

PackedVector3Array normals;
normals.resize(vertex_count);
memcpy(normals.ptrw(), core_mesh.normals, vertex_count * sizeof(Vector3));

PackedInt32Array indices;
indices.resize(index_count);
memcpy(indices.ptrw(), core_mesh.indices, index_count * sizeof(int32_t));

Array arrays;
arrays.resize(RenderingServer::ARRAY_MAX);
arrays[RenderingServer::ARRAY_VERTEX]  = positions;
arrays[RenderingServer::ARRAY_NORMAL]  = normals;
arrays[RenderingServer::ARRAY_INDEX]   = indices;

RID mesh_rid = RenderingServer::get_singleton()->mesh_create();
RenderingServer::get_singleton()->mesh_add_surface_from_arrays(
    mesh_rid, RenderingServer::PRIMITIVE_TRIANGLES, arrays);
```

Notes:

* `Packed*Array` is copy-on-write and reference-counted. `ptrw()` triggers a unique copy if the array is shared, so call it only after `resize()` on an array you own.
* The exact `RenderingServer` surface API (`mesh_add_surface_from_arrays` versus the lower-level `mesh_add_surface` overload that accepts raw attribute views) is verified against the pinned godot-cpp/Godot version in the M0 spike. If a lower-level overload exists at the pinned version, it is preferred because it avoids building three intermediate packed arrays.
* The material channel is passed as a custom attribute only if the pinned version supports it; otherwise material is applied per-chunk via a shader parameter. This is decided in the M0 spike, not later.

### 4.2 Class Mapping

#### VoxelTerrain3D (inherits Node3D)

The main editor-visible node.

* **Properties:**
  * `String save_file_path`
  * `int view_distance_chunks`
  * `int prefetch_distance_chunks` (default 2)
  * `int resident_memory_budget_mb` (default 512)
  * `float upload_budget_ms` (default 2.0)
  * `int world_height_chunks`
  * `bool save_on_exit` (default `true`)
  * `int nav_radius_chunks` (default 8)
  * `int nav_rebuild_debounce_ms` (default 200)
  * `bool use_array_mesh` (default `false`; see 4.1)
* **Methods:**
  * `void trigger_explosion(Vector3 position, float radius)`: enqueues an Edit transaction and returns immediately. **Never blocks.** If the edit queue is full or the engine is not active, it emits `explosion_failed`.
  * `void save_world()`: flushes the Persist queue and blocks until it completes. Intended for editor use and shutdown.
  * `void set_stream_focus(Vector3 position, Vector3 velocity)`: sets the main-thread streaming focus supplied by the game (normally the player/camera position and velocity). The wrapper passes this to the core once per frame.
  * `void set_nav_focus(PackedVector3Array agent_positions)`: sets the agent positions that define the navigation maintenance radius (4.5). Called at most once per frame.
  * `void request_navigation_flush()`: forces any debounced navigation rebuilds to submit on the next frame. Used before a path query that must not wait out the debounce.
  * `bool core_idle() const`: true when no core jobs are queued or in flight. Says nothing about Godot-side state.
  * `int get_effective_view_distance_chunks() const`: returns the current affordable active radius after memory-budget clamping.
  * `int get_prefetch_distance_chunks() const` / `set_prefetch_distance_chunks(int)`: controls the soft prefetch radius.
  * `int get_resident_memory_budget_mb() const` / `set_resident_memory_budget_mb(int)`: controls the hard engine-managed CPU residency budget.
  * `bool fully_committed() const`: true when core jobs are idle, the MPSC completion queue is empty, and every **required** navigation group is synchronized to its desired `(terrain_revision, source_epoch)`. "Required" is defined in 4.5: a live group inside the navigation maintenance radius. This is the readiness check gameplay code must use; an empty completion queue alone does not mean navigation has finished rebuilding. It does **not** promise that the requested `view_distance_chunks` is fully resident; use the effective-view/residency telemetry for streaming readiness.
  * `bool navigation_groups_synced() const`: the group-level predicate from 4.5, restricted to required groups.
  * `int get_required_nav_group_count() const`: number of groups currently required, for telemetry.
* **Signals:**

  | Signal | Emitted when | Payload |
  | :--- | :--- | :--- |
  | `explosion_requested(position, radius)` | The job is enqueued, on the calling thread | Call's arguments |
  | `explosion_completed(position, radius, affected_chunks)` | All phases of the edit transaction have applied to authoritative state, on the main thread | World state has changed; navigation may still be catching up |
  | `explosion_failed(reason)` | The job could not be enqueued (pool shut down, or the position is outside world bounds) | Reason string |
  | `chunk_committed(chunk_key)` | The chunk's visual mesh and collision are live | Raw chunk key. **Not** gated on navigation synchronization. |
  | `navigation_group_synced(group_key)` | A required navigation group has synchronized to its desired revision/epoch | Raw group key |
  | `data_loss_warning(reason, chunk_key)` | An edited chunk could not be persisted and was evicted anyway (3.3) | Reason string, raw chunk key |

  The v2.0 signal `explosion_applied` is removed. Its meaning was stated two different ways in the doc: "after the edit is committed" in 4.2, and "on enqueue" in 5.2. A gameplay programmer must be able to distinguish **request accepted** from **world state changed**, so the two cases are now separate signals. `explosion_completed` is emitted from the `_process` drain loop, not from within `trigger_explosion`, because worker code may not call into Godot.

  `chunk_committed` is **decoupled from navigation** in v4. v3 emitted it only after the owning navigation group synchronized, which meant that while the player moved and streaming never went quiet, the signal could be delayed indefinitely and its meaning was overloaded. v4 emits it as soon as the visual mesh and collision are live, and adds a separate `navigation_group_synced` signal for the navigation half of the contract.

* **Editor and lifecycle guards.** GDExtension classes are instantiated by the editor as well as by the running game, and v3 had no guards:

  * `_enter_tree()` returns immediately when `Engine::get_singleton()->is_editor_hint()` is true. No pool is created, no streaming starts, no region files are touched. Editor previews, the resource loader, and scene instantiation all construct nodes before they enter a tree; starting a pool there leaks a pool per preview.
  * `_process()` returns immediately in the editor, so no commit work happens there either.
  * `save_on_exit` is ignored in the editor: `_exit_tree()` calls `shutdown(save_on_exit && !Engine::get_singleton()->is_editor_hint())`. An editor save is only ever performed by an explicit `save_world()` call.
  * The node is created with `set_process_mode(PROCESS_MODE_ALWAYS)`. If the tree is paused, `_process` stops draining and the whole pipeline stalls: completions back up, backpressure engages, and the world freezes. Always-process keeps the drain loop alive.
  * **Reparenting** the node out of and back into the tree runs `_exit_tree()` then `_enter_tree()`: a full shutdown, optional save, and recreate. This is correct but expensive, and it discards all resident state. Keep the terrain node in the tree for the lifetime of the scene, or use one terrain per scene.

#### VoxelAICompanion (inherits CharacterBody3D)

A GDScript-facing node that uses Godot's navigation stack.

* **Methods:**
  * `PackedVector3Array get_voxel_path(Vector3 start, Vector3 end)`: wraps `NavigationServer3D.map_get_path()` on the terrain's navigation map. It does not call into the core.
  * `void set_target_position(Vector3 position)` and `float get_distance_to_target()`: standard agent plumbing for use with `NavigationAgent3D`.
* **Note:** The name "voxel path" is kept for compatibility with v1. It is a Godot navigation query and carries no voxel-specific meaning.

### 4.3 Main-Thread Commit Loop

The explosion path does not block the main thread.

```
trigger_explosion()          [main thread] -> enqueue Edit transaction, return
  Edit phase                 [worker]      -> bounded lock round, publish COW blocks, mark dirty
  Mesh admission             [worker]      -> pull dirty chunk only when queue/memory caps permit
  Mesh job                   [worker]      -> build mesh window from owner blocks, Surface Nets,
                                              build collision source as POD
  completion queue (MPSC)    [workers -> main]
VoxelTerrain3D::_process()   [main thread]
  drain completions within soft budget
    commit visual mesh + collision
    emit chunk_committed(...)                 (navigation-independent)
    feed committed mesh into the nav source cache
  run the pure-C++ nav group state machine
    submit debounced group bakes
    poll server synchronization
    emit navigation_group_synced(...)
```

#### The Budget Is Soft

The loop attempts to respect `upload_budget_ms` but **cannot guarantee it**. The reasons:

* The budget is checked **before** a chunk starts, not mid-chunk. A single 32³ chunk commit may exceed 2 ms on low-end hardware.
* **Commits are never split across frames.** A chunk's mesh is unusable if half its vertices are uploaded; splitting would produce visible geometry corruption. So the loop either commits a chunk whole or defers it to the next frame.

The requirement is therefore stated as: **the average frame-time contribution of voxel commits stays under `upload_budget_ms`, and no frame exceeds `upload_budget_ms` by more than the cost of one chunk commit.** The exception is bounded and measurable per hardware target.

Commit cost is not dominated by vertex copying. `RenderingServer` surface creation and `ConcavePolygonShape3D` construction are main-thread costs; `NavigationServer3D` group submission is also measured separately. The actual navigation bake is asynchronous on the server side and is not treated as a synchronous mesh-commit cost.

#### Queue Backpressure

If the completion queue reaches the hard depth cap in 3.4, `_process` drains **without the soft budget check** for that frame, because an over-deep queue is the worse failure. This is the one condition where the frame budget is knowingly exceeded. Edit phases themselves never hold locks while waiting for this condition to clear.

#### Commit Throughput

v3 had no commit-throughput target, and at roughly 1–2 chunks per frame a 1000-chunk crater or a cold start takes on the order of 10–15 seconds. v4 treats commit throughput as a first-class requirement validated in the M0 spike:

* **Target:** at least **8 dense-chunk commits per frame** within a 4 ms budget on the reference hardware (7.3), with vertex upload and collision shape creation measured independently.
* **Spike first.** The M0 spike builds the raw surface path (`RenderingServer` surface creation from worker-built POD) and the collision shape path end to end, before the full contract is built. These are the two costs most likely to miss the target, and they were validated last in v3.
* If the spike misses the target, the fallback is a coarser collision mesh (4.4) and, only then, a smaller chunk size.

### 4.4 Physics Collision

* **Source:** Collision geometry is generated from the same Surface Nets output as the visual mesh. It is not a separate meshing pass.
* **Shape:** `ConcavePolygonShape3D` (equivalent to `PhysicsServer3D` `SHAPE_CONCAVE_POLYGON_SHAPE`) on a `StaticBody3D` per chunk. Chunks are static, so this is valid.
* **Triangle soup, not indexed buffers.** Godot's concave shape takes a de-indexed triangle soup: 3 floats × 3 vertices per triangle, with no shared indices. Indexed vertex data compressed 2–3× by reuse will roughly triple in memory as a collision soup. Budget for it. The worker produces the soup directly so the main thread performs no re-indexing.
* **Cost is on the main thread.** Shape creation runs during commit, so it counts against the upload budget and is measured separately in 7.3. It is not covered by the `memcpy` cost in 4.1.
* **Decimation:** Not in v4.0. It is treated as a **planned escape hatch**, instrumented from M3 onward, not merely a future optimization. If collision bake time or memory exceeds budget, the fallback is a decimated or voxel-aligned collision mesh generated as a separate simpler Surface Nets pass (for example on a 2× subdivided sample grid).
* **Lifecycle:** The collision shape is replaced atomically when the chunk is recommitted. Stale shapes for removed chunks are freed on the main thread.
* **Player and projectile safety:** Collision for a chunk is only committed after its visual mesh. A chunk that is `Committed` but not yet collidable is not a problem for characters, because spawning logic uses `fully_committed()` (4.2), not `core_idle()`.

### 4.5 Navigation

v1 proposed a standalone `dtTileCache`. That approach was removed because:

1. It duplicates the memory and update cost of Godot's built-in navigation, which is also Recast-based.
2. `dtTileCache` rasterizes 2.5D heightfields. Overhangs and caves need a different rasterization setup.
3. Godot's `NavigationAgent3D`, RVO avoidance, and the navigation debug view all depend on `NavigationServer3D`.

#### Navigation Group Ownership

* Navigation regions are **groups of chunks**, not single chunks.
* The default group size is **4 × 4 chunks in X/Z**. A group therefore covers a **4 × 4 chunk** horizontal footprint and spans all `Y` chunk coordinates in `[0, world_height_chunks)`. (v3 stated "a 16×16-chunk footprint" here, which was a typo for the 4×4-chunk group size.)
* The group key is `NavigationGroupKey = (floor_div(Cx, 4), floor_div(Cz, 4))`.
* Each live group owns exactly one `NavigationRegion3D` and one navigation mesh resource on the terrain's map.
* A group becomes live when any chunk in its footprint needs navigation and is destroyed only when all of its chunks and all dependent border-padding source chunks are no longer resident.

#### Navigation Maintenance Radius

v3 coupled navigation to chunk commit and to `fully_committed()` with no spatial bound. Because every Load/Generate mesh commit bumped a group's `source_epoch`, frontier groups chased their desired state indefinitely while the player moved, and `fully_committed()` could never become true. v4 bounds the problem:

* The wrapper supplies agent positions once per frame via `set_nav_focus()`.
* A group is **required** when its footprint intersects the `nav_radius_chunks` Chebyshev shell around any agent position. Groups outside that shell are **not** required: they are not baked, and they are destroyed when they fall out of the shell.
* `fully_committed()` and `navigation_groups_synced()` range over **required groups only**. A group that is live but outside the radius is ignored by the predicate.
* **Rebuilds are debounced.** A group does not submit a bake the moment it becomes dirty. It waits `nav_rebuild_debounce_ms` (default 200 ms) of quiet, and never longer than `nav_rebuild_max_delay_ms` (default 1000 ms) after it first became dirty. `request_navigation_flush()` submits immediately on the next frame. This is what makes the system quiescent: while the player moves steadily, a group settles instead of chasing every commit.
* **Committed mesh buffers are retained for the bake.** A committed mesh result's POD arrays are copied into a `NavSourceCache` when the chunk belongs to a required group, and released after the group's bake synchronizes. The cache is charged to `resident_memory_budget_mb`. Without this, the bake would have to read triangles back from `RenderingServer`, which is slow and outside the memory ledger.

#### Terrain Revision and Per-Group State

The terms used by `fully_committed()` are normative and have concrete definitions.

**`terrain_revision`** is a Core-owned `uint64_t` counter:

* It starts at `0` at engine creation.
* It increments by exactly **one after each Edit phase that changed at least one authoritative canonical sample**.
* The increment occurs after that phase releases all chunk write locks.
* Load, Generate initialization, Mesh, Persist, dirty propagation, and Unload do **not** increment it.
* The value therefore identifies the order of committed terrain-edit phases. It is monotonic and never reused.

Each navigation group has this explicit state, stored and mutated on the Godot main thread:

```cpp
struct NavigationGroupState {
    bool live = false;
    bool required = false;      // inside the nav maintenance radius

    // Latest terrain content revision that this group must reflect.
    uint64_t desired_terrain_revision = 0;

    // Latest terrain revision represented by the source submitted to Godot.
    uint64_t submitted_terrain_revision = 0;

    // Latest terrain revision known to be reflected after server synchronization.
    uint64_t synced_terrain_revision = 0;

    // Changes caused by load/generate/unload membership changes.
    uint64_t desired_source_epoch = 0;
    uint64_t submitted_source_epoch = 0;
    uint64_t synced_source_epoch = 0;

    // Per-source-chunk committed mesh version watermark.
    // required_mesh_version[c] is the version of c's mesh that this group's
    // next bake must contain. A group is bake-ready only when every source
    // chunk c is resident and committed_version[c] >= required_mesh_version[c].
    std::unordered_map<uint64_t, uint64_t> required_mesh_version;

    bool rebuild_in_flight = false;
    uint64_t submitted_map_iteration = 0;
    uint64_t dirty_since_us = 0;
};
```

`desired_source_epoch` is a per-group monotonic counter. It increments whenever the group's **committed navigation source membership or geometry** changes for a reason **other than** an authoritative edit revision, including the successful Mesh commit of a chunk materialized by Load/Generate and the removal of a source chunk during Unload. This prevents residency/source changes from being hidden behind an unchanged `terrain_revision`.

When a group first becomes live, its `desired_source_epoch` is advanced from `0` to `1`; `synced_source_epoch` remains `0` until the first group bake is submitted and synchronized. A live group is therefore never treated as synced merely because all revision counters happen to start at zero.

**Revision max-merge.** Concurrent edit phases can publish their completion records out of order, so the wrapper must never move a desired revision backwards:

```cpp
state.desired_terrain_revision = std::max(state.desired_terrain_revision, record.terrain_revision);
```

The same max-merge applies to `required_mesh_version[c]` and to `desired_source_epoch`.

**The `(chunk, version)` watermark.** v3 had no way to map a `terrain_revision` to the mesh versions a group's source must contain, so "group is ready" was undecidable. v4 closes this:

* Every `EditPhaseCompleted` record carries a `(chunk_key, new_version)` pair for each chunk the phase published. The list is bounded by `kMaxEditPhaseChunks` (32), so the record stays small.
* On receipt, the wrapper max-merges each pair into every required group whose source set contains that chunk.
* A group is **bake-ready** when, for every chunk `c` in its owned footprint plus its border-padding source set, `c` is resident and `committed_mesh_version[c] >= required_mesh_version[c]`.
* `committed_mesh_version[c]` is updated by the wrapper when a Mesh result for `c` is committed on the main thread.

An internal `EditPhaseCompleted` record sent through the MPSC completion queue contains:

* the edit transaction ID,
* the phase ID and final-phase flag,
* the newly assigned `terrain_revision`,
* the affected navigation-group keys,
* the changed-chunk count,
* the `(chunk_key, new_version)` pairs.

The heavy Mesh result does not carry a duplicated navigation mesh. This control record is small and is still charged as a completion entry.

#### Navigation Group Dependency Closure

A group depends on more than the chunks physically inside its 4×4 footprint because its outer border is padded.

* A group's **owned source** is the current committed Surface Nets geometry for resident chunks inside the group's footprint.
* Its **border source** includes the world-space padding band at the outer edge of that footprint. The padding width is `agent_radius + connection_radius`.
* The border source is taken from resident chunks whose geometry intersects that band, including chunks in adjacent groups.
* Therefore an edit can invalidate **multiple navigation groups** even when the edited chunk belongs to only one group.
* The Core reports the full affected-group set in each `EditPhaseCompleted` record. The wrapper never guesses the set from the edited chunk key alone.

#### Border Seams (erosion)

v2.1 correctly identified border erosion as a correctness issue but did not define how the padded source was tracked. v4 makes it explicit:

* Regions are baked from the **group source snapshot**, not an individual chunk mesh.
* At the outer border of a group, source geometry is padded by `agent_radius + connection_radius` before rasterization, so erosion is spent on padding rather than usable area.
* Adjacent groups use the **same world-space padding width and shared source samples**, so the same border geometry is considered on both sides.
* Padding geometry is input to the bake only; it does not become owned playable area outside the group's actual region bounds.
* The render mesh and physics mesh are unaffected by navigation padding.

#### Update Lifecycle

For every group that becomes dirty:

1. The wrapper records the latest `desired_terrain_revision` and/or `desired_source_epoch`, and max-merges the `(chunk, version)` watermark.
2. The group waits until the debounce has elapsed (or a flush was requested) **and** it is bake-ready under the watermark rule above.
3. The main thread submits one group-level navigation update to `NavigationServer3D`. It records `submitted_terrain_revision`, `submitted_source_epoch`, and the map iteration observed at submission.
4. The server performs the navigation rebuild asynchronously.
5. The wrapper polls the pinned Godot navigation synchronization state. Once the submitted update is synchronized, it copies the submitted revision/epoch into `synced_*`, clears `rebuild_in_flight`, and releases that group's `NavSourceCache` entries.
6. If the group was invalidated again during the rebuild, `desired_*` is now newer than `synced_*`; the group is immediately queued for another rebuild from the latest committed source.

A new rebuild **does not overwrite** an in-flight result. Results are versioned by the `(terrain_revision, source_epoch)` pair they represent.

#### The Nav Group State Machine Is Pure C++

v3 defined `NavigationGroupState` inside the Godot wrapper, which made section 7.1's claim that it could be tested "core, no Godot" false. v4 factors the whole machine into the core behind an abstract synchronization interface:

```cpp
// core/nav/nav_sync.h  -- no Godot types
class INavServerSync {
public:
    virtual ~INavServerSync() = default;
    virtual void submit_group_bake(const NavigationGroupKey& p_key,
                                   uint64_t p_terrain_revision,
                                   uint64_t p_source_epoch,
                                   uint64_t p_map_iteration) = 0;
    virtual bool is_submission_synced(const NavigationGroupKey& p_key,
                                      uint64_t p_map_iteration) = 0;
};

// core/nav/nav_group_controller.h  -- no Godot types
class NavGroupController {
public:
    void set_sync_interface(INavServerSync* p_sync);   // owned by the wrapper

    void note_edit_phase(uint64_t p_terrain_revision,
                         std::span<const std::pair<uint64_t, uint64_t>> p_chunk_versions,
                         std::span<const NavigationGroupKey> p_affected_groups);

    void note_source_epoch(const NavigationGroupKey& p_key);
    void note_chunk_mesh_committed(uint64_t p_chunk_key, uint64_t p_version);
    void note_group_required(const NavigationGroupKey& p_key, bool p_required);

    // Drives submissions and synchronization. Returns the groups that
    // synchronized during this call.
    std::vector<NavigationGroupKey> tick(uint64_t p_now_us);

    bool is_synced(const NavigationGroupKey& p_key) const;
    bool all_required_synced() const;
};
```

The Godot wrapper implements `INavServerSync` by calling `NavigationServer3D`. Every rule in this section — max-merge, watermark readiness, debounce, stale-rebuild rejection, sync predicate — is implemented in `NavGroupController` and is unit-tested without Godot (7.1). The wrapper's only Godot-specific work is the `INavServerSync` adapter and the `NavigationRegion3D` resource lifecycle.

#### `navigation_groups_synced()`

`navigation_groups_synced()` is a concrete wrapper-internal predicate over **required** groups:

```cpp
bool navigation_groups_synced() const {
    for (const auto &[key, state] : nav_group_controller.groups()) {
        if (!state.live || !state.required) {
            continue;
        }
        if (state.rebuild_in_flight) {
            return false;
        }
        if (state.submitted_terrain_revision != state.desired_terrain_revision
            || state.synced_terrain_revision != state.desired_terrain_revision) {
            return false;
        }
        if (state.submitted_source_epoch != state.desired_source_epoch
            || state.synced_source_epoch != state.desired_source_epoch) {
            return false;
        }
        if (!nav_group_controller.is_synced(key)) {
            return false;
        }
    }
    return true;
}
```

The wrapper also requires the corresponding `NavigationServer3D` map synchronization/iteration condition for each submission before advancing `synced_*`. The exact Godot API sequence is pinned and verified at M0; the group state above is the engine-level contract independent of the exact Godot call sequence.

#### Path Safety and Signals

* A path query that crosses a recently invalidated group is considered safe only when that group satisfies `navigation_groups_synced()`.
* `fully_committed()` is true only when `core_idle()`, the MPSC completion queue is empty, and every **required** navigation group is synchronized to its desired `(terrain_revision, source_epoch)`.
* `chunk_committed(chunk_key)` is emitted as soon as the chunk's visual mesh and collision are live. It is **not** gated on navigation.
* `navigation_group_synced(group_key)` is emitted when a required group reaches `synced_* == desired_*` and the server confirms the submission.
* `explosion_completed` means the authoritative edit transaction is finished. It does **not** by itself mean navigation is synchronized; callers requiring path safety use `fully_committed()` or group-specific sync.

#### Navmesh Source

* Navigation source geometry comes from the same committed Surface Nets chunk meshes used for rendering/collision validation.
* A navigation group assembles a source snapshot from its owned chunks plus the required border-padding chunks. Source triangles are deduplicated by chunk ownership; the same triangle is never submitted twice merely because it lies in padding overlap.
* The group bake is a Godot `NavigationServer3D` operation and may execute asynchronously. The wrapper is responsible only for main-thread submission, lifecycle, and synchronization tracking.
* M4 profiles nav bake time and memory independently. If navigation cost dominates, a coarser source mesh may become a deliberate later design change; until then, render and navigation geometry remain matched.

#### Version Compatibility

The project initially targets **Godot 4.2**. The exact `NavigationServer3D` and navigation-baking APIs used by this design are verified against the pinned godot-cpp/Godot version during M0. If a required API is unavailable, either raise `compatibility_minimum` or adjust the implementation before M0 closes; the version is not treated as final until that verification passes.

---

## 5. Technical Implementation Details

### 5.1 GDExtension Entry Point (register_types.cpp)

```cpp
#include "register_types.h"
#include "voxel_terrain_3d.h"
#include "voxel_ai_companion.h"

#include <gdextension_interface.h>
#include <godot_cpp/core/class_db.hpp>
#include <godot_cpp/core/defs.hpp>
#include <godot_cpp/godot.hpp>

using namespace godot;

void initialize_voxel_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }
    ClassDB::register_class<VoxelTerrain3D>();
    ClassDB::register_class<VoxelAICompanion>();
}

void uninitialize_voxel_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }
}

extern "C" {
GDExtensionBool GDE_EXPORT voxel_engine_init(GDExtensionInterfaceGetProcAddress p_get_proc_address,
                                             GDExtensionClassLibraryPtr p_library,
                                             GDExtensionInitialization *r_initialization) {
    godot::GDExtensionBinding::InitObject init_obj(p_get_proc_address, p_library, r_initialization);
    init_obj.register_initializer(initialize_voxel_module);
    init_obj.register_uninitializer(uninitialize_voxel_module);
    init_obj.set_minimum_library_initialization_level(MODULE_INITIALIZATION_LEVEL_SCENE);
    return init_obj.init();
}
}
```

### 5.2 Explosion Entry Point (voxel_terrain_3d.cpp)

The entry point is non-blocking. Work happens on the core worker pool. Authoritative changes are phased; mesh/collision commits happen on the main thread; navigation is submitted at region-group scope.

```cpp
#include "voxel_terrain_3d.h"
#include "core_voxel_engine.h"

void VoxelTerrain3D::_bind_methods() {
    ClassDB::bind_method(D_METHOD("trigger_explosion", "position", "radius"), &VoxelTerrain3D::trigger_explosion);
    ClassDB::bind_method(D_METHOD("save_world"), &VoxelTerrain3D::save_world);
    ClassDB::bind_method(D_METHOD("set_stream_focus", "position", "velocity"), &VoxelTerrain3D::set_stream_focus);
    ClassDB::bind_method(D_METHOD("set_nav_focus", "agent_positions"), &VoxelTerrain3D::set_nav_focus);
    ClassDB::bind_method(D_METHOD("request_navigation_flush"), &VoxelTerrain3D::request_navigation_flush);
    ClassDB::bind_method(D_METHOD("core_idle"), &VoxelTerrain3D::core_idle);
    ClassDB::bind_method(D_METHOD("get_effective_view_distance_chunks"), &VoxelTerrain3D::get_effective_view_distance_chunks);
    ClassDB::bind_method(D_METHOD("get_prefetch_distance_chunks"), &VoxelTerrain3D::get_prefetch_distance_chunks);
    ClassDB::bind_method(D_METHOD("set_prefetch_distance_chunks", "distance"), &VoxelTerrain3D::set_prefetch_distance_chunks);
    ClassDB::bind_method(D_METHOD("get_resident_memory_budget_mb"), &VoxelTerrain3D::get_resident_memory_budget_mb);
    ClassDB::bind_method(D_METHOD("set_resident_memory_budget_mb", "mb"), &VoxelTerrain3D::set_resident_memory_budget_mb);
    ClassDB::bind_method(D_METHOD("fully_committed"), &VoxelTerrain3D::fully_committed);
    ClassDB::bind_method(D_METHOD("navigation_groups_synced"), &VoxelTerrain3D::navigation_groups_synced);
    ClassDB::bind_method(D_METHOD("get_upload_budget_ms"), &VoxelTerrain3D::get_upload_budget_ms);
    ClassDB::bind_method(D_METHOD("set_upload_budget_ms", "ms"), &VoxelTerrain3D::set_upload_budget_ms);

    ADD_PROPERTY(PropertyInfo(Variant::INT, "prefetch_distance_chunks", PROPERTY_HINT_RANGE, "0,8,1"),
                 "set_prefetch_distance_chunks", "get_prefetch_distance_chunks");
    ADD_PROPERTY(PropertyInfo(Variant::INT, "resident_memory_budget_mb", PROPERTY_HINT_RANGE, "128,4096,64"),
                 "set_resident_memory_budget_mb", "get_resident_memory_budget_mb");
    ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "upload_budget_ms", PROPERTY_HINT_RANGE, "0.5,16.0,0.5"),
                 "set_upload_budget_ms", "get_upload_budget_ms");
    ADD_PROPERTY(PropertyInfo(Variant::INT, "nav_radius_chunks", PROPERTY_HINT_RANGE, "1,32,1"),
                 "set_nav_radius_chunks", "get_nav_radius_chunks");
    ADD_PROPERTY(PropertyInfo(Variant::INT, "nav_rebuild_debounce_ms", PROPERTY_HINT_RANGE, "0,1000,50"),
                 "set_nav_rebuild_debounce_ms", "get_nav_rebuild_debounce_ms");

    ADD_SIGNAL(MethodInfo("explosion_requested",
        PropertyInfo(Variant::VECTOR3, "position"),
        PropertyInfo(Variant::FLOAT, "radius")));
    ADD_SIGNAL(MethodInfo("explosion_completed",
        PropertyInfo(Variant::VECTOR3, "position"),
        PropertyInfo(Variant::FLOAT, "radius"),
        PropertyInfo(Variant::INT, "affected_chunks")));
    ADD_SIGNAL(MethodInfo("explosion_failed",
        PropertyInfo(Variant::STRING, "reason")));
    ADD_SIGNAL(MethodInfo("chunk_committed",
        PropertyInfo(Variant::INT, "chunk_key")));
    ADD_SIGNAL(MethodInfo("navigation_group_synced",
        PropertyInfo(Variant::INT, "group_key")));
    ADD_SIGNAL(MethodInfo("data_loss_warning",
        PropertyInfo(Variant::STRING, "reason"),
        PropertyInfo(Variant::INT, "chunk_key")));
}

VoxelTerrain3D::VoxelTerrain3D() {
    // Constructor does NOT create the core or start threads.
    core_engine = nullptr;
    set_process_mode(PROCESS_MODE_ALWAYS);
}

VoxelTerrain3D::~VoxelTerrain3D() = default;

void VoxelTerrain3D::_enter_tree() {
    // GDExtension classes are instantiated by the editor too: previews, the
    // resource loader, and scene instantiation all construct nodes before they
    // enter a scene tree. Never start the pool or stream from the editor.
    if (Engine::get_singleton()->is_editor_hint()) {
        return;
    }

    core_engine = new CoreVoxelEngine();
    core_engine->start();
    core_engine->configure_streaming(view_distance_chunks, prefetch_distance_chunks,
                                     resident_memory_budget_mb, world_height_chunks);
    core_engine->set_upload_budget_ms(upload_budget_ms);
}

void VoxelTerrain3D::_exit_tree() {
    if (core_engine) {
        // Never save from the editor: an editor exit must not write region files.
        const bool should_save = save_on_exit && !Engine::get_singleton()->is_editor_hint();
        core_engine->shutdown(should_save);
        delete core_engine;
        core_engine = nullptr;
    }
}

void VoxelTerrain3D::_process(double delta) {
    if (!core_engine || Engine::get_singleton()->is_editor_hint()) {
        return;
    }

    const uint64_t frame_start_us = Time::get_singleton()->get_ticks_usec();
    const uint64_t budget_us = static_cast<uint64_t>(upload_budget_ms * 1000.0);

    // Streaming planning is main-thread only.
    core_engine->update_stream_focus(stream_focus_position, stream_focus_velocity);

    const bool hard_backpressure = core_engine->completion_queue_depth() >= 32;

    while (true) {
        std::optional<CompletionRecord> done = core_engine->try_pop_completion();
        if (!done) {
            break;
        }

        switch (done->kind) {
            case CompletionKind::ChunkMesh:
                commit_visual_and_collision(done->chunk_mesh);
                // chunk_committed is navigation-independent (4.2).
                emit_signal("chunk_committed", static_cast<int64_t>(done->chunk_mesh.key.raw));
                nav_group_controller.note_chunk_mesh_committed(
                    done->chunk_mesh.key.raw, done->chunk_mesh.version);
                nav_source_cache.retain(done->chunk_mesh);
                break;

            case CompletionKind::EditPhase:
                nav_group_controller.note_edit_phase(
                    done->edit_phase.terrain_revision,
                    done->edit_phase.chunk_versions,
                    done->edit_phase.affected_groups);
                if (done->edit_phase.final_phase) {
                    emit_signal("explosion_completed",
                                done->edit_phase.position,
                                done->edit_phase.radius,
                                static_cast<int>(done->edit_phase.affected_chunks));
                }
                break;
        }

        if (!hard_backpressure
            && Time::get_singleton()->get_ticks_usec() - frame_start_us >= budget_us
            && core_engine->completion_queue_depth() > 0) {
            break;
        }
    }

    // Group-level navigation submissions happen only on the main thread and
    // are driven by the pure-C++ state machine.
    for (const NavigationGroupKey key : nav_group_controller.tick(
             Time::get_singleton()->get_ticks_usec())) {
        emit_signal("navigation_group_synced", static_cast<int64_t>(key.raw));
    }
}
```

Notes:

* `trigger_explosion()` is main-thread only. It never dereferences a null `core_engine`, and it rejects non-positive radii before enqueue.
* `EditPhase` completion records are deliberately small. Heavy mesh/collision buffers remain in `ChunkMeshCompletion` entries.
* `commit_visual_and_collision()` never performs a navigation bake. Navigation updates are submitted once per affected region group after the required chunk meshes are current.
* `explosion_completed` means the authoritative edit transaction has finished all phases. It does **not** imply navigation synchronization.
* `chunk_committed` is emitted as soon as the visual/collision commit is live. Navigation synchronization is reported separately by `navigation_group_synced`.
* The main-thread upload budget remains soft. When the queue is at the hard cap, `_process` drains without the soft budget check to restore downstream capacity.
* `set_process_mode(PROCESS_MODE_ALWAYS)` is set in the constructor so the drain loop keeps running while the tree is paused.

### 5.3 Teardown Protocol

`CoreVoxelEngine::shutdown(bool save)` runs exactly one ordered sequence. v2.0 listed a Persist flush **after** the worker join, which is impossible: once workers are joined, no thread remains to process the Persist queue. Each step below names the thread that performs it.

| # | Step | Thread |
| :-- | :--- | :--- |
| 1 | Set `shutting_down = true`. Stop accepting new jobs. | Caller |
| 2 | Signal all workers. Each finishes its current job/phase at the next cancellation safe point, then returns to idle. | Workers |
| 3 | **Drain the Persist queue**: the caller thread runs remaining Persist jobs inline. | **Caller** |
| 4 | Discard, in order: completion queue entries, completed mesh results, pending collision results, pending navigation submissions. | Caller |
| 5 | Join all worker threads. | Caller |
| 6 | Release the chunk store, the MPSC queue, and the pools. No chunk is freed while a job holds a reference (3.2, Unload Rule). `SampleBlock` refcounts are released last, so no block outlives the store that owns its ledger. | Caller |
| 7 | Flush and close region files. | Caller |

Clarifications:

* **Who flushes Persist: the caller, in step 3.** Workers are still alive at that point, so the drain happens before step 5's join.
* `save == false` (or `save_on_exit == false`, or the editor guard) replaces step 3 with discarding the Persist queue. Unsaved edited chunks are then lost and regenerated on next load. This is the explicit data-loss window from 3.6.
* Pending **completion queue entries** (step 4) are chunks whose worker work is finished but whose Godot-side commit never happened. They are dropped. Their sources remain in the chunk store, so the state is not lost, but their meshes are not uploaded.
* Pending **navigation submissions** are dropped. Because navigation is derived state owned by `NavigationServer3D`, dropping them is safe; the map rebuilds from whatever regions exist.
* No step may run concurrently with another. The sequence is a single-threaded critical section. An Edit worker never retains a phase lock while teardown waits on Persist or queue state, and `VoxelTerrain3D::_exit_tree` must not call into the core after step 7.

---

## 6. Build and Deployment

### 6.1 Build Workflow and Core Linkage

#### Decision: static link, no separate `libvoxel_core`

v2.0 shipped the core as a separate shared library next to the extension. That is the wrong default for a GDExtension:

* Passing `std::optional`, `std::unique_ptr`, `std::vector`, or any std type across a shared-library boundary is a C++ ABI hazard. The extension and the core must be built with identical toolchain, flags, and stdlib version, or the ABI mismatch causes crashes that are near-impossible to diagnose.
* A separate DLL also has a loader-order problem: the extension's DLL directory is not on the OS DLL search path by default, so `libvoxel_core.dll` can fail to load.

**v2.1 and later link the core statically into the extension binary.** One `libvoxel_engine.*.dll` per platform, no side-by-side dependencies. This also removes the deployment burden entirely.

* CMake builds the core as a static library (`voxel_core`).
* The extension links `voxel_core` statically.
* If a separate DLL is ever required (for example a second host application sharing one core), the core must then expose a **C API only**, with opaque handles instead of `unique_ptr`, and no std types in signatures. That is a deliberate future change, not the default.

If a DLL is kept despite this, the `.gdextension` needs a `[dependencies]` section:

```ini
[dependencies]
windows.release.x86_64 = { "res://bin/libvoxel_core.windows.release.x86_64.dll" }
```

With static linkage this section is absent.

#### Build Steps

1. Build the core target `voxel_core` (static) with **CMake**. It does not depend on Godot.
2. Build the GDExtension wrapper against **godot-cpp** for the target Godot version, linking `voxel_core` statically.
3. Produce separate **debug** and **release** binaries per platform. Debug builds enable assertions, chunk-state validation, and the TSan/ASan builds used in CI.

#### Toolchain Specification

These are pinned once and recorded in `AGENTS.md` so builds are reproducible:

| Item | Requirement |
| :--- | :--- |
| Godot version | 4.2 or later, matching the godot-cpp branch |
| godot-cpp | Exact commit or tag recorded in the submodule |
| Compiler | MSVC v143 or later (Windows), Clang 15+ (Linux/macOS) |
| C++ standard | C++17 minimum; C++20 if `std::atomic`'s wait/notify is used |
| Runtime (Windows) | `/MD` (release) or `/MDd` (debug) consistently; never mix `/MT` and `/MD` between the core and the extension |
| FastNoise2 | Version pinned; note the SIMD-level behavior in tests |
| LZ4 | Version pinned |

### 6.2 Extension Descriptor (`.gdextension`)

Place the descriptor in the Godot project at `bin/voxel_engine.gdextension`.

```ini
[configuration]
entry_symbol = "voxel_engine_init"
compatibility_minimum = "4.2"

[libraries]
windows.debug.x86_64   = "res://bin/libvoxel_engine.windows.debug.x86_64.dll"
windows.release.x86_64 = "res://bin/libvoxel_engine.windows.release.x86_64.dll"
linux.debug.x86_64     = "res://bin/libvoxel_engine.linux.debug.x86_64.so"
linux.release.x86_64   = "res://bin/libvoxel_engine.linux.release.x86_64.so"
macos.debug            = "res://bin/libvoxel_engine.macos.debug.dylib"
macos.release          = "res://bin/libvoxel_engine.macos.release.dylib"
```

Notes:

* `compatibility_minimum` is **4.2**, not 4.1. The async navigation APIs relied on in 4.5 need verification at M0; if they are unavailable at the pinned version, raise this value or drop the async path.
* `macos.debug` and `macos.release` are universal entries covering x86_64 and arm64 if the dylib is built universal. Use per-arch keys (`macos.release.arm64`) only when shipping separate binaries.
* No `[dependencies]` section, because the core is statically linked (6.1).

### 6.3 Godot Project Setup (Greenfield)

The Godot project does not exist yet. Create it with the settings below, then record the actual values once the project is created.

| Setting | Initial value | Reason |
| :--- | :--- | :--- |
| Godot version | 4.2 stable, matching the godot-cpp branch in 6.1 | ABI compatibility; also the minimum for the navigation APIs |
| `rendering/renderer/rendering_method` | Forward+ (or Mobile for mobile targets) | Compatibility renderer is inadequate for large dynamic mesh counts |
| `physics/3d/default_gravity` | Project-defined | Set from game design |
| `display/window/size/*` | Project-defined | Not a voxel-engine concern |
| `application/run/main_scene` | A test scene containing a `VoxelTerrain3D` node | Required for `project_run(mode="main")` |

Setup steps:

1. Create the Godot project in the workspace directory (or the chosen location).
2. Add the godot-cpp submodule and the `voxel_core` CMake target.
3. Place `bin/voxel_engine.gdextension` (see 6.2).
4. Create the test scene and set it as the main scene.
5. Verify the extension loads and `VoxelTerrain3D` and `VoxelAICompanion` appear in the Create Node dialog.

---

## 7. Verification Strategy

### 7.1 Unit Tests (core, no Godot)

Concurrency and ownership (3.2):

* ChunkKey round-trip for coordinates from -1,048,576 to 1,048,575 on all three axes, including negative quadrants.
* **Canonical sample ownership:** every integer world sample maps to exactly one owner via `floor_div(sample, N)`, including all positive/negative chunk boundaries.
* **Shared-boundary edits:** editing a sample on a chunk boundary mutates exactly its canonical owner and produces identical reconstructed values in every dependent chunk's mesh window.
* **Single writer:** two Edit jobs overlapping on a chunk are serialized; TSan reports no data race.
* **Canonical lock order:** two concurrent edits touching overlapping phase lock sets complete without deadlock. Run with randomized lock-acquisition delays to force contention.
* **Lock-set invariant:** no Edit phase ever holds a lock on a chunk it does not write, and no phase holds more than `kMaxEditPhaseChunks` locks; a large synthetic explosion is partitioned into multiple phases.
* **Phase backpressure:** force the completion queue to its hard cap between Edit phases and verify all phase locks are released before the edit waits.
* **Dirty-set decoupling:** one large explosion marks many chunks dirty but cannot enqueue more than the permitted Mesh/completion capacity.
* **Dirty window rule:** a value-only change to a canonical sample on a chunk boundary dirties the neighbor chunk whose mesh window contains it, and the remeshed neighbor shows no crack against the freshly remeshed owner.
* **Snapshot isolation:** a Mesh job builds a consistent `(N+3)³` window while an Edit job concurrently publishes new blocks for contributing chunks.
* **Snapshot version set:** a Mesh result is rejected when any contributing chunk version changed after snapshot.
* **COW immutability:** a block reference held across an Edit publish observes no change; the old block's refcount reaches zero only after the last reader releases it (ASan).
* **Load initialization:** Load may publish authoritative state only before `Resident`; once resident, only Edit may publish.
* **Save consistency:** a Persist job serializing a chunk while an Edit publishes a new block writes a complete, self-consistent chunk state; no version bracket, no discard-and-retry.
* **Unload under reference:** unloading a chunk with one in-flight job does not free its memory until the job completes (ASan).
* **VoxelTerrain3D pool start:** constructing the node does not create threads; threads appear only after `_enter_tree()` outside the editor.

Streaming and residency (3.3, 3.4):

* **Desired-set bound:** a Chebyshev radius `r` requests no more than `(2r + 1)^3` active chunk coordinates, and no more than `(2r + 3)^3` including the mesh apron, before world bounds are applied.
* **Mesh apron:** an active/dirty chunk's mesh admission waits for every chunk in its `3³` owner-chunk closure to be resident or known absent.
* **Prefetch distance:** no chunk outside `view_distance_chunks + prefetch_distance_chunks` is requested unless explicitly retained by an in-flight dependency.
* **Load-before-generate:** an edited chunk with a region-file record is scheduled on Load and is not regenerated. An unedited chunk schedules Generate.
* **Completion isolation:** Load and Generate never enter the MPSC completion queue as heavy results; Edit-phase control records and Mesh chunk results obey the same depth/byte caps.
* **Memory hard cap:** aggregate resident + reserved engine-managed CPU bytes never exceed `resident_memory_budget_mb`, including mesh windows and completion buffers.
* **Budget degradation:** when the budget is exhausted, prefetch is suppressed/evicted before active chunks are dropped; effective view radius is reduced only after prefetch reaches zero.
* **Transient allocation refusal:** a mesh/collision result cannot allocate past the budget; the job defers until memory is available.
* **Eviction ordering:** chunks outside the prefetch radius are evicted farthest-first, edited chunks persist before eviction, and in-flight references prevent destruction.
* **Persist failure policy:** a failing save retries with backoff, pins the chunk, reports `data_loss_warning`, and is retried on the next flush; the pin is released once the save succeeds.
* **Persist pressure boost:** at 90% budget utilization, Persist outrains Mesh until pressure clears.
* **Generate admission cap:** no more than `kMaxConcurrentGenerateJobs` Generate jobs run concurrently and no more than `kMaxGenerateQueueDepth` are queued, regardless of streaming demand.
* **Aging cap:** a Generate job's effective priority never exceeds the Persist lane, however long it waits.
* **Edit queue cap:** more than `max_pending_edit_transactions` requests are rejected/coalesced according to the defined policy rather than accumulating unbounded memory.

Geometry (3.1, 3.5):

* **Cell/sample boundary ownership:** a 2×1×1 chunk pair reconstructs the same shared boundary samples from canonical owners with no duplicate authoritative writes.
* **Dirty window propagation:** a corner-boundary sample update marks dirty all chunks whose mesh windows contain it, including edge and corner neighbors.
* TSDF quantize/dequantize error bounded by `δ / 32767` across the band; `INT16_MIN` and `INT16_MAX` round-trip exactly and are detectable as markers.
* **Density-to-SDF approximation:** on representative generated terrain, measure surface-position error against a reference signed-distance estimate, near-surface gradient magnitude, and cumulative error after repeated explosion CSG operations. M1 cannot close until these remain within project-defined tolerances.
* **Homogeneity pre-pass:** over a seeded corpus, `coarse_bound` never returns `DEFINITELY_AIR` or `DEFINITELY_SOLID` for a chunk whose full evaluation is non-homogeneous.
* Explosion edit: every canonical sample within `R` of the center becomes air; canonical samples outside the sphere are unchanged; dependent mesh windows reconstruct the new values.
* Surface Nets: a sphere SDF produces a closed mesh with no boundary edges, including at chunk boundaries.
* **Quad emission convention:** a chunk emits quads only for cells `0..N-1`; no chunk emits a cell at index `N`; no sample outside `−1..N+1` is read.
* **Multipart meshing:** a world meshed as one 64³ region equals the union of the same world meshed as **2×2×2** chunks of 32³. Any vertex or index divergence is a seam bug. (v3 specified 2×2, which cannot exercise the Z axis.)
* **Compact promotion:** an `EMPTY` chunk edited once promotes to `DENSE` with all canonical samples initialized from the air marker; an edited `SOLID` chunk promotes likewise.
* **Load-order independence:** loading/generating chunks in different orders produces the same mesh snapshots once the required owner-chunk closure is resident or known absent.

Serialization (3.5, 3.6):

* Save/load round-trip is bit-exact for canonical `int16_t` payloads. Homogeneous chunks restore as `EMPTY` or `SOLID`.
* **No version bracket:** a mutation landing during a Persist snapshot does not cause a discard or a torn save.
* Region file: a crash during write (simulated by truncating the temp file) leaves the last good region readable.
* **Header validation:** a region file whose world seed or generator version does not match the current world is rejected rather than loaded.
* **Edited-chunk-only saving:** an unedited chunk is not written to disk at all.

Navigation state model (4.5), pure C++ with a fake `INavServerSync`:

* **Terrain revision:** an edit phase that changes canonical samples increments `terrain_revision` exactly once; Load/Generate/Unload do not.
* **Revision max-merge:** out-of-order `EditPhaseCompleted` records never move a desired revision or a per-chunk watermark backwards.
* **Group source epoch:** load/generate/unload changes increment only the affected group's `source_epoch`.
* **Group invalidation closure:** an edit affecting a border-padding dependency invalidates every impacted group, not just the group owning the edited chunk.
* **Watermark readiness:** a group is not bake-ready until every source chunk's committed mesh version meets the recorded watermark, even when its revision counters already match.
* **Stale rebuild:** a navigation group rebuilt at revision `N` is rejected as synchronized when its desired revision is `> N`.
* **Rebuild race:** a second edit arriving during an in-flight group rebuild causes a second rebuild from the latest desired revision/epoch, without partial-state exposure.
* **Debounce:** a group does not submit until `nav_rebuild_debounce_ms` of quiet has elapsed, and never waits longer than `nav_rebuild_max_delay_ms`.
* **Quiescence:** with a static agent position and no edits, `all_required_synced()` becomes true and stays true.
* **Sync predicate:** `navigation_groups_synced()` remains false until every required group matches its desired revision/epoch and the fake server reports the submission synchronized.

### 7.2 Integration Tests (Godot)

* Run the game through the Godot MCP server (`project_run`). Use `game_eval` or `game_manage` to trigger explosions and verify the completion queue drains.
* `trigger_explosion` returns in under 1 ms of main-thread time for normal validation inputs and never blocks on worker execution, locking, or queue capacity.
* `set_stream_focus` does not perform disk IO, generation, meshing, or Godot resource creation synchronously; it only updates the main-thread streaming focus state.
* Verify `explosion_requested` fires before `explosion_completed`, and that `explosion_completed` fires only after all Edit phases finish.
* Verify `fully_committed()` is false while a completed chunk result sits in the MPSC queue and remains false while a required navigation region group is rebuilding.
* Verify `fully_committed()` becomes true while the player is moving, once the nav maintenance radius and debounce are in force.
* Verify `chunk_committed` fires as soon as the visual mesh and collision are live, without waiting for navigation.
* Verify a large explosion is visibly progressive but never produces more than the configured completion-queue depth or memory budget.
* Measure main-thread frame time during small and large explosions.
* Verify `VoxelAICompanion.get_voxel_path()` returns a path after an explosion only once every affected navigation group is synchronized.
* Verify no navigation gap at a 4×4 region-group boundary with the configured agent radius and connection radius.
* Verify an edit near a region-group border invalidates both the owning group and any adjacent group whose padding source intersects the edit.
* Verify navigation groups created by Load/Generate and removed by Unload advance `source_epoch` and rebuild without changing `terrain_revision`.
* Verify `trigger_explosion` called before `_enter_tree()` or after `_exit_tree()` fails cleanly instead of dereferencing a null engine.
* **Editor guards:** instantiating the scene in the editor creates no worker threads and performs no streaming; exiting the editor writes no region files; the node keeps draining while the tree is paused.

### 7.3 Performance Targets (to validate, not assumed)

Targets apply to the **reference hardware**: an 8-core x86-64 CPU, an SSD, and a mid-range discrete GPU. Record the actual machine in `AGENTS.md` when M0 completes; until then these numbers are targets, not commitments.

| Metric | Target | Notes |
| :--- | :--- | :--- |
| Explosion, main-thread cost, mean | < 2 ms per frame | Enforced by the soft upload budget (4.3) |
| Explosion, main-thread cost, p99 frame | < `upload_budget_ms` + one chunk commit | The budget exception is bounded and measured |
| Explosion, time to visible result | < 3 frames for a 4-chunk edit | Depends on worker load |
| Large edit phase lock set | ≤ `kMaxEditPhaseChunks`, and only on owner chunks | Hard correctness invariant, not a tuning target |
| **Commit throughput** | **≥ 8 dense-chunk commits per frame within a 4 ms budget** | Validated by the M0 spike; vertex upload and collision shape creation measured independently |
| Generated terrain surface error | Project-defined tolerance | Validate the density-to-SDF approximation before M1 closes |
| Generated SDF near-surface gradient | Project-defined tolerance around 1.0 | Validate local distance behavior before repeated CSG edits |
| Navigation region update, single group | < 10 ms submission overhead on main thread | Actual bake is asynchronous and must be measured separately |
| Navigation rebuild debounce | 200 ms quiet, 1000 ms maximum | Makes the system quiescent under continuous streaming |
| Chunk memory, dense core block | ≤ 64 KiB resident per chunk | `N³ × 2` bytes |
| Chunk memory, `EMPTY`/`SOLID` | 0 bytes of sample data | From 3.1 |
| Chunk material memory | ≤ 32 KiB when allocated | `N³` canonical `uint8_t` samples |
| Streaming budget utilization | ≤ 100% of `resident_memory_budget_mb` | Hard admission bound for engine-managed CPU memory, including transient worker/completion buffers and the nav source cache |
| Completion queue depth | < 32 entries during steady state | 32 is the hard backpressure threshold |
| Active chunk admission | Meets `view_distance_chunks` target when memory permits | Report effective radius separately when budget-constrained |
| Mesh commit, split cost | vertex copy / collision shape / nav submission reported separately | Navigation bake is asynchronous and measured at group scope |

Commit cost must be reported as **average, p95, p99, and worst-case dense chunk**, for vertex upload and collision shape creation independently. Navigation submission and asynchronous bake time are reported per region group. A single aggregate average hides the case that actually causes a visible hitch.

---

## 8. Open Questions and Risks

1. **Chunk size vs. commit cost.** `kChunkSize` is a compile-time constant defaulting to 32. A 32³ chunk with a 35³ mesh window is 64 KiB of core data and may exceed 2 ms to commit on some hardware. The M0 spike builds and measures both `N = 32` and `N = 16`; the default is changed only on measured evidence.
2. **Streaming memory budget vs. view distance.** A 16-chunk Chebyshev radius with its mesh apron contains 42,875 chunk positions and cannot fit in a 512 MiB budget if all chunks are dense. The engine therefore treats `view_distance_chunks` as a target and reports an effective radius when memory-constrained.
3. **Mesh window reconstruction cost.** A Mesh job resolves up to 27 owner chunks and copies up to ~1.7 MB to produce a 70 KB window. If this shows up in profiling, add a per-chunk neighbor-pointer cache (invalidated on residency change) rather than relaxing the read rule.
4. **Collision memory.** Concave shapes require a de-indexed triangle soup, which is roughly 3× the indexed vertex data (4.4). Instrument from M3. Planned escape hatch: a decimated or voxel-aligned collision mesh.
5. **Navigation group size.** 4×4 X/Z chunks is the default. Benchmark 4×4 versus 8×8 before M4 closes while preserving the explicit padding/dependency rules.
6. **Async navigation API availability.** Verify the required `NavigationServer3D` APIs exist at the pinned Godot version during M0. If not, either raise `compatibility_minimum` or change the synchronization implementation.
7. **Cross-machine FastNoise2 determinism.** Accepted for v4. Revisit if multiplayer or seed-shared worlds are added.
8. **Edit-phase tuning.** `kMaxEditPhaseChunks = 32` and `kMinEditPhaseChunks = 4` are safety defaults. Tune them from measured lock hold time and worker contention; the hard invariant is that a phase never locks a chunk it does not write.
9. **Flush interval.** Region files flush per region, not per chunk (3.6). Choose a dirty-chunk count or time interval at M4 and record the resulting maximum data-loss window.
10. **Region size.** 16×16 chunks bounds rewrite amplification for sparsely edited regions. If profiling shows flush cost is still too high, the next lever is a per-chunk append journal; that is a format change and needs its own design pass.
11. **Nav source cache growth.** Retaining committed mesh arrays for the bake costs memory outside the chunk ledger. If it dominates, the fallback is reading triangles back from `RenderingServer`, which is slower but bounded.

---

## 9. Milestones

| ID | Milestone | Exit criteria |
| :--- | :--- | :--- |
| M0 | Core skeleton + contract + vertical slice | ChunkKey, canonical sample ownership, COW sample blocks, compact forms, streaming planner + mesh apron + hard memory admission, bounded Edit phases, dirty-set Mesh admission, worker pool, teardown, static linkage; **vertical slice: generate → mesh → commit → collide running in Godot, with the `RenderingServer` surface path and collision shape cost measured against the 4.3 targets**; `kChunkSize` parameterized and measured at 32 and 16; **all 3.2/3.3/3.4 contract tests pass under TSan/ASan** |
| M1 | Procedural generation and meshing | Gradient-normalized SDF approximation; Surface Nets from canonical-owner mesh windows; multipart-meshing equivalence (2×2×2) and quad-emission tests pass; homogeneity pre-pass validated against full evaluation; generated surface/gradient error remains within project-defined tolerances |
| M2 | Edits | Explosion CSG on canonical samples with bounded phases, dirty-window propagation, canonical lock order, completion backpressure test, and sphere-SDF closed-mesh test |
| M3 | Godot integration | `VoxelTerrain3D` renders and collides; main-thread commit meets the measured throughput and frame-budget envelope; large edits cannot exceed completion queue depth or engine-managed memory budget; editor guards verified |
| M4 | Persistence and navigation | Region-file saves with seed/generator-version validation and the persist failure policy; pure-C++ nav group state machine with watermark readiness, debounce, and quiescence; `fully_committed()` and `navigation_groups_synced()` pass stale-rebuild tests; `VoxelAICompanion` path works after edits, including across region-group borders |
| M5 | LOD | Octree or clipmap LOD, Transvoxel transitions |

M0 closes only when the lock-set invariant, dirty-set handoff, terrain-revision contract, per-group navigation sync state, and the commit-throughput spike are executable tests and measurements rather than documentation-only rules.
