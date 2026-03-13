# Phase 1: Bare-Metal Hello World — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Boot a bare-metal Rust kernel on QEMU aarch64 `virt` that prints via PL011 UART, sets up MMU with identity + high-half mappings, transitions from EL1 to EL0, handles SVC syscalls (proving the EL1↔EL0 boundary), and provides a heap allocator.

**Architecture:** Minimal exokernel at EL1 on QEMU `virt` (cortex-a76, GICv2). PL011 UART at `0x0900_0000` for serial I/O. MMU uses Level 1 block descriptors (1GB granularity) for identity map (TTBR0) and device/RAM mapping. EL0 runtime stub issues `SVC #0` to prove the syscall boundary. `linked-list-allocator` provides heap for `alloc` support.

**Tech Stack:** Rust nightly (`no_std`, `no_main`), `aarch64-unknown-none` target, QEMU 7.0+, AArch64 assembly

**QEMU invocation:** `qemu-system-aarch64 -M virt,gic-version=2,virtualization=on -cpu cortex-a76 -nographic -semihosting-config enable=on,target=native -kernel <kernel-elf>`

---

## Spec References

- `docs/spec/01-architecture.md` — Layer 0 architecture, Platform trait, syscall interface
- `docs/spec/02-bootstrap.md` — Bootstrap sequence
- `docs/spec/03-primitives.md` — Capability types
- `docs/spec/08-roadmap.md` — Phase 1 requirements
- `docs/spec/09-appendix.md` — Project structure

## Task Dependency Graph

```
01-project-setup
       │
       ▼
02-boot-and-uart
       │
       ▼
03-mmu
       │
       ▼
04-exceptions-and-el0
       │
       ▼
05-heap-allocator
```

## File Map

| File | Responsibility | Task |
|------|---------------|------|
| `rust-toolchain.toml` | Nightly toolchain + aarch64 target | 01 |
| `kernel/Cargo.toml` | Crate config, dependencies | 01 |
| `kernel/.cargo/config.toml` | Build target, linker, runner | 01 |
| `kernel/src/arch/qemu_virt/linker.ld` | Memory layout (0x40000000 base) | 01 |
| `tools/qemu-run.sh` | QEMU runner script | 01 |
| `kernel/src/main.rs` | Entry point, kernel_main | 02 |
| `kernel/src/arch/mod.rs` | Architecture module re-export | 02 |
| `kernel/src/arch/qemu_virt/mod.rs` | Boot assembly (global_asm), QemuVirt platform | 02 |
| `kernel/src/arch/qemu_virt/uart.rs` | PL011 UART read/write | 02 |
| `kernel/src/print.rs` | `print!`/`println!` macros via `core::fmt` | 02 |
| `tests/boot_test.sh` | QEMU integration test (grep UART output) | 02 |
| `kernel/src/arch/qemu_virt/mmu.rs` | Page tables, MAIR/TCR/TTBR, MMU enable | 03 |
| `kernel/src/arch/qemu_virt/exceptions.rs` | Vector table (global_asm), exception handlers | 04 |
| `kernel/src/syscall.rs` | SVC dispatch logic | 04 |
| `kernel/src/allocator.rs` | `#[global_allocator]` wrapper | 05 |

## Phase 1 Scope Decisions

1. **Everything linked at physical address 0x40000000.** The kernel runs in identity-mapped space. TTBR1 (high-half) mapping is created and verified but the kernel does not relocate. Full high-half relocation is deferred to Phase 2 when proper EL0/EL1 memory isolation is needed.

2. **EL0 code lives in the same binary.** A small `el0_entry` function runs as EL0. In the full system, the runtime is a separate binary — that separation happens in Phase 2.

3. **No capability system yet.** Phase 1 proves the EL1↔EL0 boundary works. Capability-gated syscalls come in Phase 2.

4. **QEMU only.** Pi 5 platform code is Phase 2+.

## Testing Strategy

- **Integration tests:** Boot on QEMU with 5s timeout, capture UART output, check for expected strings
- **Clean exit:** AArch64 semihosting (`HLT #0xF000` with `SYS_EXIT`) for deterministic QEMU exit
- **Test runner:** `tests/boot_test.sh <expected-string>` builds, boots, greps

## Plan Files

1. [01-project-setup.md](./01-project-setup.md) — Rust toolchain, Cargo, linker script, QEMU runner
2. [02-boot-and-uart.md](./02-boot-and-uart.md) — Boot assembly, PL011 UART, "Hello from KojinOS"
3. [03-mmu.md](./03-mmu.md) — Page tables, MMU enable, identity + high-half verification
4. [04-exceptions-and-el0.md](./04-exceptions-and-el0.md) — Exception vector table, EL0 transition, SVC handler
5. [05-heap-allocator.md](./05-heap-allocator.md) — linked-list-allocator, `alloc` crate support
