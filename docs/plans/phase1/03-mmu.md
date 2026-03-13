# Task 03: MMU Setup

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Configure the AArch64 MMU with identity mapping (TTBR0) for boot and a high-half mapping (TTBR1) at `0xFFFF_0000_0000_0000`. Verify both mappings work. The kernel continues to execute in identity-mapped space for Phase 1 — full relocation to high-half is Phase 2.

**Architecture:** Level 1 block descriptors (1GB granularity) with 4KB translation granule. Two page table roots: TTBR0 (identity, 2 entries — device + RAM) and TTBR1 (kernel virtual, 2 entries — same physical regions at high addresses). MAIR configures Device-nGnRnE (index 0) and Normal Write-Back (index 1) memory attributes.

**Tech Stack:** AArch64 MMU (ARMv8-A), Rust `core::ptr`

---

**Files:**
- Create: `kernel/src/arch/qemu_virt/mmu.rs`
- Modify: `kernel/src/arch/qemu_virt/mod.rs` — add `pub mod mmu;`
- Modify: `kernel/src/main.rs` — call `mmu::init()`, verify high-half access

## AArch64 MMU Reference

### Translation Regime (T0SZ=25, 4KB granule)

With T0SZ=25, virtual addresses are 39 bits. Translation starts at Level 1:
- Level 1: 512 entries × 1GB each = 512GB coverage
- Each L1 entry can be a 1GB block descriptor (no further levels needed)

### MAIR_EL1 Attributes

| Index | Type | Value | Usage |
|-------|------|-------|-------|
| 0 | Device-nGnRnE | `0x00` | MMIO (UART, GIC, etc.) |
| 1 | Normal WB-RWA | `0xFF` | RAM (code, data, stack) |

### Level 1 Block Descriptor Layout

```
Bit  0     : Valid (1)
Bit  1     : Block (0) vs Table (1)
Bits 4:2   : AttrIndx (MAIR index)
Bit  5     : NS (0 for non-secure)
Bits 7:6   : AP[2:1] — 00=EL1 RW/EL0 none, 01=EL1 RW/EL0 RW
Bits 9:8   : SH — 00=non, 10=outer, 11=inner shareable
Bit  10    : AF (access flag, must be 1)
Bit  11    : nG (not global, 0=global)
Bits 47:30 : Output address (1GB aligned)
Bit  53    : PXN (privileged execute-never)
Bit  54    : UXN (unprivileged execute-never)
```

### Memory Map

| TTBR0 L1 Index | Physical Range | Attributes | Purpose |
|----------------|---------------|------------|---------|
| 0 | 0x00000000–0x3FFFFFFF | Device-nGnRnE, EL1 RW, XN | UART (0x09000000), GIC |
| 1 | 0x40000000–0x7FFFFFFF | Normal WB, Inner Shareable, EL1 RW | RAM (kernel, stack, heap) |

| TTBR1 L1 Index | Virtual Range | Physical | Attributes |
|----------------|--------------|----------|------------|
| 0 | 0xFFFF000000000000–0xFFFF00003FFFFFFF | 0x00000000 | Device-nGnRnE, XN |
| 1 | 0xFFFF000040000000–0xFFFF00007FFFFFFF | 0x40000000 | Normal WB |

---

### Step 1: Write the MMU module

- [ ] Create `kernel/src/arch/qemu_virt/mmu.rs`:

```rust
//! AArch64 MMU setup for QEMU virt.
//!
//! Uses Level 1 block descriptors (1GB) with 4KB granule.
//! Sets up identity map (TTBR0) and high-half map (TTBR1).

use core::arch::asm;

// ── Descriptor bits ──

const VALID: u64 = 1 << 0;
const TABLE: u64 = 1 << 1;
// Block descriptor: bit 1 = 0 (VALID set, TABLE clear)

const ATTR_IDX_DEVICE: u64 = 0 << 2; // MAIR index 0
const ATTR_IDX_NORMAL: u64 = 1 << 2; // MAIR index 1

const AP_RW_EL1: u64 = 0b00 << 6;    // EL1 RW, EL0 no access
const AP_RW_ALL: u64 = 0b01 << 6;    // EL1 RW, EL0 RW

const SH_OUTER: u64 = 0b10 << 8;     // Outer Shareable
const SH_INNER: u64 = 0b11 << 8;     // Inner Shareable

const AF: u64 = 1 << 10;             // Access Flag

const PXN: u64 = 1 << 53;            // Privileged eXecute Never
const UXN: u64 = 1 << 54;            // Unprivileged eXecute Never

// ── Pre-built descriptors ──

/// Device memory: non-executable, EL1-only, outer-shareable
const DEVICE_BLOCK: u64 =
    VALID | ATTR_IDX_DEVICE | AP_RW_EL1 | SH_OUTER | AF | PXN | UXN;

/// Normal memory: cacheable, inner-shareable, EL1-only
const NORMAL_BLOCK: u64 =
    VALID | ATTR_IDX_NORMAL | AP_RW_EL1 | SH_INNER | AF;

// ── Physical addresses for 1GB blocks ──

const PHYS_DEVICE_BASE: u64 = 0x0000_0000; // L1[0]: device MMIO
const PHYS_RAM_BASE: u64 = 0x4000_0000;    // L1[1]: RAM

// ── Page table storage ──
// Linker script allocates 4 × 4096 bytes at __page_tables_start.
// We use the first two as TTBR0_L1 and TTBR1_L1.

extern "C" {
    static __page_tables_start: u8;
}

/// Get the physical address of TTBR0 L1 table.
fn ttbr0_l1_addr() -> u64 {
    unsafe { &__page_tables_start as *const u8 as u64 }
}

/// Get the physical address of TTBR1 L1 table.
fn ttbr1_l1_addr() -> u64 {
    ttbr0_l1_addr() + 4096
}

/// Initialize and enable the MMU.
///
/// After this call:
/// - Identity map (TTBR0) covers 0x00000000–0x7FFFFFFF
/// - High map (TTBR1) covers 0xFFFF_0000_0000_0000–0xFFFF_0000_7FFF_FFFF
///   pointing to the same physical regions
/// - Caches are enabled
///
/// # Safety
/// Must be called once, from EL1, before any EL0 transitions.
pub unsafe fn init() {
    // ── Fill TTBR0 L1 (identity map) ──
    let ttbr0 = ttbr0_l1_addr() as *mut u64;
    // Zero all 512 entries first
    for i in 0..512 {
        core::ptr::write_volatile(ttbr0.add(i), 0);
    }
    // L1[0] = device (0x00000000, 1GB block)
    core::ptr::write_volatile(ttbr0.add(0), PHYS_DEVICE_BASE | DEVICE_BLOCK);
    // L1[1] = RAM (0x40000000, 1GB block)
    core::ptr::write_volatile(ttbr0.add(1), PHYS_RAM_BASE | NORMAL_BLOCK);

    // ── Fill TTBR1 L1 (high-half map) ──
    let ttbr1 = ttbr1_l1_addr() as *mut u64;
    for i in 0..512 {
        core::ptr::write_volatile(ttbr1.add(i), 0);
    }
    // L1[0] = device at high address
    core::ptr::write_volatile(ttbr1.add(0), PHYS_DEVICE_BASE | DEVICE_BLOCK);
    // L1[1] = RAM at high address
    core::ptr::write_volatile(ttbr1.add(1), PHYS_RAM_BASE | NORMAL_BLOCK);

    // ── MAIR_EL1 ──
    // Index 0: Device-nGnRnE (0x00)
    // Index 1: Normal, Inner/Outer Write-Back Read-Allocate Write-Allocate (0xFF)
    let mair: u64 = 0x00_00_00_00_00_00_FF_00;
    asm!("msr mair_el1, {}", in(reg) mair);

    // ── TCR_EL1 ──
    // T0SZ=25  (39-bit VA, TTBR0)    → bits[5:0]
    // T1SZ=25  (39-bit VA, TTBR1)    → bits[21:16]
    // TG0=0b00 (4KB granule, TTBR0)  → bits[15:14]
    // TG1=0b10 (4KB granule, TTBR1)  → bits[31:30]
    // IRGN0=0b01 (WB-WA cacheable)   → bits[9:8]
    // ORGN0=0b01 (WB-WA cacheable)   → bits[11:10]
    // SH0=0b11  (Inner Shareable)     → bits[13:12]
    // IRGN1=0b01                      → bits[25:24]
    // ORGN1=0b01                      → bits[27:26]
    // SH1=0b11                        → bits[29:28]
    // IPS=0b010 (40-bit PA, 1TB)      → bits[34:32]
    let tcr: u64 = (25 << 0)         // T0SZ
                 | (25 << 16)        // T1SZ
                 | (0b00 << 14)      // TG0 = 4KB
                 | (0b10u64 << 30)   // TG1 = 4KB
                 | (0b01 << 8)       // IRGN0
                 | (0b01 << 10)      // ORGN0
                 | (0b11 << 12)      // SH0
                 | (0b01 << 24)      // IRGN1
                 | (0b01 << 26)      // ORGN1
                 | (0b11 << 28)      // SH1
                 | (0b010u64 << 32); // IPS = 40-bit
    asm!("msr tcr_el1, {}", in(reg) tcr);

    // ── Load page table base registers ──
    asm!("msr ttbr0_el1, {}", in(reg) ttbr0_l1_addr());
    asm!("msr ttbr1_el1, {}", in(reg) ttbr1_l1_addr());

    // ── Barrier before enabling ──
    asm!("isb");

    // ── Invalidate all TLB entries ──
    asm!("tlbi vmalle1");
    asm!("dsb ish");
    asm!("isb");

    // ── Enable MMU + caches ──
    // SCTLR_EL1: M (bit 0) = MMU enable
    //            C (bit 2) = Data cache enable
    //            I (bit 12) = Instruction cache enable
    let sctlr: u64;
    asm!("mrs {}, sctlr_el1", out(reg) sctlr);
    let sctlr = sctlr | (1 << 0) | (1 << 2) | (1 << 12);
    asm!(
        "msr sctlr_el1, {}",
        "isb",
        in(reg) sctlr,
    );
}

/// Verify the TTBR1 (high-half) mapping works by reading through it.
///
/// Writes a magic value to a known physical address, then reads it
/// through the equivalent TTBR1 virtual address.
///
/// # Safety
/// Must be called after `init()`.
pub unsafe fn verify_high_half() -> bool {
    // Physical address within RAM (0x40000000 region)
    // Use a location in the heap area to avoid clobbering code/data
    let test_phys: *mut u64 = 0x4800_0000 as *mut u64; // somewhere in RAM
    let test_virt: *const u64 = 0xFFFF_0000_4800_0000 as *const u64;

    let magic: u64 = 0xDEAD_BEEF_CAFE_F00D;
    core::ptr::write_volatile(test_phys, magic);

    // Ensure the write is visible
    asm!("dsb ish");

    let readback = core::ptr::read_volatile(test_virt);
    readback == magic
}
```

### Step 2: Register the module

- [ ] Update `kernel/src/arch/qemu_virt/mod.rs` — add after the `global_asm!` line:

```rust
pub mod mmu;
```

(Keep existing `pub mod uart;`, `global_asm!`, and `qemu_exit` function.)

### Step 3: Call MMU init from kernel_main

- [ ] Update `kernel/src/main.rs` `kernel_main` function:

```rust
#[no_mangle]
pub extern "C" fn kernel_main(dtb_addr: usize) -> ! {
    println!("[kojin] Hello from KojinOS!");
    println!("[kojin] DTB at: {:#x}", dtb_addr);
    println!("[kojin] Running at EL1 on QEMU virt (aarch64)");

    // Initialize MMU
    unsafe { arch::qemu_virt::mmu::init() };
    println!("[kojin] MMU enabled (identity map + high-half)");

    // Verify high-half mapping
    let ok = unsafe { arch::qemu_virt::mmu::verify_high_half() };
    if ok {
        println!("[kojin] TTBR1 high-half mapping verified OK");
    } else {
        println!("[kojin] TTBR1 high-half mapping FAILED");
    }

    arch::qemu_virt::qemu_exit(0);
}
```

### Step 4: Boot and verify

- [ ] Run:

```bash
cd kernel && cargo run
```

Expected output:
```
[kojin] Hello from KojinOS!
[kojin] DTB at: 0x4...
[kojin] Running at EL1 on QEMU virt (aarch64)
[kojin] MMU enabled (identity map + high-half)
[kojin] TTBR1 high-half mapping verified OK
```

### Step 5: Run integration test

- [ ] Run:

```bash
tests/boot_test.sh "TTBR1 high-half mapping verified OK"
```

Expected: `PASS`

### Step 6: Commit

- [ ] Commit:

```bash
git add kernel/src/arch/qemu_virt/mmu.rs kernel/src/
git commit -m "feat(phase1): MMU with identity map + high-half mapping

Level 1 block descriptors (1GB, 4KB granule). TTBR0 identity maps
device MMIO and RAM. TTBR1 maps same regions at 0xFFFF_0000_xxxx_xxxx.
High-half mapping verified via write/readback test."
```

## Notes for Phase 2

- **Full trampoline:** Relink the kernel at virtual address `0xFFFF_0000_4000_0000`. Boot assembly uses PC-relative addressing until MMU is on, then jumps to high-half. Remove TTBR0 identity map entries. All kernel symbols reference virtual addresses.
- **EL0 memory isolation:** Use Level 2 (2MB) descriptors for finer-grained AP permissions. Kernel memory gets AP=00 (EL1 only). Runtime memory gets AP=01 (EL0 accessible). UXN=1 on kernel pages, PXN=1 on EL0 data pages.
