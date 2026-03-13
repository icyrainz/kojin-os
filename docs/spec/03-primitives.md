# 4. The Primitives

Traditional operating systems have six core primitives: processes, threads, files, sockets, signals, and pipes. KojinOS has five entirely different ones.

## 4.1 Intent

The fundamental unit of work. An intent is a goal expressed in natural language that the AI decomposes into executable hardware operations through LLM-powered reasoning.

```
Intent: "Alert me if the room gets above 28°C"
  → AI reasons: what sensor? → BME280 on I2C → what address? → 0x76
  → AI reasons: how to configure I2C on this hardware? → register sequence
  → AI reasons: how to read temperature? → BME280 protocol
  → AI reasons: how to drive buzzer? → GPIO pin, output mode, set HIGH
  → Plan: [ConfigI2C(reasoned), PollBME280(reasoned), OnThreshold(DriveGPIO(reasoned))]
  → Tasks: [SensorTask, ThresholdWatcher, ActuatorTask]
```

Intents are persistent. The AI remembers them across reboots via the knowledge graph. When KojinOS wakes up, it resumes all active intents.

## 4.2 Capability

An unforgeable token that grants access to a specific hardware resource. The exokernel issues capabilities; the AI runtime exercises them through the syscall interface.

| Capability Type | Grants Access To | Issued By | Example |
|---|---|---|---|
| `MMIOCap` | Device register region | Exokernel at boot (from device tree) | `MMIOCap(0x107d001000, 0x1000)` (debug UART) |
| `MemCap` | Virtual page range | `sys_mem_alloc` | `MemCap(virt_addr, 4096)` |
| `DmaCap` | Physically contiguous buffer | `sys_dma_alloc` | `DmaCap(virt_addr, phys_addr, 4096)` |
| `IRQCap` | Interrupt line | Exokernel at boot (from device tree) | `IRQCap(IRQ_I2C1)` |

`DmaCap` is the key addition: it carries both virtual and physical addresses. The AI writes DMA descriptors using the physical address (which the hardware needs) and reads/writes buffer contents using the virtual address (which the CPU needs). The exokernel guarantees the buffer is contiguous, properly aligned, and belongs to the runtime.

**Capability granting at boot.** The exokernel walks the device tree's `reg` properties to create `MMIOCap` tokens for each MMIO region, the `interrupts` / `interrupt-map` properties to create `IRQCap` tokens for each interrupt line, and the PCIe ECAM range (from the device tree's `reg` property on the PCIe host controller node) to create an `MMIOCap` covering the PCIe configuration space. The AI uses `sys_caps_count()` and `sys_cap_get(index)` to enumerate all capabilities and correlates them with device tree entries to understand which cap corresponds to which peripheral.

**Dynamic capabilities for PCIe.** After boot, the AI enumerates the PCIe bus by reading/writing ECAM configuration space via `sys_mmio_read32`/`sys_mmio_write32` (ECAM addresses are calculated as `bus << 20 | dev << 15 | func << 12 | register_offset`). When the AI discovers a device's BAR address (e.g., RP1's MMIO region), it calls `sys_cap_request_mmio(phys_addr, size)` to get a new `MMIOCap`. The exokernel validates that the requested region falls within a PCIe window declared in the device tree's `ranges` property before granting.

Capabilities are the AI's "permission slips" for hardware. The AI can reason about what to do with a register region, but it can only touch the region if it holds the corresponding capability. This boundary is enforced by the exokernel — the reasoning layer above is swappable without changing the capability model.

## 4.3 Entity

The unit of persistent knowledge. Replaces files. An entity is a node in the knowledge graph with a type, attributes, and relationships to other entities.

```
Entity { type: HardwareKnowledge, id: "hw:ethernet_init",
         attrs: { register_sequence: [...], validated: true } }
Entity { type: Sensor, id: "sensor:bme280_room",
         attrs: { bus: I2C1, addr: 0x76 } }
Entity { type: Intent, id: "intent:temp_alert",
         attrs: { condition: "temp > 28", action: "gpio17 HIGH" } }
Edge { from: "intent:temp_alert", to: "sensor:bme280_room", rel: MONITORS }
```

**Entity IDs.** IDs are type-prefixed strings: `hw:ethernet_init`, `sensor:bme280_room`, `intent:temp_alert`, `event:00000042`. The prefix enables fast type filtering without scanning attributes. For high-volume entities (sensor readings, events), the runtime uses a monotonic counter suffix (`reading:00000001`, `reading:00000002`) rather than UUIDs — cheaper to generate, naturally ordered, and sufficient for a single-writer system.

**Attribute size limits.** Individual attributes are capped at 64 KB (enough for a full register sequence or LLM conversation snippet). Total entity size (all attributes serialized) is capped at 256 KB. Entities exceeding this should be split — e.g., a long LLM conversation becomes multiple linked `Event` entities rather than one massive blob.

**Granularity conventions.** High-frequency data (sensor readings) uses one entity per reading with a compact attribute set (timestamp, value, unit). This keeps entities small and enables time-range queries without deserializing large objects. Low-frequency data (hardware knowledge, intents) uses richer single entities with nested attributes. The AI can reason about compaction strategies — e.g., downsampling old readings from per-second to per-minute after 24 hours.

Entities are persisted to the SD card. The knowledge graph supports semantic queries: "what sensors exist?", "what was the temperature at 3pm?", "how do I initialize Ethernet on this board?"

## 4.4 Task

A lightweight coroutine managed by the async executor. Tasks are spawned by the Intent Decomposer and represent individual hardware operations or reasoning steps within a plan.

Tasks share the AI runtime's address space — no isolation, because they are all part of the same intelligence.

- **Lifecycle:** Spawned → Running → Waiting (on hardware/timer/LLM response) → Complete | Failed
- **Concurrency model:** Cooperative async (Rust futures). Hardware interrupts can wake sleeping tasks.
- **Watchdog:** Tasks that exceed their deadline are cancelled. The AI logs the failure and can reason about why it happened.

### Multi-core Execution (SMP)

MVP uses single-core cooperative async on Core 1, with LLM calls as non-blocking futures. Core 0 handles exokernel interrupt dispatch (GIC target). Cores 2–3 are parked.

If LLM latency (200-500ms network, 1-3s UART) causes missed sensor deadlines, the architecture supports promoting to multi-core: pin I/O tasks to Core 2, keep reasoning on Core 1, communicate via lock-free MPSC channels. Cache coherency between cores is automatic via the Cortex-A76 DSU cluster. The knowledge graph remains single-writer (Core 1 owns mutations). This promotion is a post-MVP optimization — single-core is simpler and sufficient until proven otherwise.

## 4.5 Channel

Typed, bounded, async communication between tasks and subsystems.

- Sensor tasks publish readings to channels
- The UART interface publishes user messages to a channel the AI monitors
- The AI publishes responses to a channel the UART interface drains
- The LLM client sends/receives through channels

**Capacity and backpressure.** All channels have a fixed capacity set at creation. Default sizes:

| Channel | Capacity | Backpressure |
|---|---|---|
| UART input (human → AI) | 16 messages | Block sender (UART buffer absorbs) |
| UART output (AI → human) | 64 messages | Drop oldest (human can't read that fast anyway) |
| Sensor readings | 32 per sensor | Drop oldest (stale readings are worthless) |
| LLM request/response | 4 | Block sender (natural throttle on LLM calls) |
| Cross-core mutation requests | 64 | Block sender (applies backpressure to I/O core) |

The UART input channel has implicit priority — human messages are always processed before sensor-triggered reasoning, so the AI stays responsive to the human even under load.

Channels are in-memory only. Persistent communication happens through the knowledge graph.
