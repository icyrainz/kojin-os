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

## Specification Sections

| # | Section | Description |
|---|---|---|
| 2 | [Architecture](spec/01-architecture.md) | Three-layer design: exokernel, AI runtime, interface. Syscall interface. Hardware Reasoner. |
| 3 | [Bootstrap](spec/02-bootstrap.md) | Bridge device, bootstrap sequence, cord-cutting, SD layout, hardware portability. |
| 4 | [Primitives](spec/03-primitives.md) | Intent, Capability, Entity, Task, Channel — the five OS primitives. |
| 5 | [Knowledge Graph](spec/04-knowledge-graph.md) | Schema, storage format, hardware knowledge caching, query interface. |
| 6 | [Security](spec/05-security.md) | Capability enforcement, DMA safety, accepted risks for MVP. |
| 7 | [Observability](spec/06-observability.md) | Event log, observability channels, safety gates. |
| 8–9 | [Hardware](spec/07-hardware.md) | Boot sequence, QEMU dev target, Pi 5 specs, memory budget. |
| 10–11 | [Roadmap](spec/08-roadmap.md) | MVP scope, 6-phase implementation plan (20 weeks). |
| 12–16 | [Appendix](spec/09-appendix.md) | Landscape, open questions, success criteria, references, project structure. |
