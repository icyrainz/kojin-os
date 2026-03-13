# 3. The Bootstrap: Cutting the Umbilical Cord

The AI boots with only UART as its connection to the outside world. A bridge device provides two services over this channel until the AI can provision them locally.

## 3.1 The Bridge Device

A simple device (laptop, ESP32, Raspberry Pi Zero — anything that can proxy UART to the internet) running a serial-to-API bridge. The bridge provides exactly two services:

| Service | Purpose |
|---|---|
| **LLM access** | Forwards AI reasoning requests to an LLM API endpoint, returns responses |
| **Storage** | Key-value persistence on the host until AI learns SD access |

The bridge is intentionally dumb. It has no intelligence — it just shuttles bytes. All reasoning happens on the AI side.

### Bridge Wire Protocol

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

## 3.2 The Bootstrap Sequence

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

**Bootstrap recording.** Every successful bootstrap conversation (all LLM prompts, responses, executed register sequences, and validation results) is saved as a complete replay fixture. This serves three purposes: (1) if the same hardware needs re-bootstrapping, the recorded sequence can be replayed without LLM round-trips, (2) recordings become regression test fixtures for the Hardware Reasoner, and (3) they provide ground-truth training data for future local models. The bridge stores the recording during first bootstrap; after SD is available, it's persisted as a HardwareKnowledge entity in the knowledge graph.

**Bootstrap error handling:**
- **LLM reasoning failure:** If the validation pipeline (Section 2.4) rejects an LLM response, the runtime re-prompts with the error context (e.g., "readback at offset 0x04 returned 0x00, expected 0x00200000"). Up to 3 retries per reasoning step. After 3 failures, log the error and report on UART.
- **Bridge timeout:** If no LLM response within 30 seconds, retransmit the request. After 3 retransmits, report on UART and wait for human intervention.
- **Total bootstrap timeout:** If bootstrap is not complete within 10 minutes, halt and report status on UART. The human can restart or debug.
- **Fallback:** If the bridge is absent and cached knowledge exists, skip bootstrap and cold-boot from SD. If no bridge and no cached knowledge, the system cannot boot — report on UART.

## 3.3 Subsequent Cold Boots

After the first bootstrap, the AI is self-sustaining:

1. Exokernel boots → initializes MMU, UART, capabilities
2. Exokernel reads knowledge graph from SD via `sys_sd_read_block` (PIO mode, no AI reasoning needed)
3. Passes knowledge graph + capabilities + device tree to runtime
4. Runtime loads knowledge → recalls Ethernet register sequence → brings up network → reaches LLM
5. Resumes all persistent intents
6. Sends "I'm awake" on UART

The bridge is only needed for the very first boot on new hardware, or as a fallback if Ethernet fails.

## 3.4 SD Card Partition Layout

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

## 3.5 Hardware Portability

To run KojinOS on a different ARM64 board:

1. Add a new directory under `kernel/src/arch/` with a `Platform` trait implementation (~300–500 lines)
2. Implement: boot assembly, UART, MMU, interrupt controller, storage PIO read, device discovery
3. Provide the correct device tree (or ACPI tables for x86_64)
4. Connect a bridge device
5. Boot — the AI reasons about the new hardware from scratch
6. Once bootstrapped, disconnect the bridge

The `Platform` trait (Section 2.1) is the formal swap boundary. Everything above `syscall.rs` — the entire AI runtime — never changes. Everything below `Platform` — the board-specific implementations — is isolated per target. The AI doesn't have "drivers." It has **understanding**.

### Upgrade Path: ARM64 → x86_64

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
