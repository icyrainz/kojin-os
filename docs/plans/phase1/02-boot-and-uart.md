# Task 02: Boot Assembly + UART Output

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Boot the kernel on QEMU, handle EL2→EL1 transition, set up stack, and print "Hello from KojinOS" via PL011 UART.

**Architecture:** `boot.S` (via `global_asm!`) handles AArch64 entry: detects EL2, drops to EL1, sets up stack, zeroes BSS, calls `kernel_main`. PL011 UART driver provides polled character output. `print!`/`println!` macros route through `core::fmt::Write`.

**Tech Stack:** AArch64 assembly, Rust `core::fmt`, PL011 UART at `0x0900_0000`

---

**Files:**
- Modify: `kernel/src/arch/qemu_virt/mod.rs` — add boot assembly + module declarations
- Create: `kernel/src/arch/qemu_virt/uart.rs` — PL011 UART driver
- Create: `kernel/src/print.rs` — print!/println! macros
- Modify: `kernel/src/main.rs` — add print module, update kernel_main and panic handler
- Create: `tests/boot_test.sh` — integration test runner

### Step 1: Write the PL011 UART driver

- [ ] Create `kernel/src/arch/qemu_virt/uart.rs`:

```rust
//! PL011 UART driver for QEMU virt.
//!
//! On QEMU, the UART is pre-initialized by the virtual hardware.
//! No baud rate or clock configuration is needed.

use core::fmt;

/// PL011 register offsets
const UARTDR: usize = 0x00;   // Data Register
const UARTFR: usize = 0x18;   // Flag Register

/// Flag Register bits
const FR_TXFF: u32 = 1 << 5;  // Transmit FIFO full
const FR_RXFE: u32 = 1 << 4;  // Receive FIFO empty

/// PL011 UART instance
pub struct Uart {
    base: usize,
}

impl Uart {
    /// Create a UART instance at the given MMIO base address.
    pub const fn new(base: usize) -> Self {
        Self { base }
    }

    /// Write a single byte, blocking until the TX FIFO has space.
    pub fn putc(&self, byte: u8) {
        unsafe {
            let fr = (self.base + UARTFR) as *const u32;
            let dr = (self.base + UARTDR) as *mut u32;
            while core::ptr::read_volatile(fr) & FR_TXFF != 0 {}
            core::ptr::write_volatile(dr, byte as u32);
        }
    }

    /// Read a single byte, blocking until the RX FIFO has data.
    pub fn getc(&self) -> u8 {
        unsafe {
            let fr = (self.base + UARTFR) as *const u32;
            let dr = (self.base + UARTDR) as *const u32;
            while core::ptr::read_volatile(fr) & FR_RXFE != 0 {}
            core::ptr::read_volatile(dr) as u8
        }
    }

    /// Write a string, converting `\n` to `\r\n`.
    pub fn puts(&self, s: &str) {
        for byte in s.bytes() {
            if byte == b'\n' {
                self.putc(b'\r');
            }
            self.putc(byte);
        }
    }
}

impl fmt::Write for Uart {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.puts(s);
        Ok(())
    }
}

/// QEMU virt PL011 base address
pub const QEMU_UART_BASE: usize = 0x0900_0000;
```

### Step 2: Write the print macros

- [ ] Create `kernel/src/print.rs`:

```rust
//! print!/println! macros backed by the platform UART.

use core::fmt;
use crate::arch::qemu_virt::uart::{Uart, QEMU_UART_BASE};

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => {
        $crate::print::_print(format_args!($($arg)*))
    };
}

#[macro_export]
macro_rules! println {
    () => { $crate::print!("\n") };
    ($($arg:tt)*) => {
        $crate::print!("{}\n", format_args!($($arg)*))
    };
}

#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use fmt::Write;
    let mut uart = Uart::new(QEMU_UART_BASE);
    uart.write_fmt(args).unwrap();
}
```

### Step 3: Write the boot assembly

- [ ] Update `kernel/src/arch/qemu_virt/mod.rs`:

```rust
pub mod uart;

use core::arch::global_asm;

global_asm!(include_str!("boot.S"));
```

- [ ] Create `kernel/src/arch/qemu_virt/boot.S`:

```asm
// KojinOS boot stub for QEMU virt (aarch64)
//
// Entry conditions (QEMU -M virt,virtualization=on):
//   - CPU at EL2
//   - x0 = DTB physical address
//   - MMU off, caches off
//
// This code:
//   1. Drops from EL2 to EL1
//   2. Sets up the kernel stack
//   3. Zeroes BSS
//   4. Calls kernel_main(dtb_addr)

.section .text._start
.global _start

_start:
    // ── Park secondary cores ──
    mrs     x1, mpidr_el1
    and     x1, x1, #0xFF          // Aff0 = core ID
    cbnz    x1, .Lpark             // Only core 0 continues

    // ── Save DTB pointer (x0 from QEMU) ──
    mov     x19, x0

    // ── Mask all interrupts ──
    msr     daifset, #0xF

    // ── Check exception level ──
    mrs     x0, CurrentEL
    lsr     x0, x0, #2
    cmp     x0, #2
    b.ne    .Lat_el1

    // ── EL2 → EL1 transition ──

    // EL1 uses AArch64
    mov     x0, #(1 << 31)         // HCR_EL2.RW = 1
    msr     hcr_el2, x0

    // Make generic timer accessible from EL1
    mrs     x0, cnthctl_el2
    orr     x0, x0, #3             // EL1PCEN | EL1PCTEN
    msr     cnthctl_el2, x0
    msr     cntvoff_el2, xzr

    // Return to EL1h with DAIF masked
    mov     x0, #0x3C5             // M[3:0]=0101 (EL1h), DAIF=1111
    msr     spsr_el2, x0

    adr     x0, .Lat_el1
    msr     elr_el2, x0

    eret

.Lat_el1:
    // ── Now at EL1 ──

    // Set up kernel stack
    ldr     x0, =__stack_top
    mov     sp, x0

    // Zero BSS section
    ldr     x0, =__bss_start
    ldr     x1, =__bss_end
.Lzero_bss:
    cmp     x0, x1
    b.ge    .Lbss_done
    stp     xzr, xzr, [x0], #16
    b       .Lzero_bss
.Lbss_done:

    // Call kernel_main(dtb_addr: usize) -> !
    mov     x0, x19
    bl      kernel_main

    // kernel_main should never return; halt if it does
.Lpark:
    wfe
    b       .Lpark
```

### Step 4: Write the semihosting exit helper

- [ ] Add to `kernel/src/arch/qemu_virt/mod.rs` (append after the `global_asm!` line):

```rust
/// Exit QEMU via AArch64 semihosting.
/// Used for integration tests to signal success/failure.
pub fn qemu_exit(code: u32) -> ! {
    #[repr(C)]
    struct SemihostArgs {
        reason: u64,
        subcode: u64,
    }

    let args = SemihostArgs {
        reason: 0x20026, // ADP_Stopped_ApplicationExit
        subcode: code as u64,
    };

    unsafe {
        core::arch::asm!(
            "mov x0, #0x18",   // SYS_EXIT
            "mov x1, {0}",
            "hlt #0xF000",
            in(reg) &args as *const _ as u64,
            options(noreturn)
        );
    }
}
```

### Step 5: Update main.rs

- [ ] Replace `kernel/src/main.rs` with:

```rust
#![no_std]
#![no_main]

mod arch;
mod print;

use core::panic::PanicInfo;

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("PANIC: {}", info);
    loop {
        unsafe { core::arch::asm!("wfe") };
    }
}

#[no_mangle]
pub extern "C" fn kernel_main(dtb_addr: usize) -> ! {
    println!("[kojin] Hello from KojinOS!");
    println!("[kojin] DTB at: {:#x}", dtb_addr);
    println!("[kojin] Running at EL1 on QEMU virt (aarch64)");

    // Exit QEMU cleanly for testing
    arch::qemu_virt::qemu_exit(0);
}
```

### Step 6: Verify it boots on QEMU

- [ ] Build and run:

```bash
cd kernel && cargo run
```

Expected output:
```
[kojin] Hello from KojinOS!
[kojin] DTB at: 0x4xxx...
[kojin] Running at EL1 on QEMU virt (aarch64)
```

QEMU should exit cleanly (exit code 0) thanks to semihosting.

### Step 7: Create integration test script

- [ ] Create `tests/boot_test.sh`:

```bash
#!/usr/bin/env bash
# Integration test: build kernel, boot on QEMU, check for expected output
set -euo pipefail

EXPECTED="${1:?Usage: boot_test.sh <expected-string> [timeout]}"
TIMEOUT="${2:-5}"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
REPO_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"

cd "$REPO_ROOT/kernel"

# Build
cargo build 2>&1

KERNEL="target/aarch64-unknown-none/debug/kojin-kernel"

# Run QEMU with timeout
OUTPUT=$(timeout "$TIMEOUT" "$REPO_ROOT/tools/qemu-run.sh" "$KERNEL" 2>&1 || true)

echo "$OUTPUT"
echo "---"

if echo "$OUTPUT" | grep -qF "$EXPECTED"; then
    echo "PASS: Found '$EXPECTED'"
    exit 0
else
    echo "FAIL: Expected '$EXPECTED' in output"
    exit 1
fi
```

```bash
chmod +x tests/boot_test.sh
```

### Step 8: Run the integration test

- [ ] Run:

```bash
tests/boot_test.sh "Hello from KojinOS"
```

Expected: `PASS: Found 'Hello from KojinOS'`

### Step 9: Commit

- [ ] Commit:

```bash
git add kernel/src/ tests/
git commit -m "feat(phase1): boot on QEMU virt with PL011 UART output

Bare-metal AArch64 boot: EL2→EL1 drop, stack setup, BSS zeroing.
PL011 UART driver with print!/println! macros. Semihosting exit
for clean QEMU test runs."
```
