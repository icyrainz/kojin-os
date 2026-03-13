# Task 01: Project Setup

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up the Rust bare-metal cross-compilation toolchain and build infrastructure so `cargo build` produces an AArch64 ELF that QEMU can load.

**Architecture:** Cargo project under `kernel/` targeting `aarch64-unknown-none`. Nightly Rust with `build-std` for `core` and `alloc`. Custom linker script places code at `0x40000000` (QEMU virt RAM base). QEMU runner script for `cargo run`.

**Tech Stack:** Rust nightly, aarch64-unknown-none, QEMU 7.0+

---

**Files:**
- Create: `rust-toolchain.toml`
- Create: `kernel/Cargo.toml`
- Create: `kernel/.cargo/config.toml`
- Create: `kernel/src/main.rs` (minimal stub)
- Create: `kernel/src/arch/mod.rs` (empty module)
- Create: `kernel/src/arch/qemu_virt/mod.rs` (empty module)
- Create: `kernel/src/arch/qemu_virt/linker.ld`
- Create: `tools/qemu-run.sh`

### Step 1: Create rust-toolchain.toml

- [ ] Create `rust-toolchain.toml` at repo root:

```toml
[toolchain]
channel = "nightly"
components = ["rust-src", "llvm-tools"]
targets = ["aarch64-unknown-none"]
```

### Step 2: Create kernel/Cargo.toml

- [ ] Create `kernel/Cargo.toml`:

```toml
[package]
name = "kojin-kernel"
version = "0.1.0"
edition = "2021"

[dependencies]

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
lto = true
```

### Step 3: Create kernel/.cargo/config.toml

- [ ] Create `kernel/.cargo/config.toml`:

```toml
[build]
target = "aarch64-unknown-none"

[target.aarch64-unknown-none]
rustflags = ["-C", "link-arg=-Tsrc/arch/qemu_virt/linker.ld"]
runner = "../tools/qemu-run.sh"

[unstable]
build-std = ["core", "alloc"]
build-std-features = ["compiler-builtins-mem"]
```

Notes:
- `build-std` rebuilds `core` and `alloc` from source for our target
- `compiler-builtins-mem` provides `memcpy`/`memset`/`memcmp` implementations
- Linker script path is relative to `kernel/` (the Cargo project root)

### Step 4: Create linker script

- [ ] Create `kernel/src/arch/qemu_virt/linker.ld`:

```ld
ENTRY(_start)

SECTIONS
{
    . = 0x40000000;

    .text : {
        KEEP(*(.text._start))
        *(.text .text.*)
    }

    .rodata : ALIGN(4096) {
        *(.rodata .rodata.*)
    }

    .data : ALIGN(4096) {
        *(.data .data.*)
    }

    .bss (NOLOAD) : ALIGN(4096) {
        __bss_start = .;
        *(.bss .bss.*)
        *(COMMON)
        . = ALIGN(16);
        __bss_end = .;
    }

    /* Page tables — 4 tables × 4096 bytes each, 4K-aligned */
    .page_tables (NOLOAD) : ALIGN(4096) {
        __page_tables_start = .;
        . = . + (4 * 4096);
        __page_tables_end = .;
    }

    /* Kernel stack — 64KB */
    .stack (NOLOAD) : ALIGN(4096) {
        __stack_bottom = .;
        . = . + 64K;
        __stack_top = .;
    }

    /* Heap — 1MB for Phase 1 */
    .heap (NOLOAD) : ALIGN(4096) {
        __heap_start = .;
        . = . + 1M;
        __heap_end = .;
    }

    /* EL0 code region — 64KB */
    .el0_text (NOLOAD) : ALIGN(4096) {
        __el0_region_start = .;
        . = . + 64K;
        __el0_region_end = .;
    }

    /* EL0 stack — 64KB */
    .el0_stack (NOLOAD) : ALIGN(4096) {
        __el0_stack_bottom = .;
        . = . + 64K;
        __el0_stack_top = .;
    }

    /DISCARD/ : {
        *(.comment)
        *(.note*)
        *(.eh_frame*)
    }
}
```

Notes:
- Base address 0x40000000 = QEMU virt RAM start
- Page tables statically allocated (avoids chicken-and-egg with allocator)
- EL0 regions defined now, used in Task 04
- Heap region defined now, used in Task 05

### Step 5: Create minimal main.rs stub

- [ ] Create `kernel/src/main.rs`:

```rust
#![no_std]
#![no_main]

mod arch;

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {
        unsafe { core::arch::asm!("wfe") };
    }
}

#[no_mangle]
pub extern "C" fn kernel_main(_dtb_addr: usize) -> ! {
    loop {
        unsafe { core::arch::asm!("wfe") };
    }
}
```

### Step 6: Create arch module files

- [ ] Create `kernel/src/arch/mod.rs`:

```rust
pub mod qemu_virt;
```

- [ ] Create `kernel/src/arch/qemu_virt/mod.rs`:

```rust
// Boot assembly and platform implementation for QEMU virt
// (populated in Task 02)
```

### Step 7: Create QEMU runner script

- [ ] Create `tools/qemu-run.sh` and make executable:

```bash
#!/usr/bin/env bash
set -euo pipefail

KERNEL_ELF="$1"
shift

exec qemu-system-aarch64 \
    -M virt,gic-version=2,virtualization=on \
    -cpu cortex-a76 \
    -m 128M \
    -nographic \
    -semihosting-config enable=on,target=native \
    -kernel "$KERNEL_ELF" \
    "$@"
```

```bash
chmod +x tools/qemu-run.sh
```

### Step 8: Verify build compiles

- [ ] Run from `kernel/`:

```bash
cd kernel && cargo build
```

Expected: Compilation succeeds. The binary won't boot yet (no `_start` symbol — linker will warn but build should complete). If the linker fails because `_start` is missing, that's expected and will be fixed in Task 02.

### Step 9: Commit

- [ ] Commit:

```bash
git add rust-toolchain.toml kernel/ tools/
git commit -m "feat(phase1): scaffold bare-metal Rust project for aarch64

Set up cross-compilation toolchain, Cargo config, linker script,
and QEMU runner for the kojin-kernel exokernel."
```
