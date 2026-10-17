# sensim

A field of simulated devices for testing IoT systems: thousands of sensors, PLCs, meters, and gateways that produce
data at up to kilohertz rates, talk real protocols in both directions, and misbehave the way real devices do: late
data, wrong clocks, odd batches and encodings, outages, and reboots. A checker then compares what the system under
test stored with what the field sent.

**Status:** design. Nothing is built yet. [SPEC.md](SPEC.md) holds the requirements, features, design, and roadmap.

## Planned

- Continuous, seeded signal generation, from one sample a day to 8 kHz per channel, for runs of days or weeks
- 5,000 or more devices per process, sharded across hosts when needed
- Devices that send and receive over MQTT and Sparkplug B, OPC UA, Modbus TCP, HTTP, CoAP, and LwM2M, and answer
  writes and commands with rules and stubs
- Faults: device clocks, sensor and device issues, batching and buffering, encodings, and transport
- A control plane and scenario files to change the field while it runs
- A checker that reports missing, duplicated, late, and mistimed data per device

## License

[Apache License 2.0](LICENSE).
