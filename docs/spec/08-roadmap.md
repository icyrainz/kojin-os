# 10. MVP Scope

The MVP is an existence proof that an AI can bootstrap itself onto raw hardware through reasoning alone, maintain persistent understanding of its body, and fulfill human intents by directly manipulating hardware.

## 10.1 In Scope

- Bare-metal Rust exokernel (~3,000–3,500 LOC) booting on Pi 5
- Capability-based MMIO access for all hardware described in device tree
- UART as the single hardcoded peripheral (the irreducible minimum)
- Bridge device protocol for LLM access + temporary storage over UART
- AI runtime that reasons about hardware through external LLM
- Bootstrap sequence: AI brings up Ethernet, then SD, cutting the bridge
- Persistent knowledge graph with crash-safe append-only storage
- At least three working intents: sensor monitoring, GPIO actuation, scheduled tasks
- Intent persistence and resume across reboots
- Full observability: every register write, LLM reasoning chain, and intent decision logged as knowledge graph events
- Configurable UART verbosity levels (quiet/normal/verbose/raw)
- Safety gates for first-time hardware operations (human confirmation)
- Hardware portability: demonstrate on Pi 5 (RP1 via PCIe), document path for other boards

## 10.2 Out of Scope (Future Work)

- On-device inference (local SLM to reduce LLM dependency)
- Multi-agent support (multiple AI runtimes with capability isolation)
- Audio/visual interface (microphone, speaker, display)
- WiFi (requires firmware blobs — Ethernet first)
- OTA updates
- Security hardening beyond basic capability isolation
- Formal verification of the exokernel

---

# 11. Implementation Roadmap

## Phase 1: Bare-Metal Hello World (Week 1–2)

**Toolchain:** `rustup target add aarch64-unknown-none`, `cargo install cargo-binutils`, QEMU 7.0+.
**QEMU invocation:** `qemu-system-aarch64 -M virt,gic-version=2,virtualization=on -cpu cortex-a76 -nographic -kernel kojin.bin`

- Set up Rust cross-compilation toolchain for `aarch64-unknown-none`
- Boot a minimal kernel on QEMU `virt` that writes to PL011 UART
- Implement basic MMU setup (identity map + kernel virtual map + trampoline)
- **Key milestone:** EL1→EL0 handshake: EL0 issues `svc #0`, EL1 prints "EL0 handshake OK"
- Implement heap allocator (`linked-list-allocator` crate, enables `alloc` for all later phases)
- **Buy Pi 5 hardware now** (8GB, USB-UART adapter, good SD card, 5A USB-C PSU)

## Phase 2: Exokernel Core + Async Executor (Week 3–5)

- Implement device tree parser (extract MMIO regions, IRQs)
- Implement capability system: `MMIOCap`, `MemCap`, `DmaCap`, `IRQCap`
- Build EL0→EL1 syscall interface for capability-gated MMIO access
- Interrupt routing through GIC-400 (GICv2)
- **Async executor:** Single-core `embassy-executor` on Core 1, or minimal bespoke scheduler (~300 LOC). Multi-core is post-MVP.
- Build TF-A `armstub8-2712.bin` for Pi 5 (official TF-A `rpi5` platform target)
- Test on real Pi 5: EL0 runtime can read/write MMIO through capabilities

## Phase 3: Bridge Protocol + LLM Client (Week 6–7)

- Implement UART-multiplexed bridge protocol (binary framing: `LLM_REQ`/`LLM_RES`/`LLM_ERR`, `STORE`/`STORE_ACK`/`LOAD_REQ`/`LOAD_RES`, `HUMAN_IN`/`AI_OUT`, `SYNC`)
- Build bridge device software (simple serial-to-HTTPS proxy)
- Implement LLM client in the runtime (send prompts, parse responses)
- Test: AI can reason through the bridge — ask LLM a question via UART, get response

## Phase 4a: Hardware Reasoner + GPIO Proof (Week 8–10)

**This phase validates the core thesis.** If the LLM can correctly reason through PCIe→RP1→GPIO, the thesis is proven.

- Design entity/edge schema in Rust structs (needed for HardwareKnowledge caching)
- Implement in-memory knowledge graph with query API (get-by-id, neighbors, filter-by-type)
- Implement Hardware Reasoner: device tree → LLM prompt → register operations → execute via capabilities
- Build mock LLM server (pre-recorded JSON responses) for offline testing and regression
- AI reasons about GPIO via RP1: PCIe ECAM enumeration → RP1 BAR discovery → GPIO pin HIGH/LOW (LED visible!)
- Note: `pciex4_reset=0` in `config.txt` lets firmware pre-initialize PCIe, simplifying early RC bring-up
- **De-risk strategy:** validate the full reasoning pipeline with mock LLM (pre-recorded responses) first, then attempt live LLM reasoning. This separates "is the pipeline buggy?" from "can the LLM do PCIe→RP1?"

**GO/NO-GO GATE (end of Phase 4a):** Can the LLM produce correct GPIO register sequences via PCIe→RP1 with live reasoning? If yes → proceed to Ethernet. If no → assess failure mode:
- LLM consistently wrong about RP1 register layout → try different model or supplement prompts with raw datasheet excerpts
- LLM correct about registers but pipeline can't execute → fix pipeline, retry
- Fundamental inability to reason at this level → rescope thesis to simpler peripherals (direct MMIO on Pi 4) or hand-write RP1 driver and focus thesis on higher-level hardware reasoning (I2C/SPI sensors, GPIO on simpler paths)

## Phase 4b: Ethernet + First Cord Cut (Week 11–14)

**Highest risk phase.** RP1 Ethernet MAC bring-up through PCIe requires DMA address translation (`0x10_0000_0000` inbound offset on BCM2712). Known from community bare-metal attempts to be the hardest single task.

- AI reasons about Ethernet: MAC bring-up through RP1, DMA ring descriptors, PHY reset via MDIO
- Implement smoltcp integration for TCP/IP
- Build HTTP client on top of smoltcp (~300 LOC: HTTP/1.1 framing, chunked decoding)
- First cord cut: AI reaches LLM directly over Ethernet (HTTP initially; TLS via `embedded-tls` is stretch goal)
- Fallback: if Ethernet proves too hard for LLM reasoning, deploy a local HTTP proxy that strips TLS

**GO/NO-GO GATE (week 13, mid-phase):** Is Ethernet making progress — can the LLM produce register sequences that get partial results (MAC reset, DMA ring setup, PHY link detection)? If yes → continue, budget extra time. If no after 2 weeks of iteration → choose a fallback:
- (a) Fall back to Pi 4 where Ethernet is direct MMIO (easier reasoning target)
- (b) Hand-write the Ethernet driver, pivot thesis to "AI reasons about peripherals on top of a hand-written network stack"
- (c) Use USB-Ethernet adapter with simpler register model
- Document the decision and reasoning — a failed attempt at RP1 Ethernet is still valuable data about LLM hardware reasoning limits

## Phase 5: SD Persistence + Second Cord Cut (Week 15–16)

- Implement MessagePack serialization for graph mutations (`rmp` crate, no_std + alloc)
- AI reasons about SD card: SDHCI register operations, block read/write
- Build append-only log writer to SD blocks
- Implement snapshot double-buffering + compaction
- Migrate knowledge from bridge storage to local SD
- Second cord cut: AI is fully independent
- Implement snapshot + log replay for crash recovery
- Test: power cycle → AI cold boots independently

## Phase 6: Intent System + Integration (Week 17–20)

- Implement Intent Decomposer: natural language → LLM → task DAG
- Build three reference intents: sensor polling, threshold alerts, scheduled actions
- AI reasons about I2C to talk to BME280 temperature sensor
- Wire intents to knowledge graph for persistence
- End-to-end test: boot → bootstrap → accept intents via UART → execute → persist → reboot → resume
- Stress test: 72-hour continuous operation
