# 2. Architecture

KojinOS has three layers. This is not a simplification — three layers is the entire system.

## 2.1 Layer 0: The Exokernel — The Irreducible Minimum

The exokernel is the smallest possible foundation that enables AI reasoning. It contains only what the AI literally cannot figure out for itself — protection, memory safety, and privileged CPU operations. Everything else is the AI's domain. It is ~3,000–3,500 lines of Rust + assembly.

The design principle: **if it requires EL1 privilege or enforces safety, it's in the exokernel. If it requires hardware knowledge or intelligence, the AI reasons about it.** This boundary is the syscall interface defined below. The reasoning layer (LLM, local model, rule engine — whatever sits above) is swappable. The exokernel is not.

The exokernel is organized into three tiers with explicit swap boundaries:

**Tier 1 — Platform trait** (the swap interface):

All hardware-specific code implements a `Platform` trait. Switching hardware means providing a new `Platform` implementation — no changes to generic code.

```rust
trait Platform {
    fn uart_read(&self) -> u8;
    fn uart_write(&self, byte: u8);
    fn mmu_init(&self) -> PageTableRoot;
    fn irq_init(&self) -> InterruptController;  // GIC + timer PPI + PCIe MSI demux
    fn irq_enable(&self, irq: u32);
    fn irq_ack(&self) -> u32;
    fn timer_now(&self) -> u64;
    fn timer_set(&self, deadline_ns: u64);
    fn pcie_init(&self) -> Option<PcieState>;   // RC link training (board-specific)
    fn sd_read_block(&self, block: u64, buf: &mut [u8; 512]);
    fn device_info(&self) -> &[u8];             // DTB on ARM64, ACPI on x86_64
    fn core_start(&self, core_id: u8, entry: usize, stack_top: usize);
}
```

**Tier 2 — Board-specific implementations** (~1,500–2,000 lines for Pi 5, in `arch/`; simpler boards may be less):

| File | Pi 5 (ARM64) | x86_64 (Surface) | QEMU virt |
|---|---|---|---|
| `boot.S` | TF-A lands at EL1, PSCI for SMP | UEFI→long mode, ACPI | EL2→EL1, DTB from QEMU |
| `pcie.rs` | BCM2712 RC init + link training | Standard ECAM (no RC init) | N/A (virtio) |
| `uart.rs` | PL011 at `0x107d001000` | COM1 at I/O port `0x3F8` | PL011 (QEMU PL011) |
| `mmu.rs` | ARM64 translation tables | x86_64 4-level page tables | ARM64 (same as Pi 5) |
| `gic.rs` / `apic.rs` | GIC-400 (GICv2 MMIO) | APIC/x2APIC (MSR) | GICv2 (`gic-version=2`) |
| `sd.rs` / `nvme.rs` | SDHCI PIO (on BCM2712) | AHCI/NVMe PIO or UEFI block | virtio-blk |
| `devicetree.rs` / `acpi.rs` | DTB parser | ACPI parser | DTB parser |

**Tier 3 — Architecture-generic** (~1,500 lines, never changes per target):

- **Capability system (`capability.rs`):** Unforgeable tokens granting access to MMIO regions, physical memory, interrupt lines, and DMA buffers. The exokernel issues capabilities; the AI exercises them.
- **DMA support (`dma.rs`):** Contiguous physical page allocation and cache maintenance. DMA is a safety concern — a bad DMA descriptor can corrupt arbitrary memory — so the exokernel provides safe primitives. The AI reasons about the board-specific parts (descriptor formats, controller registers).
- **Syscall interface (`syscall.rs`):** The hard boundary between exokernel and runtime. Calls into `Platform` trait methods for hardware operations. Defined below.

**Pi 5 platform notes:**
- **Boot:** Stock Pi 5 firmware boots the kernel at **EL3** (Secure state). The recommended approach is to ship a custom TF-A BL31 (`armstub8-2712.bin` on FAT partition, using the official TF-A `rpi5` platform target) which performs EL3→EL2→EL1 transition before handing to `boot.S`. This is what the RPi UEFI project uses. The exokernel's `boot.S` then starts at **EL1**. QEMU should use `-M virt,virtualization=on -cpu cortex-a76` for consistency. Secondary cores are brought up via **PSCI `CPU_ON`** (`smc #0`, function ID `0xC4000003`), not spin tables. `core_id` maps to MPIDR `Aff0` (single cluster). Each secondary core enters at EL1 with caches off — a shared secondary entry stub in `boot.S` runs MMU/cache init before calling the Rust entry point. Requires `enable_uart=1` and `pciex4_reset=0` in `config.txt` (the latter tells firmware to leave PCIe initialized, simplifying early RC bring-up).
- **UART:** PL011 at `0x107d001000` on BCM2712 SoC — directly on the SoC, not behind RP1. Clock initialized by GPU firmware; exokernel only configures baud rate divisors. ~80 lines.
- **GIC:** GIC-400 (GICv2). GICD at `0x107fff9000`, GICC at `0x107fffa000`. Timer PPI: GIC IRQ 29 (CNTP, EL1 physical timer), configured at `irq_init()` time. **RP1 interrupts** arrive via PCIe MSI → GIC SPI — the exokernel's `irq_init()` must configure both the GIC and the PCIe MSI demux for the runtime to receive RP1 peripheral interrupts (e.g., Ethernet MAC). QEMU configured with `-M virt,gic-version=2` to match.
- **PCIe RC:** The BCM2712 PCIe Root Complex requires board-specific initialization (PERST# deassert, link training wait, RC BAR/window setup — ~200–400 lines, similar to Linux's `pcie-brcmstb.c`) **before** any ECAM enumeration works. This runs in the exokernel's `pcie_init()`, not in the AI runtime. Without it, ECAM reads return `0xFFFFFFFF` ("no device"). ECAM base is at `0x1f00000000` (36-bit address). The `MMIOCap` for PCIe uses 64-bit physical addresses.
- **SDHCI:** EMMC2 controller at `0x1000fff000` on BCM2712 SoC (not behind RP1). PIO init is ~400–600 lines (CMD0→CMD8→ACMD41→CMD2→CMD3→CMD7→CMD17 sequence with clock divider setup). This is the single largest platform component. See Section 3.4 for partition layout and Section 2.1.1 for the future alternative.
- **MMU:** `TTBR0_EL1` identity-maps the boot trampoline (1GB block covering DRAM). `TTBR1_EL1` maps kernel virtual space at `0xFFFF000000000000`. After enabling `SCTLR_EL1.M`, a relative branch to the virtual address removes the identity map. Before `ERET` to EL0, the exokernel sets `SP_EL0` to a valid runtime stack top in EL0-accessible memory.
- **Cache maintenance:** `sys_cache_flush` issues `DC CIVAC` + `DSB ISH`. `sys_cache_invalidate` issues `DC IVAC` (EL1-only on ARMv8.2) + `DSB ISH`. Cortex-A76 is ARMv8.2-A — `DC IVAC` cannot be issued from EL0, which is why these are syscalls.

### The Boundary: Syscall Interface

This is the explicit, stable contract between the exokernel (below) and the AI runtime (above). Everything below this line requires EL1 privilege or enforces safety. Everything above uses intelligence. The reasoning layer is swappable; this interface is not.

```
MMIO Operations (exercise capabilities):
  sys_mmio_read32(cap: MMIOCap, offset: u32) → u32
  sys_mmio_write32(cap: MMIOCap, offset: u32, value: u32)
  sys_mmio_read64(cap: MMIOCap, offset: u32) → u64   # needed for PCIe config space / 64-bit BARs
  sys_mmio_write64(cap: MMIOCap, offset: u32, value: u64)

Memory Operations:
  sys_mem_alloc(pages: usize) → MemCap              # virtual pages for runtime use
  sys_mem_free(cap: MemCap)                          # reclaim virtual pages
  sys_dma_alloc(pages: usize) → DmaCap              # physically contiguous, returns virt + phys addr
  sys_dma_free(cap: DmaCap)

Cache Maintenance (required for DMA correctness):
  sys_cache_flush(cap: DmaCap, offset: usize, len: usize)
  sys_cache_invalidate(cap: DmaCap, offset: usize, len: usize)
  # Requiring the cap (not raw pointers) eliminates table scan and TOCTOU race.
  # Offset+len validated against cap's size in O(1). MemCap cannot be passed
  # where DmaCap is required (type-safe). On ARM64: flush issues DC CIVAC + DSB ISH;
  # invalidate issues DC IVAC + DSB ISH (EL1-only on ARMv8.2).

Interrupt Management:
  sys_irq_register(cap: IRQCap)                      # claim an interrupt line
  sys_irq_ack(cap: IRQCap)                           # acknowledge interrupt, re-enable line
  sys_irq_wait(cap: IRQCap, deadline_ns: u64) → WakeReason  # Interrupt | Timeout
    # Blocks until interrupt fires OR deadline (from sys_timer_now) is reached.
    # Prevents livelock when hardware fails to fire. deadline_ns=0 means no timeout.
  sys_irq_mask(cap: IRQCap)                          # temporarily suppress an interrupt line
  sys_irq_unmask(cap: IRQCap)                        # re-enable a masked interrupt line

Batched MMIO (reduces EL0↔EL1 transitions for multi-register sequences):
  sys_mmio_batch(ops: &[MmioOp], results: &mut [MmioResult]) → Result<usize, (usize, Error)>
    # Execute an array of read/write ops in a single EL1 entry.
    # Security: exokernel copies ops into kernel memory first (prevents
    # cross-core TOCTOU), validates ALL capabilities and offsets in a
    # pre-execution pass, then executes. If any op fails validation,
    # the entire batch is rejected with no hardware side effects.
    # Not atomic w.r.t. interrupts — caller masks IRQs if timing-critical.

Timer (monotonic clock — required for timeouts, polling, deadlines):
  sys_timer_now() → u64                              # monotonic nanoseconds since boot (ARM generic timer)
  sys_timer_set(deadline_ns: u64) → IRQCap           # one-shot timer; returns a dedicated IRQCap for the
                                                     # ARM generic timer PPI, usable with sys_irq_wait.
                                                     # This is an architectural interrupt, not device-tree-derived.

SD Card (PIO read-only — bootstrap persistence):
  sys_sd_read_block(block: u64, buf: &mut [u8; 512]) → Result<(), SdError>
  sys_sd_block_range() → (u64, u64)                    # knowledge partition start/end sectors

UART (the one hardcoded I/O channel):
  sys_uart_read() → u8
  sys_uart_write(byte: u8)

Device Tree:
  sys_devicetree_size() → usize                      # query DTB size before allocating
  sys_devicetree_blob(buf: &mut [u8]) → usize        # copy raw DTB into caller's buffer, return bytes written

Core Management:
  sys_core_start(core_id: u8, entry_addr: usize, stack_top: usize) → Result<(), Error>
    # Wake a parked secondary core. Exokernel validates entry_addr is in
    # EL0-executable range (UXN=0) and stack_top is in EL0-accessible memory.
    # Sets SP_EL0 for the core before releasing it. On Pi 5, uses PSCI CPU_ON.
    # Secondary core runs MMU/cache init stub before jumping to entry_addr.

Dynamic Capability (for PCIe BAR discovery):
  sys_cap_request_mmio(phys_addr: u64, size: u64) → Result<MMIOCap, Error>
    # Request a new MMIOCap for a discovered MMIO region (e.g., PCIe BAR).
    # Exokernel validates the ENTIRE range [phys_addr, phys_addr+size) falls
    # within a declared PCIe window (with overflow check). Returns
    # Error::OutOfRange if phys_addr+size exceeds the window end.
    # Overlapping capabilities are allowed (e.g., full RP1 BAR + sub-region).

Capability Management:
  sys_caps_count() → usize                           # number of capabilities
  sys_cap_get(index: usize) → Capability             # get capability by index (avoids cross-EL slice)
  sys_cap_free(cap: MMIOCap) → Result<(), Error>     # free a dynamically acquired cap (boot caps are irrevocable)
```

### What the exokernel does NOT do

Everything that requires intelligence or hardware-specific knowledge. No GPIO driver. No I2C driver. No SPI driver. No Ethernet driver. No filesystem. No process model. No networking stack. No DMA descriptor formatting. No peripheral initialization sequences. The exokernel provides read-only SD access for bootstrap persistence, but the AI reasons about full SD capabilities (writes, DMA mode) through the LLM. All peripheral interaction uses only the syscalls above.

### Why this boundary

| In the exokernel (safety/privilege) | AI reasons about (intelligence/knowledge) |
|---|---|
| Page table management | Which MMIO registers to read/write |
| Capability token validation | What sequence of register operations configures a peripheral |
| DMA buffer allocation | DMA descriptor format (ring buffers, scatter-gather lists) |
| Cache flush/invalidate | When to flush/invalidate relative to DMA operations |
| Interrupt dispatch from EL1→EL0 | What an interrupt from a specific peripheral means |
| SD card PIO block read (bootstrap) | Full SD driver (writes, DMA mode, performance tuning) |
| UART byte read/write | Protocol framing, message parsing, conversation |
| Device tree binary parsing | Interpreting device tree entries to understand hardware |

### 2.1.1 Future: Removing SD from the Exokernel

SD PIO read is in the exokernel today to solve the bootstrap chicken-and-egg. In the future, this could be eliminated by having the bridge always present at boot (Option B) or by having the AI write a bootstrap recipe to the FAT32 boot partition that the GPU firmware loads alongside the kernel (Option C). Both approaches would shrink the exokernel further and move SD entirely into the AI's reasoning domain. For MVP, the pragmatic choice is SDHCI PIO read in the exokernel — it's standard, portable, and enables fully independent cold boots.

## 2.2 Layer 1: The AI Runtime ("The Mind")

The AI runtime is the entire "userspace." It is a single binary that runs in unprivileged mode (EL0) and holds MMIO capabilities granted by the exokernel. This is where intelligence lives.

### Components

**LLM Client.** Communicates with an external LLM (Claude, OpenAI, Ollama, etc.). During bootstrap: via UART through a bridge device. After the AI learns Ethernet: directly over the network. The LLM is the AI's reasoning engine — the "brain" is remote, the "body" is local.

**Device Tree Parser.** Parses the raw FDT blob received from the exokernel via `sys_devicetree_blob`. Extracts relevant subtrees for Hardware Reasoner prompts (only the peripheral nodes relevant to the current task are sent to the LLM, not the entire DTB). ~200-400 lines.

**Hardware Reasoner.** The core innovation. Described in detail in Section 2.4.

**Knowledge Graph Engine.** The persistent memory system. Stores everything: hardware knowledge (learned register sequences), intents (user goals), sensor data (readings over time), and system state. Backed by a compact on-disk representation. The AI queries this graph as naturally as a human recalls a memory. This replaces the entire concept of files and directories.

**Task Executor.** A lightweight async runtime (`no_std` compatible) that executes the operation DAGs produced by intent decomposition. Each node in the DAG is a hardware operation or a reasoning step. The executor handles concurrency, retries, and watchdog timeouts.

**Intent Decomposer.** Takes natural language from the human and produces an execution plan — a DAG of hardware operations. Powered by the external LLM. For example, "monitor temperature and alert if above 30°C" becomes: reason about I2C bus for BME280 → configure sensor → poll at interval → on threshold, reason about GPIO → drive pin HIGH.

## 2.3 Layer 2: The Interface

- **UART serial console:** Primary human interface. Type natural language. The AI responds. Also serves as the bootstrap channel to the bridge device.
- **GPIO as output:** LEDs, buzzers, relays. The AI's physical expression — configured through reasoning, not drivers.
- **I2C/SPI sensors as input:** Temperature, humidity, accelerometer. The AI's senses — accessed through reasoning about the bus protocol.

## 2.4 The Hardware Reasoner

The Hardware Reasoner is the most novel component in the system and the primary interface between the LLM and physical hardware. It translates intelligence into register operations and validates the results.

### Prompt Structure

Every hardware reasoning request sent to the LLM follows a structured format:

```json
{
  "task": "configure_gpio_output",
  "description": "Set GPIO pin 17 as output and drive HIGH",
  "context": {
    "soc": "BCM2712",
    "peripheral_path": "PCIe → RP1 → GPIO",
    "device_tree_entry": {
      "compatible": "raspberrypi,rp1-gpio",
      "note": "behind RP1 southbridge, requires PCIe init first"
    },
    "pcie_cap": "MMIOCap(PCIe_RC_BASE, 0x4000)",
    "rp1_bar_cap": "MMIOCap(dynamically assigned after PCIe enum)",
    "known_state": { "pcie": "initialized", "rp1": "discovered" }
  }
}
```

Note: addresses are dynamically discovered via PCIe BAR enumeration, not hardcoded. The device tree tells the AI where the PCIe root complex is; the AI reasons about enumeration to discover RP1's BAR addresses at runtime.

The context includes only the relevant device tree subtree for the current task, not the entire DTB. For multi-step operations (e.g., Ethernet bring-up), the runtime maintains a conversation with the LLM, feeding back results from each step.

### Response Schema

The LLM must respond with a structured JSON containing executable operations and built-in validation:

```json
{
  "operations": [
    {
      "op": "write",
      "cap": "rp1_gpio_cap",
      "offset": "0x04",
      "value": "0x00200000",
      "description": "RP1 GPIO FSEL1: set pin 17 bits [23:21] = 001 (output)"
    },
    {
      "op": "write",
      "cap": "rp1_gpio_cap",
      "offset": "0x1C",
      "value": "0x00020000",
      "description": "RP1 GPIO SET0: drive pin 17 HIGH"
    }
  ],
  "validate": {
    "op": "read",
    "cap": "rp1_gpio_cap",
    "offset": "0x34",
    "mask": "0x00020000",
    "expected": "0x00020000",
    "description": "RP1 GPIO LEV0: confirm pin 17 reads HIGH"
  },
  "reasoning": "RP1 GPIO uses per-pin control registers (not BCM283x GPFSEL layout). Pin 17 is in io_bank0. GPIO_CTRL register at pin offset sets FUNCSEL and OEOVER fields. Note: register offsets and values shown here are illustrative — the actual values are determined by the LLM reasoning from the RP1 datasheet at execution time. Capability was obtained during RP1 discovery via PCIe BAR enumeration."
}
```

### Execution Pipeline

The runtime processes LLM responses through a strict pipeline:

```
LLM response
  │
  ▼
1. Parse JSON — reject if malformed
  │
  ▼
2. Validate capabilities — every operation must reference
   a capability the runtime actually holds, and every
   offset must be within the capability's range.
   Reject if any operation targets an unowned region.
  │
  ▼
3. Safety gate — if this is a first-time operation on
   this peripheral, show the plan to the human via UART
   and wait for approval (see Section 7.4)
  │
  ▼
4. Execute — issue sys_mmio_write / sys_mmio_read
   via exokernel syscalls, in order
  │
  ▼
5. Validate — execute the validate step, compare
   readback against expected value with mask.
   If mismatch: log error, do NOT cache, report
   to LLM for re-reasoning.
  │
  ▼
6. Cache — store validated operation sequence in
   knowledge graph as HardwareKnowledge entity.
   Future identical operations replay from cache
   without LLM round-trip.
  │
  ▼
7. Log — record all operations as Event entities
   in the knowledge graph (Section 7.2)
```

### Multi-step Conversations

Complex hardware bring-up (e.g., Ethernet) requires multiple LLM round-trips as a conversation:

1. AI: "I need to bring up Ethernet. It's behind RP1 on PCIe. Here's the device tree for the PCIe controller and RP1."
2. LLM: "First, reset the MAC. Here are the operations..." → AI executes, validates, reports results
3. LLM: "Now configure DMA rings. You'll need DMA buffers. Allocate N pages..." → AI calls `sys_dma_alloc`, reports physical addresses
4. LLM: "Write these ring descriptors with your physical addresses..." → AI executes
5. LLM: "Enable the MAC and check link status..." → AI executes, reports link state
6. ...until Ethernet is operational

Each step's results feed into the next prompt. The full conversation is logged and the final validated sequence is cached as a single HardwareKnowledge entity for replay on subsequent boots.

### LLM Requirements

The Hardware Reasoner requires a frontier-class LLM with strong knowledge of:
- ARM peripheral programming and register-level hardware interfaces
- BCM2712 (and target SoC) datasheet content
- Embedded protocols (I2C, SPI, UART, Ethernet MAC/PHY)
- DMA descriptor formats and cache coherency requirements

Models like Claude Opus/Sonnet or GPT-4 class are expected to work. Smaller models will likely produce incorrect register sequences. The validation step catches errors, but a model that fails frequently will make bootstrap impractically slow.

### Token Cost Estimate

A full first-boot bootstrap (Ethernet + SD + GPIO discovery) is estimated at:
- ~10-20 LLM round-trips
- ~5-10KB of context per prompt (device tree entries, prior results)
- ~2-5KB per response
- **Total: ~100-300K tokens, roughly $1-5 at current API pricing**

This is a one-time cost per hardware platform. After bootstrap, cached knowledge means routine operations execute without LLM calls. Novel intents require new LLM reasoning, but at much smaller scale (single peripheral, not full bootstrap).
