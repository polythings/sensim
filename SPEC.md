# sensim — Specification

> **Status:** Draft v0.1 — design, nothing built yet (§19)
> **Last updated:** 2026-10-17
> **Scope of this document:** requirements, features, architecture, and design decisions, with open questions. It is
> the list of everything sensim should do, and where each item stands. Not an implementation guide yet.

### Decision log

| ID | Decision | Status |
|----|----------|--------|
| D1 | Name: **sensim**. Crates are `sensim` and `sensim-<part>` (§5.2). | **Accepted** |
| D2 | Its own public repository, `github.com/polythings/sensim`, separate from Timelake. Timelake's benchmarks can use it, but sensim doesn't depend on Timelake. | **Accepted** |
| D3 | License: **Apache License 2.0**, as for Timelake. | **Accepted** |
| D4 | Rust for everything that runs in the data path. One process simulates many devices: a device is state stepped by a worker, not a process or a thread (§5.3). | **Accepted** |
| D5 | Every sample has three times: true time, device time, and arrival time. sensim always knows the true values, so a checker can tell whether the system under test handled each issue correctly (§10.1, §12). | Proposed |
| D6 | Deterministic: a fleet's seed and configuration fix every device's data and every injected fault. Each device has its own seed derived from the fleet seed and its ID, so one device can be reproduced alone (§7.4). | Proposed |
| D7 | High rates travel as frames (batches of samples), the way real high-rate devices send them, never as one message per sample (§7.3). | Proposed |
| D8 | Behaviour runs on events (a write, a command, a state change, a scenario step), never per sample. Signals are compiled Rust. Behaviour is declarative rules, with a sandboxed script for what rules can't express (§8.3, Q3). | Proposed |
| D9 | Every fault and behaviour is a setting that can be turned on or off or tuned, per fleet, device type, device, or at runtime. Settings are named for the mechanism they change (§15). | Proposed |
| D10 | Device clocks follow a simulation clock that runs in real time by default, and can run faster than real time or as fast as possible to generate history (§5.4). | Proposed |
| D11 | Domain: **sensim.dev** (registered 2026-10-17). Nothing is served there yet. | **Accepted** |

---

## 1. Vision

**sensim simulates a field of devices: thousands of sensors, PLCs, meters, and gateways that produce data, talk real
protocols in both directions, and misbehave the way real devices do.**

It is for testing and benchmarking whatever receives that data — a broker, a pipeline, a historian, a time-series
database, a lake, or a whole IoT stack — continuously, for days or weeks, at realistic rates, with a check at the end
that says what arrived correctly and what didn't.

## 2. Requirements (as stated)

| ID | Requirement |
|----|-------------|
| R1 | Native (Rust, C, or C++), not a script: generate large volumes continuously, at high frequency (up to 8 kHz per channel), for streaming and long-term benchmarking. Not one-off datasets. |
| R2 | Emulate devices, to test entire IoT infrastructure, not only a database. |
| R3 | Run 5,000 or more devices at once, to mimic a real field. |
| R4 | A device layer on top of the generator: devices send data over different protocols. |
| R5 | Devices also receive data over different protocols, respond with stubs, and adjust their behaviour. |
| R6 | Simulate issues on devices. |
| R7 | Simulate late data arrival. |
| R8 | Simulate incorrect device clocks. |
| R9 | Simulate data sent in different batches. |
| R10 | Simulate data sent in different encodings. |
| R11 | Beyond IoT devices: click events and finance events for time-series testing (from the first question that led to sensim). |

## 3. Goals and non-goals

### Goals
- **G1 — Realistic:** signals, devices, and faults look like a real field, including its mix of slow and fast devices
  and its failures.
- **G2 — Fast:** generation is never the bottleneck. The limits are encoding and the network, and sensim reports when
  it falls behind instead of slowing down without saying so.
- **G3 — Reproducible:** the same seed and configuration produce the same data and the same faults (D6).
- **G4 — Verifiable:** the checker compares what reached the system under test with what was sent (§12).
- **G5 — Controllable while running:** behaviour and faults change at runtime and follow scenarios (§11).
- **G6 — Long-running:** runs for weeks with constant memory and no drift in rates.
- **G7 — Modular:** each protocol and each part is its own crate, usable as a library without the CLI.

### Non-goals (for now)
- A broker load tester for millions of idle connections. Tools for that exist (§17).
- A digital twin platform or device registry. sensim plays the devices; it doesn't manage real ones.
- Hard real-time timing (microsecond-accurate sample clocks, real-time OS). Timing is accurate to what a normal OS and
  network allow, and sensim measures how far it is off.
- Synthetic data learned from real data (GANs, diffusion models). Signals are explicit models.
- A graphical interface.

## 4. Concepts

| Term | Meaning |
|------|---------|
| **Field** | Everything sensim simulates in one run: all devices, their links, and the scenario. |
| **Device type** | A template: points, signals, protocol, encoding, batching, clock quality, and fault rates. A fleet is device types times counts. |
| **Device** | One simulated device: an ID, a seed, its points, its state machine, its clock, and its buffer. |
| **Point** | One value on a device: a sensor channel, tag, register, or object resource. Read-only or writable, with a type, unit, and scaling. |
| **Signal** | The model that produces a point's true values over time (§7). |
| **Frame** | A batch of samples sent together (D7). |
| **Adapter** | The code that speaks one protocol, in both directions (§9). |
| **Sink** | An adapter that only sends: Kafka, files, stdout, a direct database write. |
| **Scenario** | A timeline of changes to the field during a run (§11). |
| **True / device / arrival time** | When a value really happened; what the device's clock stamped on it; when it reached the system under test (§10.1). |
| **Checker** | Compares what landed with what was sent (§12). |

## 5. Architecture

### 5.1 Layers

```
signals ─→ device model ─→ faults ─→ protocol adapters ←→ system under test
(true values)  (state, rules)  (clock, batching,     (both directions)
                                encoding, transport)
                    ↑
              control plane (runtime changes, scenarios)

true values (recomputed from the seed, or recorded) ─→ checker ←─ what the system under test stored
```

### 5.2 Crates

| Crate | Contents |
|-------|----------|
| `sensim` | CLI and fleet runner: reads the configuration, runs the workers, serves the control plane |
| `sensim-signal` | Signal models and the simulation clock |
| `sensim-device` | Device model: points, state machine, behaviour rules, stubs, record and replay |
| `sensim-faults` | Sensor, clock, lifecycle, batching, encoding, and transport faults |
| `sensim-encode` | Payload encodings (§10.5) |
| `sensim-mqtt`, `sensim-opcua`, `sensim-modbus`, … | One crate per protocol (§9) |
| `sensim-check` | The checker (§12) |

Crates are added as they're built; today only `sensim` exists.

### 5.3 Process model

- A run has N worker threads (one per core by default). Each owns a fixed slice of the devices and steps them; there is
  no task or thread per device.
- **Schedule from the start time:** a device's k-th frame is due at `start + k × period`, computed, never accumulated
  from the previous send, so rates don't drift over a week.
- **Falling behind is reported, not hidden:** when a worker can't send on time, it records how late it is (schedule
  lag) and keeps the intended timestamps, so latency measured downstream isn't understated. Whether a late worker skips
  frames or sends them in a burst is a setting.
- **Scaling out:** a run can be split into shards by device range (`--shard 3/8`). Each shard is an independent
  process with the same configuration and seed, so shards need no coordinator. Containers on Kubernetes take the shard
  from their ordinal.
- **Connections:** one host handles thousands of TCP connections. Past about 30,000 to one address, a host runs out of
  source ports; more source addresses or more shards fix that.

### 5.4 Simulation clock (D10)

| Mode | Use |
|------|-----|
| `real-time` (default) | Streaming tests and long-running benchmarks |
| `accelerated` (×N) | Long scenarios in less time |
| `as-fast-as-possible` | Generating history for backfill tests and datasets |

Device clocks (§10.2) are derived from the simulation clock, so clock faults work the same in every mode.

## 6. Fleet configuration

A run is described by one file. Format to be decided (Q6); a sketch in TOML:

```toml
[fleet]
seed = 42
clock = "real-time"

[[device-type]]
name = "temperature"
count = 4000
protocol = "mqtt"
encoding = "json-row"
points = [{ name = "temp_c", signal = "random-walk", min = 15.0, max = 35.0, rate-hz = 1 }]

[[device-type]]
name = "vibration"
count = 100
protocol = "mqtt"
encoding = "binary-f32-le"
batch = { mode = "time-window", window-ms = 100 }
points = [{ name = "accel_x", signal = "harmonics", base-hz = 49.5, rate-hz = 8000 }]
faults = { clock-drift-ppm = 50, link-outage = { every = "6h", duration = "10m" } }

[[device-type]]
name = "plc"
count = 900
protocol = "opcua"
points = [
  { name = "setpoint", writable = true, default = 60.0 },
  { name = "flow", signal = "first-order-lag", follows = "setpoint", tau-s = 8, rate-hz = 10 },
]
```

Settings resolve from the most specific level: runtime override, device, device type, fleet, then the default (§15).

## 7. Signals (R1, R11)

### 7.1 Models

| Model | Use |
|-------|-----|
| Constant, step, ramp, counter (with wraparound) | Status, energy meters, totals |
| Sine with harmonics and noise | Vibration, AC current |
| Bounded random walk, Ornstein–Uhlenbeck | Temperature, pressure, slowly varying process values |
| First-order lag towards a target | A process value following a setpoint (§8.3) |
| State machine with dwell times | Machine states, door open/closed, alarms |
| Daily and weekly cycles | Load, occupancy, energy |
| Correlated channels | Several points driven by one hidden process (flow and pressure, 3-axis vibration) |
| Replay | A recorded series, looped or stretched |

Models compose: a sum of a cycle, a walk, and noise is one signal.

### 7.2 Rates

From one sample a day up to 8 kHz per channel (R1), set per point. Fast and slow points mix in one device.

### 7.3 Frames (D7)

At 5,000 devices × 8 kHz, one message per sample would be 40 million messages a second, which no broker takes. Real
high-rate devices send frames (for example 100 ms = 800 samples), so sensim does too: 10 messages a second per device,
50,000 a second for the whole fleet. Batching is set per device type (§10.4).

### 7.4 Determinism (D6)

Each device's random generator is seeded from the fleet seed and the device ID. The value of any point at any true time
can be recomputed without replaying the run, which the checker uses (§12).

### 7.5 Event workloads (R11)

Not every time series comes from a sensor. Two event generators use the same clock, faults, and adapters:

- **Click events:** users arriving through the day, sessions, page popularity following a power law, funnels with
  drop-off, and bots.
- **Finance events:** trades per symbol arriving at random (Poisson), prices following a random process (geometric
  Brownian motion), bid and ask quotes, and bursts at market open and close.

Lower priority than devices (§19).

## 8. Device emulation (R2, R4, R5)

### 8.1 Point map

Each device type declares its points: name, type, unit, scaling (raw int16 × 0.1), read-only or writable, and where the
point appears in each protocol (MQTT topic and field, OPC UA node, Modbus register, LwM2M object and resource).

### 8.2 State machine

Every device has a lifecycle: `booting → running → degraded → fault → offline`, with transitions driven by time, rules,
commands, and faults. A reboot resets counters and sequence numbers; going offline empties or keeps the buffer
depending on the device type (§10.4).

### 8.3 Behaviour (D8)

| Kind | Example |
|------|---------|
| **Rule** | Writing `setpoint` moves `flow` towards it with an 8 s time constant. A `reboot` command takes the device offline for 30 s, then it boots with counters reset. Closing a valve drops flow and raises upstream pressure. |
| **Stub** | A fixed or templated reply to a request: a device-info read, a method call, an HTTP endpoint. |
| **Record and replay** | Record a real device's traffic once, then replay its replies and data. |
| **Script** | For what rules can't express, a sandboxed script runs on the same events (Q3). |

Rules and scripts never run per sample, so their cost doesn't grow with sample rate.

### 8.4 Inbound traffic (R5)

Each adapter delivers inbound messages to the device as events: writes to points, method calls and commands,
configuration changes, firmware updates, and reads. The device answers according to its rules and stubs, and can
reject, delay, or ignore a request when a fault says so.

## 9. Protocols (R4, R5)

| Protocol | Device role | Out | In | Candidate library | Priority |
|----------|-------------|-----|----|-------------------|----------|
| MQTT 3.1.1 / 5 | client | telemetry, events, replies | commands, config | `rumqttc` | First (Q2) |
| Sparkplug B (on MQTT) | edge node | births, deaths, data | node and device commands | own, protobuf | First (Q2) |
| OPC UA | server | subscriptions, reads | writes, method calls | `async-opcua` | First (Q2) |
| Modbus TCP | server | register and coil reads | register and coil writes | `tokio-modbus` | First (Q2) |
| HTTP | client and server | POSTs, webhooks | REST requests | `hyper`, `axum` | Second |
| CoAP / LwM2M | client | observe notifications | reads, writes, executes | to evaluate | Second |
| Kafka | sink | records | — | `rdkafka` | Second |
| Files (Parquet, NDJSON, CSV) | sink | — | — | `arrow`, `parquet` | First |
| stdout | sink | — | — | — | First |
| Modbus RTU, DNP3, BACnet, IEC 61850 | — | — | — | to evaluate | Later |

Library choices are to be confirmed when each adapter is built, against their latest releases.

## 10. Faults (R6–R10)

### 10.1 Three times per sample (D5)

- **True time:** when the value really happened, by the simulation clock.
- **Device time:** what the device's clock stamped on it. Wrong when a clock fault is on.
- **Arrival time:** when it reached the system under test.

Faults are applied in this order, so they combine the way they do in the field:

```
signal ─→ sensor faults ─→ device clock ─→ buffer and batching ─→ encoding ─→ transport ─→ system under test
```

Setting names below are a first draft (§15).

### 10.2 Device clock (R8)

| Fault | Setting |
|-------|---------|
| Fixed offset | `clock-offset` |
| Drift | `clock-drift-ppm` |
| Steps forward or backward (NTP corrections) | `clock-step` |
| Reset to 1970 after a reboot (dead real-time-clock battery) | `clock-reset-on-boot` |
| Local time stamped as UTC; DST hour repeated or skipped | `clock-timezone` |
| Same timestamp repeated (frozen clock) | `clock-freeze` |
| Wrong epoch unit (seconds, milliseconds, microseconds) | `clock-epoch-unit` |
| Truncated resolution (whole seconds only) | `clock-resolution` |

### 10.3 Sensor and device issues (R6)

| Fault | Setting |
|-------|---------|
| Stuck value | `sensor-flatline` |
| Spikes | `sensor-spike-rate` |
| Clipping at the range limit | `sensor-range` |
| Rising noise | `sensor-noise` |
| Slow bias drift | `sensor-bias-drift` |
| Disconnected-sensor sentinels (`-9999`, `NaN`, `0xFFFF`) | `sensor-disconnect` |
| Unit change (°C to °F after an update) | `sensor-unit-change` |
| Reboot (counters and sequence numbers reset) | `device-reboot` |
| Firmware update that changes the payload schema | `device-firmware` |
| Two devices with the same ID (cloned configuration); a device that changes its ID | `device-id-collision`, `device-id-change` |
| Lower send rate on low battery | `device-power-save` |

### 10.4 Buffering and batching (R7, R9)

| Behaviour | Setting |
|-----------|---------|
| One sample per message, fixed count, time window, size limit, or adaptive (larger batches when behind) | `batch-mode` |
| One timestamp per batch plus offsets, or a timestamp per sample | `batch-timestamps` |
| Store and forward during an outage, flushed oldest first or newest first | `buffer-flush-order` |
| Buffer capacity, and what overflow drops | `buffer-capacity`, `buffer-overflow` |
| A gateway merging many devices into one batch | `gateway` |
| A frame split across messages | `batch-split` |

### 10.5 Encodings (R10)

| Encoding | Variants |
|----------|----------|
| JSON | Row, columnar, nested; numbers as strings; field names that differ by firmware version |
| Sparkplug B | Protobuf, with aliases and metrics |
| CBOR, MessagePack | Compact binary maps |
| CSV, InfluxDB line protocol | Text lines |
| Raw binary | Endianness, Modbus word order, scaled integers (int16 × 0.1), packed frames |
| Compression | gzip, zstd, none |
| Malformed | Truncated, invalid UTF-8, `NaN` and `Inf`, nulls, wrong types (`encoding-malformed-rate`) |

### 10.6 Transport (R7)

| Fault | Setting |
|-------|---------|
| Delay drawn from a distribution, including a long tail | `link-delay` |
| Loss | `link-loss` |
| Duplicates (QoS 1 redelivery, retries after a timeout) | `link-duplicate` |
| Reordering within a window | `link-reorder` |
| Outages and flapping connections | `link-outage` |
| Bandwidth limit | `link-bandwidth` |
| Data days old sent as backfill | `link-backfill` |

## 11. Control plane and scenarios (R5)

- **Control plane:** an HTTP API on the running field. Selectors pick devices by ID, device type, tag, region, or a
  seeded percentage; actions change any setting, send a command to a device, or start a scenario step. Every change is
  logged with the simulation time, so the checker knows what was injected and when.
- **Scenarios:** a file of timed steps, for example:
  - at 6 h, region A loses its link for 2 h, then flushes newest first
  - at 12 h, 20% of `plc` devices update firmware (schema v2)
  - at 18 h, 3% of devices reset their clocks to 1970

  A scenario plus a seed reproduces the same bad week exactly.

## 12. Checker

The checker answers: did the system under test store what the field sent, at the right time?

- **Expected:** recomputed from the seed (§7.4), plus the log of injected faults (§11). A record of what was sent is an
  alternative (Q4).
- **Per device and time window:** missing samples, duplicates, values that differ, late samples stored in the wrong
  place, stored timestamps vs true time, and payloads that failed to decode.
- **Reads the system under test** through a reader per target: SQL (DuckDB, PostgreSQL), files (Parquet), or a
  Timelake table (Q5).

## 13. Observability

- Prometheus metrics: samples and messages sent per adapter, schedule lag, connections by state, inbound requests,
  injected faults by kind.
- Optional OpenTelemetry traces for request and reply flows.
- Structured logs; no per-sample logging.

## 14. Deployment

- One static binary and a container image.
- Shards across hosts or Kubernetes pods by device range (§5.3).
- Runs as a library inside a test or benchmark harness.

## 15. Settings (D9)

- Every fault and behaviour in §7–§11 is a setting with an off state.
- Levels, most specific first: runtime override, device, device type, fleet, default.
- Names describe the mechanism (`clock-drift-ppm`, `buffer-flush-order`), not the outcome (`bad-clock`, `flaky`).
- Each setting is documented with what it does, what it costs, and why its default is what it is.

## 16. Performance targets

To be measured by sensim's own benchmark once the generator exists:

- Generation of simple signals well above what one network interface can carry, so encoding and the network are the
  limits (G2).
- 5,000 devices with the mix in §6 on one host, with schedule lag reported for every device.
- Constant memory over a week-long run.

## 17. Prior art

Tools reviewed on 2026-10-16 when deciding to build sensim. None covers continuous native generation, device
emulation in both directions, and faults together. Ideas sensim uses are credited in `PRIOR_ART.md`.

| Tool | Covers | Missing for sensim |
|------|--------|--------------------|
| TSBS | Seeded, deterministic IoT and DevOps data | One-off datasets, no devices or protocols |
| Apache IoTDB's IoT-Benchmark | Many devices, out-of-order data, long runs | Java; writes to databases, no device protocols |
| mqttwrk | Rust MQTT load at scale | Dummy payloads, no device model |
| mer (iot-data-generator) | Rust, MQTT/HTTP/TCP, per-device state | Not built for load |
| rumsim | Rust MQTT device simulator on Kubernetes | Small; MQTT only |
| opc-plc | OPC UA server with fast and slow nodes | C#; simple value patterns |
| eMQTT-Bench | Millions of MQTT connections | Erlang; synthetic payloads |
| Fluent Bit `dummy`, Vector `demo_logs` | Continuous generation at a set rate | Log records, not sensors or devices |
| Mountebank | Stubbed, scripted, and recorded fake services over several protocols | HTTP, TCP, and SMTP only; not devices |
| Toxiproxy | Network faults (latency, bandwidth, timeouts) between services | Transport only |
| Eclipse Leshan and Wakaama, Eclipse Tahu | LwM2M and Sparkplug B reference clients | One protocol each |

## 18. Open questions

| # | Question | Current leaning |
|---|----------|-----------------|
| Q1 | Main goal first: benchmarking a database's ingest (Timelake), or testing a whole IoT stack (devices → broker or gateway → pipeline → storage)? | Decides whether the generator and sinks or the device layer comes first |
| Q2 | Which protocols first? | MQTT with Sparkplug B, OPC UA, and Modbus TCP |
| Q3 | Script engine for behaviour beyond rules: Rhai, Lua, or WebAssembly? | Rhai (pure Rust, sandboxed); WebAssembly if scripts must come in many languages |
| Q4 | Checker's expected values: recompute from the seed, or record what was sent? | Recompute; record only where recomputing is impossible (replay, inbound-driven behaviour) |
| Q5 | Which systems can the checker read first? | Parquet files and SQL (DuckDB, PostgreSQL), then Timelake |
| Q6 | Configuration format: TOML or YAML? | TOML, as in the Rust ecosystem |
| Q7 | Python bindings for driving sensim from tests? | Later; the CLI and control plane first |

## 19. Roadmap (draft)

Phases in order; none is started.

| Phase | Contents |
|-------|----------|
| 0 | Repository, workspace, checks, CI, this specification (done 2026-10-17) |
| 1 | Simulation clock, signal models, fleet configuration, workers with schedule lag, file and stdout sinks, MQTT out, metrics |
| 2 | Faults: clock, batching and buffering, encodings, transport, sensor and device issues |
| 3 | Device model, rules, stubs; MQTT and Sparkplug B commands; OPC UA and Modbus TCP servers |
| 4 | Control plane and scenarios; checker |
| 5 | More protocols (HTTP, CoAP and LwM2M, Kafka), event workloads (§7.5), record and replay, scripts |

Phases 1–3 may change order once Q1 is answered.

## 20. Features

| ID | Feature | Req | Section | Phase | Status |
|----|---------|-----|---------|-------|--------|
| F1 | Signal models and the simulation clock | R1 | §7, §5.4 | 1 | Planned |
| F2 | Fleet runner: thousands of devices per process, schedule lag, shards | R1, R3 | §5.3 | 1 | Planned |
| F3 | Sinks: files, stdout, Kafka | R1 | §9 | 1, 5 | Planned |
| F4 | Device clock faults | R8 | §10.2 | 2 | Planned |
| F5 | Sensor and device issues | R6 | §10.3 | 2 | Planned |
| F6 | Buffering and batching | R7, R9 | §10.4 | 2 | Planned |
| F7 | Encodings | R10 | §10.5 | 2 | Planned |
| F8 | Transport faults | R7 | §10.6 | 2 | Planned |
| F9 | Device model: points, state machine, rules, stubs | R2, R5 | §8 | 3 | Planned |
| F10 | MQTT and Sparkplug B adapters, both directions | R4, R5 | §9 | 1, 3 | Planned |
| F11 | OPC UA server adapter | R4, R5 | §9 | 3 | Planned |
| F12 | Modbus TCP server adapter | R4, R5 | §9 | 3 | Planned |
| F13 | Control plane and scenarios | R5, R6 | §11 | 4 | Planned |
| F14 | Checker | R6–R10 | §12 | 4 | Planned |
| F15 | HTTP, CoAP, and LwM2M adapters | R4, R5 | §9 | 5 | Planned |
| F16 | Click and finance event workloads | R11 | §7.5 | 5 | Planned |
| F17 | Record and replay; scripts | R5 | §8.3 | 5 | Planned |
| F18 | Observability: metrics, traces, logs | R1 | §13 | 1 | Planned |
