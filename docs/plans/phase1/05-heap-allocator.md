# Task 05: Heap Allocator

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up a global heap allocator using `linked-list-allocator`, enabling `alloc` crate features (`Vec`, `String`, `Box`, etc.) for all subsequent phases.

**Architecture:** A 1MB heap region is defined in the linker script (already done in Task 01). The `linked-list-allocator` crate provides a `#[global_allocator]`. Initialized once in `kernel_main` before the EL0 transition. OOM handler panics.

**Tech Stack:** `linked-list-allocator` crate (no_std), Rust `alloc` crate

---

**Files:**
- Create: `kernel/src/allocator.rs` — global allocator wrapper
- Modify: `kernel/Cargo.toml` — add `linked-list-allocator` dependency
- Modify: `kernel/src/main.rs` — add `extern crate alloc`, init allocator, test allocation

### Step 1: Add the dependency

- [ ] Update `kernel/Cargo.toml` dependencies section:

```toml
[dependencies]
linked-list-allocator = "0.10"
```

### Step 2: Write the allocator module

- [ ] Create `kernel/src/allocator.rs`:

```rust
//! Global heap allocator using linked-list-allocator.

use linked_list_allocator::LockedHeap;

#[global_allocator]
static ALLOCATOR: LockedHeap = LockedHeap::empty();

/// Initialize the heap allocator with the region defined in the linker script.
///
/// # Safety
/// Must be called exactly once, before any heap allocations.
pub unsafe fn init() {
    extern "C" {
        static __heap_start: u8;
        static __heap_end: u8;
    }

    let heap_start = &__heap_start as *const u8 as usize;
    let heap_end = &__heap_end as *const u8 as usize;
    let heap_size = heap_end - heap_start;

    unsafe {
        ALLOCATOR.lock().init(heap_start as *mut u8, heap_size);
    }
}
```

### Step 3: Update main.rs

- [ ] Add at the top of `kernel/src/main.rs` (after the `#![feature(...)]` lines):

```rust
extern crate alloc;
```

- [ ] Add module declaration (alongside existing `mod arch;`, `mod print;`, `mod syscall;`):

```rust
mod allocator;
```

- [ ] In `kernel_main`, add allocator init and a test allocation **before** the EL0 transition section. Insert after the "Exception vectors installed" line and before "Enable EL0 access to RAM":

```rust
    // Initialize heap allocator
    unsafe { allocator::init() };
    println!("[kojin] Heap allocator initialized (1MB)");

    // Test allocation
    {
        use alloc::vec;
        let v = vec![1u32, 2, 3, 4, 5];
        println!("[kojin] Heap test: vec sum = {}", v.iter().sum::<u32>());
    }
```

### Step 4: Build and test

- [ ] Run:

```bash
cd kernel && cargo run
```

Expected output (relevant lines):
```
[kojin] Heap allocator initialized (1MB)
[kojin] Heap test: vec sum = 15
...
[kojin] EL0 handshake OK
```

### Step 5: Run integration test

- [ ] Run:

```bash
tests/boot_test.sh "Heap test: vec sum = 15"
```

Expected: `PASS`

### Step 6: Commit

- [ ] Commit:

```bash
git add kernel/src/allocator.rs kernel/Cargo.toml kernel/src/main.rs
git commit -m "feat(phase1): heap allocator with linked-list-allocator

1MB heap region from linker script. Global allocator enables
alloc crate (Vec, String, Box) for all subsequent phases.
Verified with test allocation."
```

## Phase 1 Complete

At this point, all Phase 1 milestones are achieved:

- [x] Rust cross-compilation for `aarch64-unknown-none`
- [x] Boot on QEMU `virt` with PL011 UART output
- [x] MMU with identity map + high-half mapping
- [x] **EL1→EL0 handshake: EL0 issues `SVC #0`, EL1 prints "EL0 handshake OK"**
- [x] Heap allocator (`alloc` crate enabled)

### Final integration test

Run all tests to verify everything works together:

```bash
tests/boot_test.sh "Hello from KojinOS" && \
tests/boot_test.sh "MMU enabled" && \
tests/boot_test.sh "high-half mapping verified OK" && \
tests/boot_test.sh "Heap test: vec sum = 15" && \
tests/boot_test.sh "EL0 handshake OK"
```

All should pass.
