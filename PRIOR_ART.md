# Prior art

This file credits the ideas sensim takes from other open-source projects and keeps notes on what we're studying. We
take ideas, never code. How to credit is in [CONTRIBUTING.md](CONTRIBUTING.md#credit); how sensim compares with other
tools is in SPEC.md §17.

## Ideas

Methods we plan to adopt and implement ourselves. Licenses checked on 2026-10-17.

| Idea | From | License of the source | Where in sensim |
|---|---|---|---|
| Seeded, deterministic generation, so a dataset is reproducible from its parameters | [TSBS](https://github.com/timescale/tsbs) | MIT | Per-device seeds (SPEC D6, §7.4) |
| Generating out-of-order and missing data as configured ratios | [IoT-Benchmark](https://github.com/thulab/iot-benchmark) (Apache IoTDB) | Apache-2.0 | Transport and batching faults (§10.4, §10.6) |
| Node count and change rate as plain settings for a simulated server | [opc-plc](https://github.com/Azure-Samples/iot-edge-opc-plc) | MIT | Device types and point rates (§6, §9) |
| Stubs selected by request predicates, and a proxy that records real replies for replay | [Mountebank](https://github.com/bbyars/mountebank) | MIT | Stubs, record and replay (§8.3) |
| Named network faults attached to a connection and changed at runtime over an API | [Toxiproxy](https://github.com/Shopify/toxiproxy) | MIT | Transport faults and the control plane (§10.6, §11) |
| Sending on a schedule from the intended start time, and measuring latency from it (avoiding coordinated omission) | [wrk2](https://github.com/giltene/wrk2) | Apache-2.0 | Worker schedule and lag (§5.3) |
| Checking a recorded history of what a system did against a model of what it should have done | [Jepsen](https://github.com/jepsen-io/jepsen) | EPL-1.0 | The checker (§12) |

## Studying

| Project | License | Notes |
|---|---|---|
| [mqttwrk](https://github.com/bytebeamio/mqttwrk) | Apache-2.0 | wrk-style MQTT load in Rust: connections, publishers per connection, rate |
| [Eclipse Tahu](https://github.com/eclipse-tahu/tahu) | EPL-2.0 | Sparkplug B reference edge nodes: births, deaths, commands |
