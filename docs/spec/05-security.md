# 6. Security Model

KojinOS MVP is a single-user, single-AI system on a dev board. The security model is scoped accordingly — it protects against bugs and bad LLM output, not adversarial attacks. Hardening against active adversaries is out of scope (Section 10.2).

## 6.1 What Is Protected

**Capability enforcement (EL1 boundary).** The runtime in EL0 never touches MMIO directly. Every hardware operation goes through a syscall to the exokernel in EL1, which validates the capability handle against its internal table before performing the operation. A fabricated or out-of-range capability handle is rejected. This is hardware-enforced by the ARM64 privilege model — EL0 code physically cannot access EL1 memory or MMIO regions.

**DMA safety.** The exokernel allocates DMA buffers and tracks their physical addresses. The runtime cannot instruct the DMA engine to target arbitrary memory — only buffers allocated through `sys_dma_alloc` are valid. (Note: without an IOMMU, a compromised DMA controller could bypass this. This is a known hardware limitation on most SBCs.)

**Validation pipeline (Section 2.4).** Every LLM-generated register sequence is validated before caching: capability ownership check, offset range check, readback validation. Failed validation is logged and not cached. This catches most LLM reasoning errors.

**Safety gates (Section 7.4).** First-time operations on unknown peripherals require human confirmation via UART before execution.

## 6.2 What Is NOT Protected (Accepted Risks for MVP)

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
| Stale cached knowledge after firmware update | A new exokernel binary may change syscall semantics, capability formats, or offset validation rules, but the SD card retains HardwareKnowledge cached by the old version. Replayed sequences could fail in unexpected ways. Mitigated: the schema version field in the partition header (0x39) detects format-level incompatibility; for semantic changes, the runtime should embed a "knowledge compatibility version" in each HardwareKnowledge entity and invalidate cached sequences when it changes. MVP: acceptable to wipe the knowledge partition on firmware updates. Future: selective invalidation based on which syscalls/caps changed. |
