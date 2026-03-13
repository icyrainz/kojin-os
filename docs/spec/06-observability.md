# 7. Observability

KojinOS gives the AI intimate control over raw hardware. With that comes a hard requirement: **every action the AI takes must be visible, traceable, and queryable.** Observability is not an afterthought — it is a core primitive of the system.

## 7.1 Why Observability Is Non-Negotiable

The AI is writing directly to hardware registers based on LLM reasoning. A wrong register write can hang a peripheral, corrupt a bus, or damage hardware. The human must be able to:

- See what the AI is doing in real time
- Understand why it made a decision (the reasoning chain)
- Replay any sequence of operations after the fact
- Intervene before a dangerous operation executes

## 7.2 The Event Log

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

## 7.3 Observability Channels

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

## 7.4 Safety Gates

For operations the AI has never performed before (first-time register writes to a new peripheral), the system can require human confirmation:

```
AI: I'm about to write to RP1 GPIO pin 17 control register (via PCIe BAR capability).
    This will configure GPIO pin 17 as output.
    Reasoning: [link to LLM conversation]
    Shall I proceed? [y/n]
```

The human can approve, deny, or ask for more explanation. This gate can be relaxed for known-good operations (validated register sequences stored in the knowledge graph).
