# 12. Landscape and Differentiation

| Project | Approach | KojinOS Difference |
|---|---|---|
| Steve OS | AI-native OS with shared agent memory | Still runs on conventional OS. KojinOS eliminates the host OS entirely. |
| AthenaOS | Rust/C++ with swarm agents managing processes | Agents manage traditional OS concepts. KojinOS has no processes to manage. |
| OpenAI OS vision | ChatGPT as app platform | Cloud-dependent, app-store model. KojinOS is hardware-native with intimate register-level control. |
| Agent-OS (TechRxiv) | Microkernel for agent coordination | Multi-agent orchestration at scale. KojinOS is single embodied intelligence that reasons about its own body. |
| llm-baremetal | LLM running on UEFI, no OS | Inference without OS. No persistent state, no hardware reasoning, no intent system. |

KojinOS is unique in that the AI has no pre-written drivers — it reasons about raw hardware through an external LLM, bootstraps itself from a single UART channel, and learns to operate its own body. The exokernel is ~3,000–3,500 lines. Everything above that is intelligence.

---

# 13. Open Questions

These are genuinely unresolved design decisions that the MVP will help answer:

- **LLM latency tolerance.** Hardware reasoning requires LLM round-trips (~200-500ms each). How does this affect real-time sensor monitoring? Can the AI cache enough hardware knowledge that routine operations don't need LLM calls?

- **Knowledge graph scalability.** How large can the in-memory graph get before RAM is exhausted? What's the right eviction strategy? Can the AI reason about what to forget?

- **Hardware reasoning reliability.** The validation pipeline (Section 2.4) provides readback validation and the bootstrap error handling (Section 3.2) defines retry limits. Remaining question: how often will the LLM produce incorrect sequences in practice? What's the empirical failure rate for PCIe enumeration, Ethernet bring-up, and I2C configuration? This determines whether 3 retries is sufficient.

- **Bootstrap resilience.** Section 3.2 defines retry policy (3 retries per step, 30s timeout, 10min total). Remaining questions: what if the AI gets stuck in a loop of plausible-but-wrong register sequences that pass readback but don't achieve the goal (e.g., link never comes up despite correct-looking register writes)? How does the AI detect "correct writes, wrong outcome" vs. "incorrect writes"?

- **LLM dependency.** The system is non-functional without LLM access. Can we implement a lightweight local rule engine for cached operations so routine intents execute without LLM round-trips?

- **Multi-board portability.** How much of the AI's hardware knowledge transfers between similar boards (e.g., Pi 5 to Pi 4)? Can it adapt existing knowledge or must it reason from scratch?

- **Development ergonomics.** Section 7 defines four verbosity levels and structured event logging. Remaining questions: what tooling is needed on the host side to make sense of the UART event stream? Is a TUI viewer sufficient, or do we need a web-based dashboard? How do we replay and inspect reasoning chains from the event log offline?

- **Error recovery architecture.** What happens when the LLM is unreachable (network down after bootstrap)? The AI cannot reason without its brain. Options: (a) continue executing cached intents from the knowledge graph without LLM, (b) enter a degraded mode that maintains running tasks but cannot accept new intents, (c) queue new intents and process them when LLM reconnects. What about hardware errors — a hung I2C bus, a DMA timeout, an SD write failure?

- **Clock and time source.** The Pi 5 has an RTC header but no battery by default. All timestamps in the knowledge graph (Section 5.2) use the ARM monotonic timer until the AI brings up Ethernet and can reach NTP. The graph must handle the transition from monotonic-only to wall-clock time — early entities have relative timestamps that need rebasing once NTP provides an absolute reference.

- **Power loss resilience.** The Pi 5 has a PMIC (power management IC) accessible via I2C. The AI could reason about reading it to detect brown-out conditions and flush the knowledge graph preemptively. Alternatively, the append-only log with per-entry CRC (Section 5.3) provides crash safety without graceful shutdown. MVP relies on the latter; proactive voltage monitoring is a future improvement.

- **Testing strategy.** Unit testing the exokernel is straightforward (QEMU). Testing the hardware reasoner is harder — options include: recorded LLM conversations replayed as fixtures, a mock LLM server returning known-good register sequences, and physical hardware-in-the-loop testing on real Pi 5. The validation pipeline (Section 2.4) provides a natural test oracle — if readback matches expectations, the sequence is correct. The MVP should maintain a suite of recorded LLM conversations for regression testing.

---

# 14. Success Criteria

**Exokernel success criteria** (testable on QEMU, no AI runtime needed):

- EL0 code can issue all syscalls and get correct results
- Capability validation rejects out-of-range offsets and invalid handles
- MMU correctly isolates EL0 from EL1 memory (EL0 access to EL1 pages faults)
- `sys_irq_wait` wakes an EL0 task when the interrupt fires
- SD PIO read returns correct data from a QEMU block device
- `sys_timer_now` returns monotonically increasing values

**End-to-end success criteria** (requires Pi 5 + LLM):

- The exokernel boots on Pi 5 with UART + SDHCI PIO read as the only hardcoded drivers (~3,000–3,500 LOC).
- The AI bootstraps from UART bridge: reasons about Ethernet, brings it up, reaches LLM directly.
- The AI reasons about SD card, brings up persistence, migrates knowledge from bridge.
- The bridge is fully disconnected — AI operates independently (with only remote LLM endpoint).
- A user types "read the temperature every 10 seconds and flash the LED if it's above 28 degrees" and the AI reasons about I2C, GPIO, and the sensor protocol to make it happen.
- Intents persist across power cycles. Unplug, plug back in, AI cold boots from SD knowledge and resumes.
- The knowledge graph records 24+ hours of sensor history.
- The system runs stable for 72 hours without crashes or memory leaks.
- Every hardware operation is traceable: a human can ask "what did you do in the last hour?" and get a complete, structured answer from the knowledge graph.
- Safety gates work: the AI asks for confirmation before first-time register writes to unknown peripherals.

---

# 15. Key References

- [rust-raspberrypi-OS-tutorials](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials) — Bare-metal Rust kernel tutorials for Pi 3/4
- [Exokernel: An OS Architecture for Application-Level Resource Management](https://pdos.csail.mit.edu/6.828/2008/readings/engler95exokernel.pdf) — Engler et al., MIT
- [smoltcp](https://github.com/smoltcp-rs/smoltcp) — `no_std` TCP/IP stack for embedded Rust
- [llm-baremetal](https://github.com/djibydiop/llm-baremetal) — LLM running on UEFI without an OS
- [RusPiRo](https://github.com/RusPiRo/ruspiro-kernel) — Rust bare-metal development for Raspberry Pi
- [OSDev Wiki: Exokernel](https://wiki.osdev.org/Exokernel) — Community reference on exokernel design

---

# 16. Project Structure

```
kojin-os/
├── docs/
│   └── SPEC.md                    # This file
├── kernel/                        # Exokernel (~3,000–3,500 LOC)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                # Entry point, boot handoff
│   │   ├── platform.rs            # Platform trait definition (Tier 1 swap interface)
│   │   ├── capability.rs          # Capability types and validation (Tier 3 — generic)
│   │   ├── syscall.rs             # The boundary — EL0↔EL1 syscall interface (Tier 3)
│   │   ├── dma.rs                 # DMA buffer allocation + cache maintenance (Tier 3)
│   │   └── arch/                  # Tier 2 — board-specific Platform implementations
│   │       ├── pi5/               # Raspberry Pi 5 target (~1,500–2,000 LOC)
│   │       │   ├── mod.rs         # Pi5Platform impl
│   │       │   ├── uart.rs        # PL011 UART at 0x107d001000 (~80 LOC)
│   │       │   ├── mmu.rs         # ARM64 translation tables + identity map trampoline
│   │       │   ├── gic.rs         # GIC-400 (GICv2) + PCIe MSI demux (~300 LOC)
│   │       │   ├── pcie.rs        # BCM2712 PCIe RC init + link training (~300 LOC)
│   │       │   ├── sd.rs          # SDHCI PIO (EMMC2 at 0x1000fff000, ~500 LOC)
│   │       │   ├── devicetree.rs  # DTB parser
│   │       │   ├── linker.ld      # Pi 5 memory map
│   │       │   └── boot.S         # ARM64 entry (EL1 from TF-A, PSCI for SMP)
│   │       └── qemu_virt/         # QEMU virt target (shares most ARM64 code)
│   │           ├── mod.rs         # QemuPlatform impl
│   │           ├── linker.ld      # QEMU memory map
│   │           └── boot.S         # QEMU-specific entry
├── runtime/                       # AI Runtime ("The Mind")
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                # Runtime entry, receives capabilities
│   │   ├── llm/                   # LLM client
│   │   │   ├── mod.rs
│   │   │   ├── client.rs          # Send prompts, parse responses
│   │   │   └── bridge.rs          # UART bridge protocol (bootstrap)
│   │   ├── reasoner/              # Hardware reasoning
│   │   │   ├── mod.rs
│   │   │   ├── hardware.rs        # Device tree → LLM prompt → register ops
│   │   │   ├── fdt.rs             # FDT parser — extracts subtrees for LLM prompts
│   │   │   └── validator.rs       # Read-back validation of register writes
│   │   ├── graph/                 # Knowledge graph engine
│   │   │   ├── mod.rs
│   │   │   ├── entity.rs          # Node types and attributes
│   │   │   ├── edge.rs            # Relationship types
│   │   │   ├── storage.rs         # Append-only log, snapshots
│   │   │   └── query.rs           # Traversal and filter API
│   │   ├── intent/                # Intent decomposition
│   │   │   ├── mod.rs
│   │   │   ├── decomposer.rs      # NL → LLM → task DAG
│   │   │   └── planner.rs         # DAG optimization
│   │   ├── executor/              # Async task executor
│   │   │   ├── mod.rs
│   │   │   ├── task.rs            # Task lifecycle
│   │   │   └── channel.rs         # Typed async channels
│   │   ├── observe/               # Observability system
│   │   │   ├── mod.rs
│   │   │   ├── events.rs          # Event types (RegisterWrite, LLMRequest, etc.)
│   │   │   ├── logger.rs          # Event → knowledge graph + UART output
│   │   │   └── safety.rs          # Safety gates (human confirmation for new ops)
│   │   └── interface/             # Human interface
│   │       ├── mod.rs
│   │       └── serial.rs          # UART conversation loop (verbosity modes)
├── bridge/                        # Bridge device software
│   ├── Cargo.toml
│   └── src/
│       └── main.rs                # Serial-to-HTTPS proxy + key/value store
└── tools/
    ├── flash.sh                   # SD card image builder
    ├── serial.sh                  # UART monitor helper
    └── qemu-run.sh                # QEMU testing
```
