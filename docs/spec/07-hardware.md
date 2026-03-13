# 8. Boot Sequence

Quick-reference summary. **Section 3 is authoritative** — this table is derived from it. See Section 3 for the full bootstrap narrative, error handling, and first-boot vs subsequent-boot flows.

| Phase | Component | Action |
|---|---|---|
| 0 | EEPROM + GPU Firmware | Pi 5 boots from onboard EEPROM (no `bootcode.bin`). Loads `config.txt`, kernel image, DTB from FAT32. |
| 1 | Exokernel | EL2→EL1 transition, MMU init, page tables. |
| 2 | Exokernel | Parse device tree, create capability tokens. |
| 3 | Exokernel | Initialize UART + SDHCI PIO. Read knowledge graph from SD partition 2 (if present). |
| 4 | Exokernel | Load AI runtime into EL0. The exokernel and runtime are a single binary linked together. The linker script (`linker.ld`) places kernel code in an EL1-only section (PXN=0, UXN=1) and runtime code in an EL0-executable section (PXN=1, UXN=0). The exokernel's MMU setup maps these sections with the appropriate page table attributes, then drops to EL0 and jumps to the runtime entry point. Pass: capabilities + device tree + UART handle + knowledge graph data (loaded directly into EL0-accessible memory to avoid copying). |
| 5a | AI Runtime (first boot) | No knowledge. Use bridge (Section 3.1) to bootstrap Ethernet + SD write. |
| 5b | AI Runtime (subsequent) | Replay learned Ethernet sequence from knowledge graph. Reach LLM directly. |
| 6 | AI Runtime | Resume persistent intents. "I'm awake" on UART. |

**Target boot time (subsequent boots):** under 10 seconds from power-on to "I'm awake." Breakdown: GPU firmware ~2-3s, exokernel init + SD read ~1s, Ethernet PHY auto-negotiation ~3-4s, TCP connect to LLM ~1s.

**Time source:** The ARM generic timer provides monotonic time from boot. Wall-clock time is unavailable until the AI brings up Ethernet and reaches NTP. Early knowledge graph entries use monotonic timestamps, which are rebased to wall-clock time once NTP synchronizes.

---

# 9. Hardware Target

## 9.1 Development: QEMU

Initial exokernel development uses `qemu-system-aarch64 -M virt,gic-version=2` for fast iteration. The `virt` machine provides PL011 UART, GICv2 (matching Pi 5's GIC-400), and virtio devices — ideal for developing MMU, capabilities, interrupt routing, and boot handoff without physical hardware.

**QEMU limitations:** The `virt` machine does not model the Pi 5's BCM2712, RP1 southbridge, or PCIe bus. All RP1/PCIe-dependent functionality (GPIO via RP1, Ethernet, Hardware Reasoner's PCIe enumeration path) can only be tested on real Pi 5 hardware. QEMU development validates the exokernel core, syscall interface, capability system, knowledge graph, and bridge protocol — not board-specific hardware reasoning.

## 9.2 Primary: Raspberry Pi 5 (8GB recommended)

- CPU: Broadcom BCM2712, quad-core Cortex-A76 @ 2.4GHz (ARM64)
- RAM: 2GB / 4GB / 8GB / 16GB LPDDR4X (8GB recommended; 2GB insufficient for knowledge graph)
- Storage: microSD (FAT32 boot partition + raw partitions for knowledge graph)
- Peripherals: 40-pin GPIO, 2× I2C, 2× SPI, 6× UART, Gigabit Ethernet, USB 3.0
- PCIe: 2.0 x4 (internal, to RP1 southbridge) + 2.0 x1 (external FPC connector)
- RP1 southbridge: GPIO, I2C, SPI, and peripheral UART are accessed through the RP1 chip connected via PCIe. The debug UART is directly on the BCM2712.

## 9.3 Why Pi 5

The Pi 5's RP1 southbridge makes peripheral access harder than Pi 4's direct MMIO — the AI must reason about PCIe initialization, RP1 register maps, and an additional layer of indirection. This is intentional: **if the AI can bootstrap through PCIe → RP1, it proves the thesis.** Direct MMIO is easy. Reasoning through an intermediary chip is the real test of AI hardware understanding.

The RP1 datasheet is publicly available and Linux kernel drivers exist, giving the LLM good training data to reason from. The debug UART (used by the exokernel) is directly on the BCM2712, not through RP1, so the exokernel's board-specific code remains simple.

Pi 4 remains a viable fallback target if RP1 reasoning proves too difficult for the LLM during MVP.

## 9.4 Memory Budget (8GB)

| Component | Allocation | Notes |
|---|---|---|
| Exokernel + page tables | ~16 MB | Minimal, runs in EL1 |
| Knowledge graph (in-memory) | ~512 MB | Scales with learned knowledge + sensor history |
| AI runtime + task executor | ~64 MB | Async runtime, channels, buffers |
| Network stack (smoltcp) + HTTP | ~32 MB | After AI learns Ethernet. Includes TCP/IP buffers and HTTP request/response buffers (LLM responses can be several KB). TLS memory TBD. |
| Peripheral buffers | ~32 MB | DMA buffers, UART FIFOs |
| **Free** | **~7.4 GB** | Available for larger graphs or future use |

No model weights. No KV cache. The reasoning happens on the remote LLM.

### Back-of-envelope: 72-hour event log

Every register write, sensor reading, and LLM call is logged as a knowledge graph Event (Section 7.2). Estimated event rate during steady-state operation with one sensor polling at 1Hz:

| Source | Rate | Size per event | 72hr total |
|---|---|---|---|
| Sensor readings | 1/sec | ~64 bytes | ~16 MB |
| Register ops (cached replay) | ~2/sec | ~96 bytes | ~50 MB |
| LLM calls (novel intents only) | ~10/day | ~4 KB | ~120 KB |
| Task state changes | ~5/min | ~48 bytes | ~1 MB |
| **Total** | | | **~67 MB** |

With 512 MB allocated, the graph can sustain ~72 hours comfortably before needing compaction or eviction. If multiple sensors poll at higher rates, the AI can reason about downsampling or summarization strategies to manage growth.
