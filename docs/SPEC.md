# KojinOS — 個人

**The Personal Intelligence**

MVP Specification · v0.2.0 · March 2026

*Named for Binz · Goku · K2 · Joko*

---

## 1. Thesis

Every operating system ever built assumes the same thing: the human is the orchestrator. Processes, files, permissions, shells, window managers — every abstraction exists to give a human being legible control over hardware. The AI is always a guest in someone else's house, constrained by abstractions designed for a fundamentally different kind of intelligence.

KojinOS inverts this assumption. The AI is the native intelligence. Hardware is its body. There are no processes, no filesystem hierarchy, no shell, no pre-written hardware drivers. The fundamental primitive is **intent**, and the AI reasons its way from intent to hardware register operations — not through driver code, but through intimate understanding of the hardware itself. Protocol stacks (TCP/IP, TLS) are pre-written software the AI uses as tools; hardware interaction is always reasoned.

This is not AI bolted onto Linux. This is not an agent framework running in userspace. This is what an operating system looks like when you design it from first principles for a mind that thinks in relationships, reasons over registers, and acts through peripherals.

### 1.1 Why Now

Three convergences make this feasible as a solo-dev-with-AI-coders project on a Raspberry Pi:

- Large language models have deep knowledge of hardware — datasheets, register maps, peripheral protocols, timing constraints. The AI doesn't need pre-written drivers; it already understands how hardware works.
- The Raspberry Pi 5 exposes GPIO, I2C, SPI, UART, and Gigabit Ethernet on a board with up to 16GB RAM and a quad-core Cortex-A76. Its RP1 southbridge adds a PCIe-connected peripheral layer — a harder test of AI hardware reasoning than direct MMIO.
- Rust's `no_std` ecosystem has matured to the point where you can write bare-metal ARM64 kernels with memory safety, async runtimes, and real peripheral interaction.

### 1.2 Design Philosophy

**Exokernel heritage.** MIT's exokernel research (Engler et al.) proved that separating protection from abstraction lets applications manage hardware directly. KojinOS takes the exokernel's core insight — the kernel only multiplexes and protects, never abstracts — and replaces the "library OS" with an AI that reasons about hardware.

**Intent over instruction.** Traditional OS: human types command → shell parses → kernel dispatches syscall → hardware acts. KojinOS: human states intent → AI reasons about hardware → AI directly manipulates registers through protected capabilities. No shell. No drivers. No syscalls in the traditional sense.

**The AI learns its body.** There are no pre-written peripheral drivers. The exokernel gives the AI raw MMIO capabilities and a device tree describing what hardware exists. The AI reasons about how to operate each peripheral — reading registers, configuring buses, driving outputs — the same way a human engineer reads a datasheet. Swap the hardware, and the AI adapts.

**Knowledge graph, not filesystem.** Data is stored as entities with semantic relationships, not as bytes at paths. The AI doesn't "open a file" — it recalls knowledge from its persistent graph. The storage medium is an implementation detail the AI manages internally.

**The boot sequence is waking up.** When the Pi powers on, KojinOS loads the AI runtime, and the AI reasons its way to full hardware control. There is no login screen. The AI is simply awake.

---

## 2. Architecture

KojinOS has three layers. This is not a simplification — three layers is the entire system.

### 2.1 Layer 0: The Exokernel — The Irreducible Minimum

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

#### The Boundary: Syscall Interface

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

#### What the exokernel does NOT do

Everything that requires intelligence or hardware-specific knowledge. No GPIO driver. No I2C driver. No SPI driver. No Ethernet driver. No filesystem. No process model. No networking stack. No DMA descriptor formatting. No peripheral initialization sequences. The exokernel provides read-only SD access for bootstrap persistence, but the AI reasons about full SD capabilities (writes, DMA mode) through the LLM. All peripheral interaction uses only the syscalls above.

#### Why this boundary

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

#### 2.1.1 Future: Removing SD from the Exokernel

SD PIO read is in the exokernel today to solve the bootstrap chicken-and-egg. In the future, this could be eliminated by having the bridge always present at boot (Option B) or by having the AI write a bootstrap recipe to the FAT32 boot partition that the GPU firmware loads alongside the kernel (Option C). Both approaches would shrink the exokernel further and move SD entirely into the AI's reasoning domain. For MVP, the pragmatic choice is SDHCI PIO read in the exokernel — it's standard, portable, and enables fully independent cold boots.

### 2.2 Layer 1: The AI Runtime ("The Mind")

The AI runtime is the entire "userspace." It is a single binary that runs in unprivileged mode (EL0) and holds MMIO capabilities granted by the exokernel. This is where intelligence lives.

#### Components

**LLM Client.** Communicates with an external LLM (Claude, OpenAI, Ollama, etc.). During bootstrap: via UART through a bridge device. After the AI learns Ethernet: directly over the network. The LLM is the AI's reasoning engine — the "brain" is remote, the "body" is local.

**Device Tree Parser.** Parses the raw FDT blob received from the exokernel via `sys_devicetree_blob`. Extracts relevant subtrees for Hardware Reasoner prompts (only the peripheral nodes relevant to the current task are sent to the LLM, not the entire DTB). ~200-400 lines.

**Hardware Reasoner.** The core innovation. Described in detail in Section 2.4.

**Knowledge Graph Engine.** The persistent memory system. Stores everything: hardware knowledge (learned register sequences), intents (user goals), sensor data (readings over time), and system state. Backed by a compact on-disk representation. The AI queries this graph as naturally as a human recalls a memory. This replaces the entire concept of files and directories.

**Task Executor.** A lightweight async runtime (`no_std` compatible) that executes the operation DAGs produced by intent decomposition. Each node in the DAG is a hardware operation or a reasoning step. The executor handles concurrency, retries, and watchdog timeouts.

**Intent Decomposer.** Takes natural language from the human and produces an execution plan — a DAG of hardware operations. Powered by the external LLM. For example, "monitor temperature and alert if above 30°C" becomes: reason about I2C bus for BME280 → configure sensor → poll at interval → on threshold, reason about GPIO → drive pin HIGH.

### 2.3 Layer 2: The Interface

- **UART serial console:** Primary human interface. Type natural language. The AI responds. Also serves as the bootstrap channel to the bridge device.
- **GPIO as output:** LEDs, buzzers, relays. The AI's physical expression — configured through reasoning, not drivers.
- **I2C/SPI sensors as input:** Temperature, humidity, accelerometer. The AI's senses — accessed through reasoning about the bus protocol.

### 2.4 The Hardware Reasoner

The Hardware Reasoner is the most novel component in the system and the primary interface between the LLM and physical hardware. It translates intelligence into register operations and validates the results.

#### Prompt Structure

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

#### Response Schema

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

#### Execution Pipeline

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

#### Multi-step Conversations

Complex hardware bring-up (e.g., Ethernet) requires multiple LLM round-trips as a conversation:

1. AI: "I need to bring up Ethernet. It's behind RP1 on PCIe. Here's the device tree for the PCIe controller and RP1."
2. LLM: "First, reset the MAC. Here are the operations..." → AI executes, validates, reports results
3. LLM: "Now configure DMA rings. You'll need DMA buffers. Allocate N pages..." → AI calls `sys_dma_alloc`, reports physical addresses
4. LLM: "Write these ring descriptors with your physical addresses..." → AI executes
5. LLM: "Enable the MAC and check link status..." → AI executes, reports link state
6. ...until Ethernet is operational

Each step's results feed into the next prompt. The full conversation is logged and the final validated sequence is cached as a single HardwareKnowledge entity for replay on subsequent boots.

#### LLM Requirements

The Hardware Reasoner requires a frontier-class LLM with strong knowledge of:
- ARM peripheral programming and register-level hardware interfaces
- BCM2712 (and target SoC) datasheet content
- Embedded protocols (I2C, SPI, UART, Ethernet MAC/PHY)
- DMA descriptor formats and cache coherency requirements

Models like Claude Opus/Sonnet or GPT-4 class are expected to work. Smaller models will likely produce incorrect register sequences. The validation step catches errors, but a model that fails frequently will make bootstrap impractically slow.

#### Token Cost Estimate

A full first-boot bootstrap (Ethernet + SD + GPIO discovery) is estimated at:
- ~10-20 LLM round-trips
- ~5-10KB of context per prompt (device tree entries, prior results)
- ~2-5KB per response
- **Total: ~100-300K tokens, roughly $1-5 at current API pricing**

This is a one-time cost per hardware platform. After bootstrap, cached knowledge means routine operations execute without LLM calls. Novel intents require new LLM reasoning, but at much smaller scale (single peripheral, not full bootstrap).

---

## 3. The Bootstrap: Cutting the Umbilical Cord

The AI boots with only UART as its connection to the outside world. A bridge device provides two services over this channel until the AI can provision them locally.

### 3.1 The Bridge Device

A simple device (laptop, ESP32, Raspberry Pi Zero — anything that can proxy UART to the internet) running a serial-to-API bridge. The bridge provides exactly two services:

| Service | Purpose |
|---|---|
| **LLM access** | Forwards AI reasoning requests to an LLM API endpoint, returns responses |
| **Storage** | Key-value persistence on the host until AI learns SD access |

The bridge is intentionally dumb. It has no intelligence — it just shuttles bytes. All reasoning happens on the AI side.

#### Bridge Wire Protocol

Communication between the AI runtime and bridge uses length-prefixed binary frames over UART at **115200 baud** (standard, universally reliable). **MAX_FRAME_SIZE: 256 KB.** Frames with `Length > 262144` are treated as protocol errors — the receiver sends SYNC and discards. This prevents a malicious or buggy bridge from OOM-killing the runtime via oversized frames.

```
┌──────┬────────┬───────────┬───────┐
│ Type │ Length │  Payload  │  CRC  │
│ 1B   │ 4B LE │  N bytes  │ 2B    │
└──────┴────────┴───────────┴───────┘
```

| Type | Direction | Payload |
|---|---|---|
| `0x01` LLM_REQ | AI → bridge | LLM prompt (JSON) |
| `0x02` LLM_RES | bridge → AI | LLM response (JSON) |
| `0x03` STORE | AI → bridge | key (null-terminated) + value bytes |
| `0x04` LOAD_REQ | AI → bridge | key (null-terminated) |
| `0x05` LOAD_RES | bridge → AI | status byte (0=found, 1=not found) + value bytes |
| `0x06` HUMAN_IN | bridge → AI | raw text from human terminal |
| `0x07` AI_OUT | AI → bridge | text for human terminal |
| `0x08` STORE_ACK | bridge → AI | status byte (0=success, 1=error) |
| `0x09` LLM_ERR | bridge → AI | error_code (1B) + retry_after_ms (4B LE) + message. Codes: 0=rate_limited, 1=server_error, 2=invalid_request, 3=context_too_long |
| `0x00` SYNC | either | *Not a framed message* — 7 raw `0x00` bytes outside the frame format. Type `0x00` is reserved so no valid frame starts with `0x00`, making the SYNC pattern unambiguous. |

**CRC-16** on every frame catches UART corruption. On CRC failure, the receiver discards the frame and sends a SYNC. The sender can detect the SYNC and retransmit. The SYNC pattern (7 consecutive zero bytes) cannot appear in a valid frame — since Type byte `0x00` is reserved for SYNC and never used for valid message types, and the minimum frame overhead is 7 bytes (1 Type + 4 Length + 2 CRC), 7 consecutive zeros cannot span within a well-formed frame. This allows the receiver to re-align after byte loss.

**Bridge detection at startup.** On boot, the runtime sends a SYNC frame and waits for a SYNC-ACK (a SYNC frame echoed back within 500ms). If the bridge responds, framed mode is active and bootstrap proceeds. If no response after 3 retries, the runtime assumes the bridge is absent and falls through to independent boot using cached knowledge from the SD card. This handshake lets the same firmware cold-boot with or without the bridge attached.

**After bootstrap** (bridge disconnected), the runtime switches UART to plain text mode — no framing, just raw bytes in/out for human conversation. The framing protocol is only active while the bridge is present.

### 3.2 The Bootstrap Sequence

```
Power on
  │
  ▼
Exokernel boots
  │ initializes: MMU, UART, capability table from device tree
  │ passes to runtime: device_tree + capabilities + uart_handle
  ▼
AI Runtime starts
  │ has: UART (one channel to the world)
  │ has: device tree (map of available hardware)
  │ has: MMIO capabilities (permission to touch hardware)
  │ needs: LLM (to reason) + storage (to persist)
  ▼
Bridge provides both via UART
  │
  ▼
AI reasons about Ethernet hardware
  │ reads device tree → identifies MAC controller
  │ asks LLM: "I have MMIO at 0x... on BCM2712, how do I bring up Ethernet?"
  │ executes register operations via capabilities
  │ validates: link up, can reach LLM endpoint directly
  │
  ▼
✂️ First cord cut — AI has direct LLM access via Ethernet
  │ Bridge now only provides storage
  │
  ▼
AI reasons about SD card hardware
  │ reads device tree → identifies SDHCI controller
  │ asks LLM: "I have MMIO at 0x... how do I read/write SD blocks?"
  │ executes register operations, validates read/write
  │ migrates stored knowledge from bridge to local SD
  │
  ▼
✂️ Second cord cut — AI has local persistence
  │
  ▼
Bridge disconnected. AI is sovereign.
  UART is now just the human conversation interface.
```

**Bootstrap error handling:**
- **LLM reasoning failure:** If the validation pipeline (Section 2.4) rejects an LLM response, the runtime re-prompts with the error context (e.g., "readback at offset 0x04 returned 0x00, expected 0x00200000"). Up to 3 retries per reasoning step. After 3 failures, log the error and report on UART.
- **Bridge timeout:** If no LLM response within 30 seconds, retransmit the request. After 3 retransmits, report on UART and wait for human intervention.
- **Total bootstrap timeout:** If bootstrap is not complete within 10 minutes, halt and report status on UART. The human can restart or debug.
- **Fallback:** If the bridge is absent and cached knowledge exists, skip bootstrap and cold-boot from SD. If no bridge and no cached knowledge, the system cannot boot — report on UART.

### 3.3 Subsequent Cold Boots

After the first bootstrap, the AI is self-sustaining:

1. Exokernel boots → initializes MMU, UART, capabilities
2. Exokernel reads knowledge graph from SD via `sys_sd_read_block` (PIO mode, no AI reasoning needed)
3. Passes knowledge graph + capabilities + device tree to runtime
4. Runtime loads knowledge → recalls Ethernet register sequence → brings up network → reaches LLM
5. Resumes all persistent intents
6. Sends "I'm awake" on UART

The bridge is only needed for the very first boot on new hardware, or as a fallback if Ethernet fails.

### 3.4 SD Card Partition Layout

```
SD Card
┌──────────────────────────────────────────────┐
│ Partition 1: FAT32 (boot)                    │
│   config.txt (kernel=kojin.img), kojin.img,   │
│   *.dtb                                       │
│   Managed by: GPU firmware (read-only to AI) │
├──────────────────────────────────────────────┤
│ Partition 2: Raw (knowledge)                 │
│   Append-only log + snapshots (MessagePack)  │
│   Managed by: AI runtime (read via exokernel │
│   at boot, full read/write after AI learns   │
│   SDHCI through reasoning)                   │
└──────────────────────────────────────────────┘
```

The exokernel reads the partition table at boot, finds partition 2's start/end sectors, and exposes them via `sys_sd_block_range()`. At boot, the exokernel reads raw blocks from partition 2 using PIO mode and passes the bytes to the runtime — the exokernel parses the 512-byte binary header (magic, CRC-32, slot layout — see format below) to determine which slot is active and how many blocks to load, but does NOT parse the graph contents (MessagePack entities, edges, etc.). The exokernel reads: (1) the header block, (2) validates magic + CRC, (3) reads the active slot's blocks + append log blocks, and (4) passes the raw bytes to the runtime. If the header is corrupt, it passes an empty buffer. The runtime parses and validates the graph contents. After the AI reasons about full SDHCI access (including writes and DMA mode), the runtime takes over SD management directly.

**Knowledge partition header format** (512 bytes at block 0 of partition 2):

| Offset | Size | Field | Description |
|---|---|---|---|
| 0x00 | 4B | Magic | `0x4B4F4A4E` ("KOJN") |
| 0x04 | 1B | Active slot | 0 = slot A, 1 = slot B |
| 0x05 | 4B | Sequence number | Monotonically increasing, used to pick the newer valid slot |
| 0x09 | 8B | Slot A offset | Start sector of snapshot slot A |
| 0x11 | 8B | Slot A size | Size in bytes |
| 0x19 | 8B | Slot B offset | Start sector of snapshot slot B |
| 0x21 | 8B | Slot B size | Size in bytes |
| 0x29 | 8B | Log offset | Start sector of append log |
| 0x31 | 8B | Log tail | Byte offset of next free log position |
| 0x39 | 2B | Schema version | Graph format version (runtime checks for compatibility/migration) |
| 0x3B | 4B | CRC-32 | Over bytes 0x00–0x3A |

On corrupt header (bad magic or CRC), the exokernel passes an empty buffer to the runtime, which bootstraps fresh via the bridge. The `sys_sd_read_block` 512-byte block size is the SDHCI default and works for all SDHC/SDXC cards; non-standard block sizes are not supported.

### 3.5 Hardware Portability

To run KojinOS on a different ARM64 board:

1. Add a new directory under `kernel/src/arch/` with a `Platform` trait implementation (~300–500 lines)
2. Implement: boot assembly, UART, MMU, interrupt controller, storage PIO read, device discovery
3. Provide the correct device tree (or ACPI tables for x86_64)
4. Connect a bridge device
5. Boot — the AI reasons about the new hardware from scratch
6. Once bootstrapped, disconnect the bridge

The `Platform` trait (Section 2.1) is the formal swap boundary. Everything above `syscall.rs` — the entire AI runtime — never changes. Everything below `Platform` — the board-specific implementations — is isolated per target. The AI doesn't have "drivers." It has **understanding**.

#### Upgrade Path: ARM64 → x86_64

The architecture is designed to support an upgrade from Pi 5 to x86_64 hardware (e.g., a Surface laptop). This is a larger step than switching ARM64 boards, but the design isolates the changes:

**What changes:** A new `kernel/src/arch/x86_64/` directory with a `Platform` trait implementation:

| Platform method | Pi 5 (ARM64) | Surface (x86_64) |
|---|---|---|
| Boot | EL2→EL1, DTB from firmware | UEFI→long mode, ACPI from firmware |
| `uart_read/write` | PL011 MMIO | COM1 via I/O port `0x3F8` |
| `mmu_init` | ARM64 translation tables | x86_64 4-level page tables |
| `irq_init/enable/ack` | GIC-400 (GICv2, MMIO) | APIC/x2APIC (MSR) |
| `timer_now/set` | ARM generic timer (CNTVCT_EL0, CNTV_CVAL_EL0) | TSC + APIC timer (RDTSC, MSR) |
| `pcie_init` | BCM2712 RC link training (~300 LOC) | Firmware-initialized; ECAM from MCFG ACPI table |
| `sd_read_block` | SDHCI PIO (BCM2712) | AHCI/NVMe PIO or UEFI block |
| `device_info` | DTB blob | ACPI tables |
| `core_start` | PSCI `CPU_ON` (SMC to EL3) | MP startup via SIPI/SIPI sequence or ACPI `_MAT` |

x86_64 also uses I/O port instructions (`in`/`out`) for some peripherals. The `Platform` trait adds `io_read8`/`io_write8` methods, and `syscall.rs` exposes `sys_io_read8`/`sys_io_write8` — a minor interface addition.

**What stays the same (everything above `syscall.rs`):**
- The entire AI runtime (LLM client, Hardware Reasoner, Knowledge Graph, Intent system)
- The bridge protocol
- All learned knowledge about protocols, sensors, and high-level hardware concepts

**What the AI re-learns:**
- All hardware register sequences (x86_64 peripheral addresses and programming models differ completely)
- PCIe enumeration (similar concept, different base addresses and ECAM location)
- NIC bring-up (Intel/Realtek Ethernet instead of RP1, but the LLM knows both)
- Storage controller (NVMe/AHCI instead of SDHCI)

**Estimated delta:** ~800–1,200 lines of exokernel rewrite. The AI runtime is unchanged — it just gets new capabilities from the x86_64 exokernel and reasons about new hardware through the same LLM. This is the thesis in action: the intelligence layer is hardware-independent.

**Practical note:** x86_64 bare-metal is arguably easier than Pi 5 in some ways (UEFI provides a standardized boot environment, ACPI provides richer hardware descriptions than device tree) and harder in others (more complex memory model, legacy compatibility baggage, IOMMU configuration). The Surface laptop's specific hardware (Intel/Qualcomm WiFi, touchscreen, keyboard) would each become new reasoning challenges for the AI.

---

## 4. The Primitives

Traditional operating systems have six core primitives: processes, threads, files, sockets, signals, and pipes. KojinOS has five entirely different ones.

### 4.1 Intent

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

### 4.2 Capability

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

### 4.3 Entity

The unit of persistent knowledge. Replaces files. An entity is a node in the knowledge graph with a type, attributes, and relationships to other entities.

```
Entity { type: HardwareKnowledge, id: "ethernet_init",
         attrs: { register_sequence: [...], validated: true } }
Entity { type: Sensor, id: "bme280_room",
         attrs: { bus: I2C1, addr: 0x76 } }
Entity { type: Intent, id: "temp_alert",
         attrs: { condition: "temp > 28", action: "gpio17 HIGH" } }
Edge { from: "temp_alert", to: "bme280_room", rel: MONITORS }
```

Entities are persisted to the SD card. The knowledge graph supports semantic queries: "what sensors exist?", "what was the temperature at 3pm?", "how do I initialize Ethernet on this board?"

### 4.4 Task

A lightweight coroutine managed by the async executor. Tasks are spawned by the Intent Decomposer and represent individual hardware operations or reasoning steps within a plan.

Tasks share the AI runtime's address space — no isolation, because they are all part of the same intelligence.

- **Lifecycle:** Spawned → Running → Waiting (on hardware/timer/LLM response) → Complete | Failed
- **Concurrency model:** Cooperative async (Rust futures). Hardware interrupts can wake sleeping tasks.
- **Watchdog:** Tasks that exceed their deadline are cancelled. The AI logs the failure and can reason about why it happened.

#### Multi-core Execution (SMP)

The Pi 5 has four Cortex-A76 cores. The MVP uses them as follows:

| Core | Role |
|---|---|
| Core 0 | Exokernel interrupt handling (GIC target) |
| Core 1 | AI runtime main executor — LLM calls, intent decomposition, reasoning |
| Core 2 | Hardware I/O tasks — sensor polling, GPIO actuation, time-sensitive operations |
| Core 3 | Reserved / available for future use |

This separation is critical because LLM round-trips (200-500ms over network, 1-3 seconds over UART bridge) are blocking from the perspective of cooperative async. By pinning I/O tasks to a dedicated core, sensor polling at 1Hz or 10Hz continues uninterrupted while the LLM is thinking. The two executor instances communicate via channels.

**Synchronization model:** Cross-core channels are lock-free MPSC queues (single consumer per channel). The knowledge graph is single-writer (Core 1 owns mutations; Core 2 sends mutation requests via channel). Cache coherency between cores is automatic via the Cortex-A76 DSU cluster — no explicit cache maintenance needed for shared memory. Atomic operations use `core::sync::atomic` with `Ordering::Release`/`Acquire` for channel state.

For MVP, this can start simpler — single-core async with LLM calls as non-blocking futures. If LLM latency causes missed sensor deadlines, promote to multi-core. The architecture supports both.

### 4.5 Channel

Typed, bounded, async communication between tasks and subsystems.

- Sensor tasks publish readings to channels
- The UART interface publishes user messages to a channel the AI monitors
- The AI publishes responses to a channel the UART interface drains
- The LLM client sends/receives through channels

Channels are in-memory only. Persistent communication happens through the knowledge graph.

---

## 5. The Knowledge Graph

The knowledge graph is not a database the OS uses. It IS the storage layer. There are no files. There are no directories. There is only the graph.

### 5.1 Why Not a Filesystem

A filesystem is an abstraction designed for humans who think in hierarchies. An AI thinks in relationships, embeddings, and semantic proximity. Asking an AI to store data in `/home/user/sensors/temperature/2026-03-12.csv` forces it to translate its native representation into an alien format. KojinOS eliminates this translation entirely.

### 5.2 Schema

The knowledge graph uses a property-graph model with typed nodes and edges:

- **Nodes** have a unique ID, a type, and a bag of typed attributes.
- **Node types include:** `HardwareKnowledge` (learned register sequences), `Sensor`, `Reading`, `Intent`, `Task`, `Event`, `Config`, `NetworkKnowledge`, `SystemKnowledge`
- **Edges** are directed, labeled (`MONITORS`, `TRIGGERS`, `PRODUCED_BY`, `DEPENDS_ON`, `LEARNED_FROM`), and can carry attributes.
- **Temporal indexing:** Every node and edge carries a `created_at` and optionally an `expired_at` timestamp. This enables time-travel queries.

### 5.3 Storage Format

On-disk, the knowledge graph is stored as an append-only log of mutations serialized in MessagePack on the raw SD partition (Section 3.4). Periodically, the AI compacts the log into a snapshot.

- **Crash safety:** Each log entry is a self-contained MessagePack record with a CRC-32 trailer. On recovery, the runtime replays from the last valid snapshot, skipping any trailing entry with a bad CRC (partial write due to power loss). No "fsync" concept — we're writing raw blocks, so crash safety comes from the append-only structure and per-entry checksums.
- **Snapshot double-buffering:** The partition maintains two snapshot slots (A and B) plus the append log. Compaction writes a new snapshot to the inactive slot, then updates a 512-byte header block pointing to the active slot. The header contains a magic number and CRC-32 — the runtime reads both slots at boot and picks the one whose header is valid and has the higher sequence number. This means "atomicity" comes from the CRC check, not from SD hardware guarantees. If power is lost mid-header-write, the CRC will fail and the runtime falls back to the other slot.
- **Cheap writes:** appending is O(1).
- **Time travel:** replaying the log from a snapshot reconstructs any historical state.
- **Minimal SD card wear:** append-only with periodic compaction is friendlier to flash storage than random writes. The AI can reason about wear-leveling strategies if the graph grows large.
- **Partition full:** The runtime monitors free space and alerts the human via UART when the partition exceeds 80% capacity. When full, the runtime stops appending new events (losing crash safety for new data) rather than corrupting the log. The AI can reason about downsampling old sensor readings or pruning stale knowledge to reclaim space.

**Compaction procedure:** Compaction is triggered when the append log exceeds 50% of partition free space (or can be triggered by the AI). Steps: (1) Replay current active snapshot + all log entries into a new in-memory graph state. (2) Serialize the full graph to the INACTIVE snapshot slot. (3) Write a new header block with incremented sequence number and the new slot as active. (4) Reset `Log tail` to 0 (truncate the log). If power is lost during step 2, the old slot remains valid. If power is lost during step 3, the CRC check on the header will fail and the runtime falls back to the old slot. Compaction does not block reads — the in-memory graph continues serving queries during the write. Since the knowledge graph is single-writer (Section 4.4), compaction and log appends are serialized on Core 1 — no concurrent appends can occur during steps 1-4.

### 5.4 Hardware Knowledge

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

### 5.5 Query Interface

The AI queries the graph through a native Rust API exposing traversal operations (neighbors, shortest path, subgraph extraction) and attribute filters. The LLM can also generate graph queries from natural language.

---

## 6. Security Model

KojinOS MVP is a single-user, single-AI system on a dev board. The security model is scoped accordingly — it protects against bugs and bad LLM output, not adversarial attacks. Hardening against active adversaries is out of scope (Section 10.2).

### 6.1 What Is Protected

**Capability enforcement (EL1 boundary).** The runtime in EL0 never touches MMIO directly. Every hardware operation goes through a syscall to the exokernel in EL1, which validates the capability handle against its internal table before performing the operation. A fabricated or out-of-range capability handle is rejected. This is hardware-enforced by the ARM64 privilege model — EL0 code physically cannot access EL1 memory or MMIO regions.

**DMA safety.** The exokernel allocates DMA buffers and tracks their physical addresses. The runtime cannot instruct the DMA engine to target arbitrary memory — only buffers allocated through `sys_dma_alloc` are valid. (Note: without an IOMMU, a compromised DMA controller could bypass this. This is a known hardware limitation on most SBCs.)

**Validation pipeline (Section 2.4).** Every LLM-generated register sequence is validated before caching: capability ownership check, offset range check, readback validation. Failed validation is logged and not cached. This catches most LLM reasoning errors.

**Safety gates (Section 7.4).** First-time operations on unknown peripherals require human confirmation via UART before execution.

### 6.2 What Is NOT Protected (Accepted Risks for MVP)

| Risk | Why accepted |
|---|---|
| Compromised LLM endpoint returns malicious register sequences | Bridge/LLM is trusted by design — the AI's reasoning depends on it. Mitigation: validation pipeline catches incorrect sequences. Future: on-device inference eliminates this. |
| Bridge device is compromised | Physical access required. Single-user system. Future: Ethernet-only mode after first bootstrap eliminates bridge. |
| Valid register writes cause hardware issues | Worst case on a Pi: hung peripheral, reboot required. Cannot brick the board through GPIO/I2C/SPI/Ethernet register writes. Observability (Section 7) provides full audit trail for post-incident analysis. |
| Cached HardwareKnowledge replayed without re-validation | Once a sequence is validated and cached, it's trusted for replay. If the hardware changes (different sensor on I2C bus), the cached sequence may fail. Readback validation on replay catches this. |
| No TLS during UART bridge communication | Physical serial connection. Attacker would need physical access to the wire. After bootstrap, the AI reaches the LLM over Ethernet. MVP uses unencrypted HTTP initially — TLS on bare metal requires significant porting effort (e.g., `embedded-tls` or a stripped `rustls`). TLS is a Phase 4 stretch goal, not a blocker. |
| SD card physical tampering | An attacker with physical SD access can modify cached HardwareKnowledge (CRC-32 is not integrity protection). Tampered sequences replay without LLM round-trip or safety gate. Accepted: physical access to a dev board is game-over. Future: HMAC-SHA256 over snapshots using a boot-derived key. |
| Prompt injection via sensor data | A malicious I2C/SPI device could embed prompt injection payloads in readback values that the AI feeds to the LLM. Mitigated: sensor values are placed in a structured `data:` block in LLM prompts, isolated from instruction text. The validation pipeline (Section 2.4) is the last-resort defense — even if the LLM is tricked, invalid register sequences are rejected. |
| LLM cost amplification | Post-bootstrap, no per-intent LLM call limit exists. A complex intent could trigger many reasoning rounds. Mitigated: the Hardware Reasoner enforces a per-intent maximum of 10 LLM calls; beyond that, it halts and alerts the human via UART. |

---

## 7. Observability

KojinOS gives the AI intimate control over raw hardware. With that comes a hard requirement: **every action the AI takes must be visible, traceable, and queryable.** Observability is not an afterthought — it is a core primitive of the system.

### 7.1 Why Observability Is Non-Negotiable

The AI is writing directly to hardware registers based on LLM reasoning. A wrong register write can hang a peripheral, corrupt a bus, or damage hardware. The human must be able to:

- See what the AI is doing in real time
- Understand why it made a decision (the reasoning chain)
- Replay any sequence of operations after the fact
- Intervene before a dangerous operation executes

### 7.2 The Event Log

Every significant action in the system produces an **Event** entity in the knowledge graph. This is not optional logging — it's structural. The knowledge graph IS the observability system.

Event types:

| Event Type | Captures | Example |
|---|---|---|
| `RegisterWrite` | MMIO address, value, capability used, result | `Write 0x... to RP1 GPIO pin 17 ctrl via MMIOCap(rp1_gpio)` |
| `RegisterRead` | MMIO address, value returned | `Read RP1 GPIO pin 17 status → 0x...` |
| `LLMRequest` | Prompt sent, model, tokens used | `"How do I configure I2C1 on BCM2712?"` |
| `LLMResponse` | Response received, latency | `"Write 0x... to BSC1_C register..."` (347ms) |
| `IntentReceived` | Raw user input | `"monitor temperature every 10s"` |
| `IntentDecomposed` | Generated task DAG | `[ConfigI2C, PollSensor, ThresholdCheck]` |
| `TaskStateChange` | Task ID, old state, new state | `SensorTask: Spawned → Running` |
| `BootstrapProgress` | Cord being cut, success/failure | `Ethernet: link up, LLM reachable directly` |
| `HardwareValidation` | Expected vs actual register readback | `Expected bit 17 HIGH, got HIGH ✓` |
| `Error` | What failed, context, AI's reasoning about the failure | `I2C NAK on address 0x76, bus may not be configured` |

Every event carries a monotonic timestamp and links to related entities (the intent that triggered it, the capability used, the LLM conversation that reasoned about it).

### 7.3 Observability Channels

**UART (always available).** The primary observability channel. Configurable verbosity:

- **Quiet mode:** Only human conversation and errors
- **Normal mode:** Intent lifecycle + hardware operations summary
- **Verbose mode:** Full register-level traces + LLM reasoning chains
- **Raw mode:** Every event, every register write, every byte — for debugging

The human can switch modes at any time by typing a command (e.g., `verbosity: verbose`).

**Network (after Ethernet bootstrap).** Once the AI has Ethernet, it can expose:

- A structured event stream (JSON over TCP) for external monitoring tools
- A query endpoint for the knowledge graph — ask "what happened in the last 5 minutes?" and get a structured answer

**Knowledge graph (always, retroactive).** Since every event is an entity in the graph, the human can ask questions after the fact:

- "What register writes did you do in the last hour?"
- "Show me the reasoning chain for the temperature alert intent"
- "What failed during the Ethernet bootstrap?"

The AI answers these by querying its own event history.

### 7.4 Safety Gates

For operations the AI has never performed before (first-time register writes to a new peripheral), the system can require human confirmation:

```
AI: I'm about to write to RP1 GPIO pin 17 control register (via PCIe BAR capability).
    This will configure GPIO pin 17 as output.
    Reasoning: [link to LLM conversation]
    Shall I proceed? [y/n]
```

The human can approve, deny, or ask for more explanation. This gate can be relaxed for known-good operations (validated register sequences stored in the knowledge graph).

---

## 8. Boot Sequence

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

## 9. Hardware Target

### 9.1 Development: QEMU

Initial exokernel development uses `qemu-system-aarch64 -M virt,gic-version=2` for fast iteration. The `virt` machine provides PL011 UART, GICv2 (matching Pi 5's GIC-400), and virtio devices — ideal for developing MMU, capabilities, interrupt routing, and boot handoff without physical hardware.

**QEMU limitations:** The `virt` machine does not model the Pi 5's BCM2712, RP1 southbridge, or PCIe bus. All RP1/PCIe-dependent functionality (GPIO via RP1, Ethernet, Hardware Reasoner's PCIe enumeration path) can only be tested on real Pi 5 hardware. QEMU development validates the exokernel core, syscall interface, capability system, knowledge graph, and bridge protocol — not board-specific hardware reasoning.

### 9.2 Primary: Raspberry Pi 5 (8GB recommended)

- CPU: Broadcom BCM2712, quad-core Cortex-A76 @ 2.4GHz (ARM64)
- RAM: 2GB / 4GB / 8GB / 16GB LPDDR4X (8GB recommended; 2GB insufficient for knowledge graph)
- Storage: microSD (FAT32 boot partition + raw partitions for knowledge graph)
- Peripherals: 40-pin GPIO, 2× I2C, 2× SPI, 6× UART, Gigabit Ethernet, USB 3.0
- PCIe: 2.0 x4 (internal, to RP1 southbridge) + 2.0 x1 (external FPC connector)
- RP1 southbridge: GPIO, I2C, SPI, and peripheral UART are accessed through the RP1 chip connected via PCIe. The debug UART is directly on the BCM2712.

### 9.3 Why Pi 5

The Pi 5's RP1 southbridge makes peripheral access harder than Pi 4's direct MMIO — the AI must reason about PCIe initialization, RP1 register maps, and an additional layer of indirection. This is intentional: **if the AI can bootstrap through PCIe → RP1, it proves the thesis.** Direct MMIO is easy. Reasoning through an intermediary chip is the real test of AI hardware understanding.

The RP1 datasheet is publicly available and Linux kernel drivers exist, giving the LLM good training data to reason from. The debug UART (used by the exokernel) is directly on the BCM2712, not through RP1, so the exokernel's board-specific code remains simple.

Pi 4 remains a viable fallback target if RP1 reasoning proves too difficult for the LLM during MVP.

### 9.4 Memory Budget (8GB)

| Component | Allocation | Notes |
|---|---|---|
| Exokernel + page tables | ~16 MB | Minimal, runs in EL1 |
| Knowledge graph (in-memory) | ~512 MB | Scales with learned knowledge + sensor history |
| AI runtime + task executor | ~64 MB | Async runtime, channels, buffers |
| Network stack (smoltcp) + HTTP | ~32 MB | After AI learns Ethernet. Includes TCP/IP buffers and HTTP request/response buffers (LLM responses can be several KB). TLS memory TBD. |
| Peripheral buffers | ~32 MB | DMA buffers, UART FIFOs |
| **Free** | **~7.4 GB** | Available for larger graphs or future use |

No model weights. No KV cache. The reasoning happens on the remote LLM.

#### Back-of-envelope: 72-hour event log

Every register write, sensor reading, and LLM call is logged as a knowledge graph Event (Section 7.2). Estimated event rate during steady-state operation with one sensor polling at 1Hz:

| Source | Rate | Size per event | 72hr total |
|---|---|---|---|
| Sensor readings | 1/sec | ~64 bytes | ~16 MB |
| Register ops (cached replay) | ~2/sec | ~96 bytes | ~50 MB |
| LLM calls (novel intents only) | ~10/day | ~4 KB | ~120 KB |
| Task state changes | ~5/min | ~48 bytes | ~1 MB |
| **Total** | | | **~67 MB** |

With 512 MB allocated, the graph can sustain ~72 hours comfortably before needing compaction or eviction. If multiple sensors poll at higher rates, the AI can reason about downsampling or summarization strategies to manage growth.

---

## 10. MVP Scope

The MVP is an existence proof that an AI can bootstrap itself onto raw hardware through reasoning alone, maintain persistent understanding of its body, and fulfill human intents by directly manipulating hardware.

### 10.1 In Scope

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

### 10.2 Out of Scope (Future Work)

- On-device inference (local SLM to reduce LLM dependency)
- Multi-agent support (multiple AI runtimes with capability isolation)
- Audio/visual interface (microphone, speaker, display)
- WiFi (requires firmware blobs — Ethernet first)
- OTA updates
- Security hardening beyond basic capability isolation
- Formal verification of the exokernel

---

## 11. Implementation Roadmap

### Phase 1: Bare-Metal Hello World (Week 1–2)

**Toolchain:** `rustup target add aarch64-unknown-none`, `cargo install cargo-binutils`, QEMU 7.0+.
**QEMU invocation:** `qemu-system-aarch64 -M virt,gic-version=2,virtualization=on -cpu cortex-a76 -nographic -kernel kojin.bin`

- Set up Rust cross-compilation toolchain for `aarch64-unknown-none`
- Boot a minimal kernel on QEMU `virt` that writes to PL011 UART
- Implement basic MMU setup (identity map + kernel virtual map + trampoline)
- **Key milestone:** EL1→EL0 handshake: EL0 issues `svc #0`, EL1 prints "EL0 handshake OK"
- Implement heap allocator (`linked-list-allocator` crate, enables `alloc` for all later phases)
- **Buy Pi 5 hardware now** (8GB, USB-UART adapter, good SD card, 5A USB-C PSU)

### Phase 2: Exokernel Core + Async Executor (Week 3–5)

- Implement device tree parser (extract MMIO regions, IRQs)
- Implement capability system: `MMIOCap`, `MemCap`, `DmaCap`, `IRQCap`
- Build EL0→EL1 syscall interface for capability-gated MMIO access
- Interrupt routing through GIC-400 (GICv2)
- **Async executor:** Single-core `embassy-executor` on Core 1, or minimal bespoke scheduler (~300 LOC). Multi-core is post-MVP.
- Build TF-A `armstub8-2712.bin` for Pi 5 (official TF-A `rpi5` platform target)
- Test on real Pi 5: EL0 runtime can read/write MMIO through capabilities

### Phase 3: Bridge Protocol + LLM Client (Week 6–7)

- Implement UART-multiplexed bridge protocol (binary framing: `LLM_REQ`/`LLM_RES`/`LLM_ERR`, `STORE`/`STORE_ACK`/`LOAD_REQ`/`LOAD_RES`, `HUMAN_IN`/`AI_OUT`, `SYNC`)
- Build bridge device software (simple serial-to-HTTPS proxy)
- Implement LLM client in the runtime (send prompts, parse responses)
- Test: AI can reason through the bridge — ask LLM a question via UART, get response

### Phase 4a: Hardware Reasoner + GPIO Proof (Week 8–10)

**This phase validates the core thesis.** If the LLM can correctly reason through PCIe→RP1→GPIO, the thesis is proven.

- Design entity/edge schema in Rust structs (needed for HardwareKnowledge caching)
- Implement in-memory knowledge graph with query API (get-by-id, neighbors, filter-by-type)
- Implement Hardware Reasoner: device tree → LLM prompt → register operations → execute via capabilities
- Build mock LLM server (pre-recorded JSON responses) for offline testing and regression
- AI reasons about GPIO via RP1: PCIe ECAM enumeration → RP1 BAR discovery → GPIO pin HIGH/LOW (LED visible!)
- Note: `pciex4_reset=0` in `config.txt` lets firmware pre-initialize PCIe, simplifying early RC bring-up

### Phase 4b: Ethernet + First Cord Cut (Week 11–14)

**Highest risk phase.** RP1 Ethernet MAC bring-up through PCIe requires DMA address translation (`0x10_0000_0000` inbound offset on BCM2712). Known from community bare-metal attempts to be the hardest single task.

- AI reasons about Ethernet: MAC bring-up through RP1, DMA ring descriptors, PHY reset via MDIO
- Implement smoltcp integration for TCP/IP
- Build HTTP client on top of smoltcp (~300 LOC: HTTP/1.1 framing, chunked decoding)
- First cord cut: AI reaches LLM directly over Ethernet (HTTP initially; TLS via `embedded-tls` is stretch goal)
- Fallback: if Ethernet proves too hard for LLM reasoning, deploy a local HTTP proxy that strips TLS

### Phase 5: SD Persistence + Second Cord Cut (Week 15–16)

- Implement MessagePack serialization for graph mutations (`rmp` crate, no_std + alloc)
- AI reasons about SD card: SDHCI register operations, block read/write
- Build append-only log writer to SD blocks
- Implement snapshot double-buffering + compaction
- Migrate knowledge from bridge storage to local SD
- Second cord cut: AI is fully independent
- Implement snapshot + log replay for crash recovery
- Test: power cycle → AI cold boots independently

### Phase 6: Intent System + Integration (Week 17–20)

- Implement Intent Decomposer: natural language → LLM → task DAG
- Build three reference intents: sensor polling, threshold alerts, scheduled actions
- AI reasons about I2C to talk to BME280 temperature sensor
- Wire intents to knowledge graph for persistence
- End-to-end test: boot → bootstrap → accept intents via UART → execute → persist → reboot → resume
- Stress test: 72-hour continuous operation

---

## 12. Landscape and Differentiation

| Project | Approach | KojinOS Difference |
|---|---|---|
| Steve OS | AI-native OS with shared agent memory | Still runs on conventional OS. KojinOS eliminates the host OS entirely. |
| AthenaOS | Rust/C++ with swarm agents managing processes | Agents manage traditional OS concepts. KojinOS has no processes to manage. |
| OpenAI OS vision | ChatGPT as app platform | Cloud-dependent, app-store model. KojinOS is hardware-native with intimate register-level control. |
| Agent-OS (TechRxiv) | Microkernel for agent coordination | Multi-agent orchestration at scale. KojinOS is single embodied intelligence that reasons about its own body. |
| llm-baremetal | LLM running on UEFI, no OS | Inference without OS. No persistent state, no hardware reasoning, no intent system. |

KojinOS is unique in that the AI has no pre-written drivers — it reasons about raw hardware through an external LLM, bootstraps itself from a single UART channel, and learns to operate its own body. The exokernel is ~3,000–3,500 lines. Everything above that is intelligence.

---

## 13. Open Questions

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

## 14. Success Criteria

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

## 15. Key References

- [rust-raspberrypi-OS-tutorials](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials) — Bare-metal Rust kernel tutorials for Pi 3/4
- [Exokernel: An OS Architecture for Application-Level Resource Management](https://pdos.csail.mit.edu/6.828/2008/readings/engler95exokernel.pdf) — Engler et al., MIT
- [smoltcp](https://github.com/smoltcp-rs/smoltcp) — `no_std` TCP/IP stack for embedded Rust
- [llm-baremetal](https://github.com/djibydiop/llm-baremetal) — LLM running on UEFI without an OS
- [RusPiRo](https://github.com/RusPiRo/ruspiro-kernel) — Rust bare-metal development for Raspberry Pi
- [OSDev Wiki: Exokernel](https://wiki.osdev.org/Exokernel) — Community reference on exokernel design

---

## 16. Project Structure

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
