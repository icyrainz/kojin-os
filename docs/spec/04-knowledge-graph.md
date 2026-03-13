# 5. The Knowledge Graph

The knowledge graph is not a database the OS uses. It IS the storage layer. There are no files. There are no directories. There is only the graph.

## 5.1 Why Not a Filesystem

A filesystem is an abstraction designed for humans who think in hierarchies. An AI thinks in relationships, embeddings, and semantic proximity. Asking an AI to store data in `/home/user/sensors/temperature/2026-03-12.csv` forces it to translate its native representation into an alien format. KojinOS eliminates this translation entirely.

## 5.2 Schema

The knowledge graph uses a property-graph model with typed nodes and edges:

- **Nodes** have a unique ID, a type, and a bag of typed attributes.
- **Node types include:** `HardwareKnowledge` (learned register sequences), `Sensor`, `Reading`, `Intent`, `Task`, `Event`, `Config`, `NetworkKnowledge`, `SystemKnowledge`
- **Edges** are directed, labeled (`MONITORS`, `TRIGGERS`, `PRODUCED_BY`, `DEPENDS_ON`, `LEARNED_FROM`), and can carry attributes.
- **Temporal indexing:** Every node and edge carries a `created_at` and optionally an `expired_at` timestamp. This enables time-travel queries.

## 5.3 Storage Format

On-disk, the knowledge graph is stored as an append-only log of mutations serialized in MessagePack on the raw SD partition (Section 3.4). Periodically, the AI compacts the log into a snapshot.

- **Crash safety:** Each log entry is a self-contained MessagePack record with a CRC-32 trailer. On recovery, the runtime replays from the last valid snapshot, skipping any trailing entry with a bad CRC (partial write due to power loss). No "fsync" concept — we're writing raw blocks, so crash safety comes from the append-only structure and per-entry checksums.
- **Snapshot double-buffering:** The partition maintains two snapshot slots (A and B) plus the append log. Compaction writes a new snapshot to the inactive slot, then updates a 512-byte header block pointing to the active slot. The header contains a magic number and CRC-32 — the runtime reads both slots at boot and picks the one whose header is valid and has the higher sequence number. This means "atomicity" comes from the CRC check, not from SD hardware guarantees. If power is lost mid-header-write, the CRC will fail and the runtime falls back to the other slot.
- **Cheap writes:** appending is O(1).
- **Time travel:** replaying the log from a snapshot reconstructs any historical state.
- **Minimal SD card wear:** append-only with periodic compaction is friendlier to flash storage than random writes. The AI can reason about wear-leveling strategies if the graph grows large.
- **Partition full:** The runtime monitors free space and alerts the human via UART when the partition exceeds 80% capacity. When full, the runtime stops appending new events (losing crash safety for new data) rather than corrupting the log. The AI can reason about downsampling old sensor readings or pruning stale knowledge to reclaim space.

**Compaction procedure:** Compaction is triggered when the append log exceeds 50% of partition free space (or can be triggered by the AI). Steps: (1) Replay current active snapshot + all log entries into a new in-memory graph state. (2) Serialize the full graph to the INACTIVE snapshot slot. (3) Write a new header block with incremented sequence number and the new slot as active. (4) Reset `Log tail` to 0 (truncate the log). If power is lost during step 2, the old slot remains valid. If power is lost during step 3, the CRC check on the header will fail and the runtime falls back to the old slot. Compaction does not block reads — the in-memory graph continues serving queries during the write. Since the knowledge graph is single-writer (Section 4.4), compaction and log appends are serialized on Core 1 — no concurrent appends can occur during steps 1-4.

## 5.4 Hardware Knowledge

A special category of entities that store the AI's learned understanding of its hardware:

```
HardwareKnowledge:ethernet_init
  register_sequence: [
    { offset: 0x00, value: 0x80000000, description: "reset MAC" },
    { offset: 0x10, value: "{dma_tx_ring_phys}", description: "TX DMA ring base" },
    { offset: 0x14, value: "{dma_rx_ring_phys}", description: "RX DMA ring base" },
    ...
  ]
  parameters: ["dma_tx_ring_phys", "dma_rx_ring_phys"]
  validated: true
  learned_via: "LLM reasoning on RP1 Ethernet MAC via PCIe"
  device_tree_path: "/soc/ethernet@7d580000"
```

Register sequences distinguish between **literal values** (hardcoded constants like reset bits) and **parameterized placeholders** (values like DMA buffer physical addresses that change each boot). On replay, the runtime resolves parameters by calling the appropriate syscalls (e.g., `sys_dma_alloc`) and substituting the results before executing the sequence. This ensures cached knowledge works across reboots where physical addresses differ.

These entities are what enable subsequent boots without the bridge. The AI reads its own hardware knowledge from the graph and replays the register sequences to bring up peripherals.

## 5.5 Query Interface

The AI queries the graph through a native Rust API exposing traversal operations (neighbors, shortest path, subgraph extraction) and attribute filters. The LLM can also generate graph queries from natural language.

### In-memory Index Structures

The graph is deserialized into memory at boot. To support the expected query patterns without O(n) scans, the runtime maintains the following indexes:

| Index | Structure | Supports |
|---|---|---|
| ID lookup | `HashMap<EntityId, Entity>` | `get_by_id()` — O(1) |
| Type index | `HashMap<NodeType, Vec<EntityId>>` | `get_by_type(Sensor)` — O(1) lookup, O(k) iteration |
| Adjacency list | `HashMap<EntityId, Vec<(EdgeLabel, EntityId)>>` | `neighbors()`, `edges_from()` — O(degree) |
| Reverse adjacency | `HashMap<EntityId, Vec<(EdgeLabel, EntityId)>>` | `edges_to()` — O(degree) |
| Time index | `BTreeMap<Timestamp, Vec<EntityId>>` | Time-range queries on readings — O(log n + k) |

**Memory overhead estimate:** For a graph with 500K entities (mostly sensor readings after 72 hours), the indexes add roughly 40-60 MB on top of the entity data itself — well within the 512 MB budget. The time index is the largest consumer since every entity is indexed by `created_at`.
