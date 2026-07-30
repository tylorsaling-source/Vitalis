# Mesh33 Packetized Training Harness

Status: proposal / field note

This note captures a downstream experiment that combines Vitalis/jcode-style
durable harness ideas with packetized training traffic from the VITALIS-X
packetizer work.

## Context

Mesh33 is an experimental Pi 5 / Uno Q / ESP32 training fabric. A Pi 5 acts as
the coordinator, an Uno Q is planned as a command/router bridge, and ESP32s are
planned as small worker shards. The current prototype trains a 33M-parameter
int8 windowed model by sending each worker a roughly 556 KB training packet.

The worker packet is split into transport frames with CRC-protected reassembly.
The first implementation used a conservative 512-byte frame payload because it
matches BLE/serial stability concerns.

## VITALIS-X Packetizer Finding

The VITALIS-X Rust packetizer benchmark has packet-aligned copy strategies for
4096-byte and 8192-byte regions, including ready-barrier variants that exclude
thread startup/join noise from the timed copy loop.

That maps cleanly onto Mesh33 transport profiles:

| Profile | Frame payload | Intended path |
| --- | ---: | --- |
| `ble_512` | 512 bytes | ESP32 BLE / conservative serial |
| `wifi_4096` | 4096 bytes | ESP32 Wi-Fi / Pi staging |
| `wifi_8192` | 8192 bytes | Pi to Uno Q / strong local links |

For a representative 556,032-byte worker packet:

| Frame payload | Frames per worker job | Estimated frame overhead |
| ---: | ---: | ---: |
| 512 | 1086 | 3.516% |
| 4096 | 136 | 0.44% |
| 8192 | 68 | 0.22% |

## Local Harness Result

A local Python Mesh33 harness benchmark was run for 20 training batches at each
frame size. It measured the full local batch path: packet encode, frame split,
frame reassemble, worker train, and metric emit.

| Frame payload | Average seconds per batch | Frames per worker job |
| ---: | ---: | ---: |
| 512 | 0.4821 | 1086 |
| 4096 | 0.4332 | 136 |
| 8192 | 0.4324 | 68 |

The 8192-byte profile was about 10.3% faster than 512-byte frames in this local
harness loop. This is not yet a hardware radio benchmark. It is evidence that
packet sizing is already measurable before real BLE/Wi-Fi transfer costs are
included.

## Why This Belongs In A Vitalis-Style Harness

The useful part is not only bigger frames. The harness needs to choose and
verify transport profiles as part of durable job orchestration:

- select frame size by link type and worker capability
- record frame count, CRC, packet ID, and route/checksum hints per job
- resume or retransmit failed worker packets
- keep training alive with a coordinator/watchdog
- expose metrics to local, remote, and phone clients
- run deterministic smoke tests against packet and worker protocols

This fits the existing Vitalis/jcode design direction:

- single durable coordinator process
- thin clients that attach/detach without killing work
- task-DAG-first scheduling
- worker lifecycle states
- report-back artifacts
- remote attach before full migration
- deterministic harness tests

## Suggested First Slice

Add a small packet-profile benchmark/smoke-test surface to the harness:

1. Generate a representative worker packet.
2. Run frame split/reassembly using 512, 4096, and 8192 byte payloads.
3. Emit JSON metrics with frame count, overhead, CRC, elapsed time, and selected
   transport profile.
4. Gate correctness with round-trip CRC checks.
5. Let a coordinator choose the largest profile supported by the current route.

This gives the harness a concrete, measurable optimization path for distributed
training and other high-volume worker traffic.
