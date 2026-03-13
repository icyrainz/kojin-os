# Task 04: Exception Vectors + EL0 Handshake

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up the AArch64 exception vector table, transition from EL1 to EL0, and handle an `SVC #0` from EL0 — printing "EL0 handshake OK". This is the key Phase 1 milestone.

**Architecture:** Exception vector table installed at VBAR_EL1. SVC from EL0 arrives at offset `0x400` (Synchronous from Lower EL, AArch64). The handler saves registers, dispatches to Rust `handle_sync_el0()`, which checks ESR_EL1 for SVC and calls the syscall dispatcher. EL0 code is copied to a dedicated region with EL0-accessible page permissions.

**Tech Stack:** AArch64 exception model, `SVC` instruction, SPSR_EL1/ELR_EL1

---

**Files:**
- Create: `kernel/src/arch/qemu_virt/exceptions.rs` — vector table + exception handlers
- Create: `kernel/src/syscall.rs` — SVC dispatch logic
- Modify: `kernel/src/arch/qemu_virt/mod.rs` — add modules
- Modify: `kernel/src/arch/qemu_virt/mmu.rs` — add EL0 page mapping
- Modify: `kernel/src/main.rs` — add modules, EL0 transition, updated kernel_main

## AArch64 Exception Vector Reference

The vector table has 16 entries, each 0x80 bytes (32 instructions):

| Offset | Source | Type |
|--------|--------|------|
| 0x000 | Current EL, SP_EL0 | Synchronous |
| 0x080 | Current EL, SP_EL0 | IRQ |
| 0x100 | Current EL, SP_EL0 | FIQ |
| 0x180 | Current EL, SP_EL0 | SError |
| 0x200 | Current EL, SP_ELx | Synchronous |
| 0x280 | Current EL, SP_ELx | IRQ |
| 0x300 | Current EL, SP_ELx | FIQ |
| 0x380 | Current EL, SP_ELx | SError |
| **0x400** | **Lower EL, AArch64** | **Synchronous** ← SVC from EL0 |
| 0x480 | Lower EL, AArch64 | IRQ |
| 0x500 | Lower EL, AArch64 | FIQ |
| 0x580 | Lower EL, AArch64 | SError |
| 0x600 | Lower EL, AArch32 | Synchronous |
| 0x680 | Lower EL, AArch32 | IRQ |
| 0x700 | Lower EL, AArch32 | FIQ |
| 0x780 | Lower EL, AArch32 | SError |

### ESR_EL1 (Exception Syndrome Register)

For SVC from AArch64:
- `EC` (bits [31:26]) = `0b010101` (0x15) = SVC instruction execution
- `ISS` (bits [15:0]) = immediate value from `SVC #imm`

---

### Step 1: Write the exception vector table and handlers

- [ ] Create `kernel/src/arch/qemu_virt/exceptions.rs`:

```rust
//! AArch64 exception vector table and handlers for QEMU virt.

use core::arch::global_asm;

/// Exception context saved on the kernel stack when entering from EL0.
#[repr(C)]
pub struct ExceptionContext {
    /// General-purpose registers x0–x30
    pub gpr: [u64; 31],
    /// Exception Link Register (return address)
    pub elr: u64,
    /// Saved Program Status Register
    pub spsr: u64,
    /// Exception Syndrome Register
    pub esr: u64,
}

// ESR_EL1 exception class values
const EC_SVC64: u64 = 0x15; // SVC instruction from AArch64

/// Called from the vector table for Synchronous exceptions from Lower EL (AArch64).
#[no_mangle]
extern "C" fn handle_sync_el0(ctx: &mut ExceptionContext) {
    let ec = (ctx.esr >> 26) & 0x3F;

    match ec {
        EC_SVC64 => {
            let svc_num = (ctx.esr & 0xFFFF) as u16;
            crate::syscall::handle_svc(svc_num, ctx);
        }
        _ => {
            crate::println!(
                "[kojin] Unexpected sync exception from EL0: EC={:#x}, ESR={:#x}, ELR={:#x}",
                ec, ctx.esr, ctx.elr
            );
            // Halt — in the real system we'd kill the task
            loop {
                unsafe { core::arch::asm!("wfe") };
            }
        }
    }
}

/// Called for all unhandled exception vectors (placeholder).
#[no_mangle]
extern "C" fn handle_unhandled_exception() {
    crate::println!("[kojin] Unhandled exception!");
    loop {
        unsafe { core::arch::asm!("wfe") };
    }
}

global_asm!(r#"
// ──────────────────────────────────────────────
// AArch64 Exception Vector Table
// ──────────────────────────────────────────────
//
// Each entry is 0x80 bytes (128 bytes / 32 instructions).
// We only fully handle "Lower EL, AArch64, Synchronous" (offset 0x400).
// All other entries branch to a generic unhandled handler.

.section .text.vectors
.balign 2048
.global __exception_vectors
__exception_vectors:

// ── Current EL, SP_EL0 (not used — we use SP_ELx) ──
.balign 0x80
    b       handle_unhandled_exception  // Sync
.balign 0x80
    b       handle_unhandled_exception  // IRQ
.balign 0x80
    b       handle_unhandled_exception  // FIQ
.balign 0x80
    b       handle_unhandled_exception  // SError

// ── Current EL, SP_ELx ──
.balign 0x80
    b       handle_unhandled_exception  // Sync
.balign 0x80
    b       handle_unhandled_exception  // IRQ
.balign 0x80
    b       handle_unhandled_exception  // FIQ
.balign 0x80
    b       handle_unhandled_exception  // SError

// ── Lower EL, AArch64 ──
.balign 0x80
    b       __el0_sync_entry            // Sync ← SVC lands here
.balign 0x80
    b       handle_unhandled_exception  // IRQ
.balign 0x80
    b       handle_unhandled_exception  // FIQ
.balign 0x80
    b       handle_unhandled_exception  // SError

// ── Lower EL, AArch32 (not supported) ──
.balign 0x80
    b       handle_unhandled_exception  // Sync
.balign 0x80
    b       handle_unhandled_exception  // IRQ
.balign 0x80
    b       handle_unhandled_exception  // FIQ
.balign 0x80
    b       handle_unhandled_exception  // SError

// ──────────────────────────────────────────────
// EL0 Synchronous Exception Entry
// ──────────────────────────────────────────────
// Save all GP registers + ELR + SPSR + ESR, then call Rust handler.
// The saved frame (ExceptionContext) is passed as first argument.

__el0_sync_entry:
    // Allocate stack frame: 31 GPR + ELR + SPSR + ESR = 34 × 8 = 272 bytes
    // Round up to 16-byte alignment: 272 (already aligned)
    sub     sp, sp, #272

    // Save x0–x29 (30 registers, 15 pairs)
    stp     x0,  x1,  [sp, #(0  * 8)]
    stp     x2,  x3,  [sp, #(2  * 8)]
    stp     x4,  x5,  [sp, #(4  * 8)]
    stp     x6,  x7,  [sp, #(6  * 8)]
    stp     x8,  x9,  [sp, #(8  * 8)]
    stp     x10, x11, [sp, #(10 * 8)]
    stp     x12, x13, [sp, #(12 * 8)]
    stp     x14, x15, [sp, #(14 * 8)]
    stp     x16, x17, [sp, #(16 * 8)]
    stp     x18, x19, [sp, #(18 * 8)]
    stp     x20, x21, [sp, #(20 * 8)]
    stp     x22, x23, [sp, #(22 * 8)]
    stp     x24, x25, [sp, #(24 * 8)]
    stp     x26, x27, [sp, #(26 * 8)]
    stp     x28, x29, [sp, #(28 * 8)]

    // Save x30 (LR)
    str     x30,       [sp, #(30 * 8)]

    // Save ELR_EL1, SPSR_EL1, ESR_EL1
    mrs     x0, elr_el1
    mrs     x1, spsr_el1
    mrs     x2, esr_el1
    str     x0,        [sp, #(31 * 8)]   // elr
    str     x1,        [sp, #(32 * 8)]   // spsr
    str     x2,        [sp, #(33 * 8)]   // esr

    // Call Rust handler: handle_sync_el0(&mut ExceptionContext)
    mov     x0, sp
    bl      handle_sync_el0

    // ── Restore and return to EL0 ──

    // Restore ELR_EL1, SPSR_EL1
    ldr     x0,        [sp, #(31 * 8)]
    ldr     x1,        [sp, #(32 * 8)]
    msr     elr_el1, x0
    msr     spsr_el1, x1

    // Restore x0–x29
    ldp     x0,  x1,  [sp, #(0  * 8)]
    ldp     x2,  x3,  [sp, #(2  * 8)]
    ldp     x4,  x5,  [sp, #(4  * 8)]
    ldp     x6,  x7,  [sp, #(6  * 8)]
    ldp     x8,  x9,  [sp, #(8  * 8)]
    ldp     x10, x11, [sp, #(10 * 8)]
    ldp     x12, x13, [sp, #(12 * 8)]
    ldp     x14, x15, [sp, #(14 * 8)]
    ldp     x16, x17, [sp, #(16 * 8)]
    ldp     x18, x19, [sp, #(18 * 8)]
    ldp     x20, x21, [sp, #(20 * 8)]
    ldp     x22, x23, [sp, #(22 * 8)]
    ldp     x24, x25, [sp, #(24 * 8)]
    ldp     x26, x27, [sp, #(26 * 8)]
    ldp     x28, x29, [sp, #(28 * 8)]

    // Restore x30
    ldr     x30,       [sp, #(30 * 8)]

    // Deallocate frame
    add     sp, sp, #272

    // Return to EL0
    eret
"#);

/// Install the exception vector table.
///
/// # Safety
/// Must be called from EL1 before any EL0 transition.
pub unsafe fn init() {
    core::arch::asm!(
        "adr x0, __exception_vectors",
        "msr vbar_el1, x0",
        "isb",
        out("x0") _,
    );
}
```

### Step 2: Write the syscall dispatcher

- [ ] Create `kernel/src/syscall.rs`:

```rust
//! SVC syscall dispatcher.
//!
//! Phase 1: Only SVC #0 (handshake) is implemented.
//! Phase 2 adds capability-gated MMIO, memory, IRQ, and timer syscalls.

use crate::arch::qemu_virt::exceptions::ExceptionContext;

/// Syscall numbers (from spec Section 2.1)
pub const SYS_HANDSHAKE: u16 = 0;

/// Handle an SVC from EL0.
///
/// `svc_num` is the immediate from `SVC #imm`.
/// `ctx` contains the saved register state — x0–x7 are arguments,
/// and the handler can modify x0 to return a value.
pub fn handle_svc(svc_num: u16, ctx: &mut ExceptionContext) {
    match svc_num {
        SYS_HANDSHAKE => {
            crate::println!("[kojin] EL0 handshake OK");
            // Return success in x0
            ctx.gpr[0] = 0;
        }
        _ => {
            crate::println!("[kojin] Unknown syscall: SVC #{}", svc_num);
            // Return error in x0
            ctx.gpr[0] = u64::MAX;
        }
    }
}
```

### Step 3: Add EL0-accessible page mapping

The EL0 code needs to execute from memory with AP=01 (EL0 accessible).

- [ ] Add to `kernel/src/arch/qemu_virt/mmu.rs` — a new public constant and modify `init()`:

Add these constants near the top (after existing constants):

```rust
/// Normal memory: cacheable, inner-shareable, EL0+EL1 RW, executable
const NORMAL_BLOCK_EL0: u64 =
    VALID | ATTR_IDX_NORMAL | AP_RW_ALL | SH_INNER | AF;
```

Add this function after `verify_high_half()`:

```rust
/// Remap the RAM block (L1[1]) in TTBR0 to allow EL0 access.
///
/// In Phase 1, we use a single 1GB block for all of RAM with AP=01
/// (EL0+EL1 accessible). Phase 2 will use finer-grained (2MB) mappings
/// to isolate kernel and runtime memory.
///
/// # Safety
/// Must be called after `init()` and before any EL0 transition.
pub unsafe fn enable_el0_access() {
    let ttbr0 = ttbr0_l1_addr() as *mut u64;
    // Update L1[1] (RAM) to allow EL0 access
    core::ptr::write_volatile(ttbr0.add(1), PHYS_RAM_BASE | NORMAL_BLOCK_EL0);

    // Flush TLB to pick up the new permissions
    asm!("tlbi vmalle1");
    asm!("dsb ish");
    asm!("isb");
}
```

### Step 4: Register the new modules

- [ ] Update `kernel/src/arch/qemu_virt/mod.rs` to include:

```rust
pub mod exceptions;
```

(Add alongside existing `pub mod uart;` and `pub mod mmu;`)

- [ ] Update `kernel/src/main.rs` to include:

```rust
mod syscall;
```

(Add alongside existing `mod arch;` and `mod print;`)

### Step 5: Write the EL0 entry stub

The EL0 code is a small function that issues `SVC #0`. We write it as a naked function that gets copied to the EL0 region.

- [ ] Add to `kernel/src/main.rs` (before `kernel_main`):

```rust
/// EL0 entry point — issues SVC #0 (handshake) then loops.
///
/// This is a standalone function whose bytes will be copied to the
/// EL0-accessible memory region. It must not reference any symbols
/// outside itself (no function calls, no globals).
#[naked]
unsafe extern "C" fn el0_entry() -> ! {
    core::arch::asm!(
        "svc #0",       // Handshake syscall
        "1:",
        "wfe",          // Wait after handshake
        "b 1b",
        options(noreturn)
    );
}
```

### Step 6: Add the EL0 transition to kernel_main

- [ ] Replace `kernel_main` in `kernel/src/main.rs`:

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
        arch::qemu_virt::qemu_exit(1);
    }

    // Install exception vector table
    unsafe { arch::qemu_virt::exceptions::init() };
    println!("[kojin] Exception vectors installed");

    // Enable EL0 access to RAM
    unsafe { arch::qemu_virt::mmu::enable_el0_access() };

    // Copy EL0 code to the dedicated region
    let el0_code = el0_entry as *const u8;
    let el0_dest: *mut u8;
    let el0_stack_top: u64;

    unsafe {
        extern "C" {
            static __el0_region_start: u8;
            static __el0_stack_top: u8;
        }
        el0_dest = &__el0_region_start as *const u8 as *mut u8;
        el0_stack_top = &__el0_stack_top as *const u8 as u64;

        // Copy 64 bytes of EL0 code (more than enough for the stub)
        core::ptr::copy_nonoverlapping(el0_code, el0_dest, 64);

        // Ensure the copy is visible to instruction fetch
        core::arch::asm!(
            "dc cvau, {addr}",  // Clean data cache to PoU
            "dsb ish",
            "ic ivau, {addr}",  // Invalidate instruction cache
            "dsb ish",
            "isb",
            addr = in(reg) el0_dest,
        );
    }

    println!("[kojin] EL0 code at: {:#x}", el0_dest as u64);
    println!("[kojin] Transitioning to EL0...");

    // ── Transition to EL0 ──
    // Set up SPSR_EL1: return to EL0t (M[3:0]=0b0000), no DAIF masking
    // Set ELR_EL1: EL0 entry address
    // Set SP_EL0: EL0 stack top
    unsafe {
        core::arch::asm!(
            "msr sp_el0, {stack}",       // EL0 stack pointer
            "msr elr_el1, {entry}",      // Return-to address
            "mov x0, #0",               // SPSR: EL0t, all interrupts unmasked
            "msr spsr_el1, x0",
            "eret",                      // Jump to EL0!
            entry = in(reg) el0_dest as u64,
            stack = in(reg) el0_stack_top,
            options(noreturn),
        );
    }
}
```

### Step 7: Add #![feature(naked_functions)] to main.rs

- [ ] Add at the top of `kernel/src/main.rs`, after `#![no_main]`:

```rust
#![feature(naked_functions)]
```

### Step 8: Build and test

- [ ] Run:

```bash
cd kernel && cargo build
```

Fix any compilation errors.

- [ ] Run on QEMU:

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
[kojin] Exception vectors installed
[kojin] EL0 code at: 0x40...
[kojin] Transitioning to EL0...
[kojin] EL0 handshake OK
```

**Note:** After the handshake, EL0 enters a `wfe` loop. QEMU won't exit cleanly. For the test, we need to either:
- (a) Have the SVC handler call `qemu_exit(0)` after printing the handshake message, or
- (b) Use the test script with a timeout and grep

Option (a) is cleaner. Update `kernel/src/syscall.rs` `handle_svc`:

```rust
SYS_HANDSHAKE => {
    crate::println!("[kojin] EL0 handshake OK");
    ctx.gpr[0] = 0;
    // Exit QEMU after successful handshake (test only)
    crate::arch::qemu_virt::qemu_exit(0);
}
```

### Step 9: Run integration test

- [ ] Run:

```bash
tests/boot_test.sh "EL0 handshake OK"
```

Expected: `PASS: Found 'EL0 handshake OK'`

### Step 10: Commit

- [ ] Commit:

```bash
git add kernel/src/
git commit -m "feat(phase1): EL1↔EL0 handshake via SVC #0

Install AArch64 exception vector table at VBAR_EL1.
Transition from EL1 to EL0 via ERET. EL0 stub issues SVC #0,
exception handler dispatches to syscall handler which prints
'EL0 handshake OK'. Key Phase 1 milestone achieved."
```
