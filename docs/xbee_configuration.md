---
layout: default
title: XBee Configuration
parent: Radio Configuration
nav_order: 1
last_modified_at: 2026-09-25 12:00:00 -0400
---

# XBee Configuration (Motion Capture Link)
{: .no_toc }

**Modules:** Digi XBee-PRO 900HP (S3B), DigiMesh firmware  
**Purpose:** one-way stream of motion-capture pose data from the ground station to the vehicles

The mocap link is separate from the ExpressLRS control link on the [previous page]({% link docs/radio_configuration.md %}) — ELRS is 2.4 GHz and carries RC and MAVLink telemetry, while these radios are 900 MHz and carry pose data only. They do not interfere with each other.

One radio is the **sender**, on the ground station next to the mocap PC. Every other radio is a **receiver**, one per vehicle, wired to a spare flight-controller UART. The sender broadcasts; receivers only listen.

---

## Configure with XCTU

Connect each radio to a USB adapter, open it in XCTU, set the values below, and click **Write** to store them permanently. Every radio gets the common settings; the role settings differ.

### Common to every radio

| XCTU setting | AT | Value |
|---|---|---|
| Network ID | `ID` | `474` |
| Preamble ID | `HP` | `4` |
| Channel Mask | `CM` | `FFFFFF8000000000` |
| Transmit Options | `TO` | `41` |
| Broadcast Multi-Transmits | `MT` | `0` |
| Unicast Mac Retries | `RR` | `0` |
| TX Power Level | `PL` | `2` (mid) |
| Network Hops | `NH` | `1` |
| Destination Address High / Low | `DH` / `DL` | `0` / `FFFF` (broadcast) |
| API Enable | `AP` | `0` (transparent) |
| Baud Rate | `BD` | `7` (115200) |
| Sleep Mode | `SM` | `0` |
| AES Encryption Enable | `EE` | `0` |
| DIO6/RTS | `D6` | `0` |
| DIO7/CTS | `D7` | `1` |

### Sender (ground station)

| XCTU setting | AT | Value |
|---|---|---|
| Routing / Messaging Mode | `CE` | `0` (standard) |
| Node Identifier | `NI` | `MOCAP-TX` |

### Receivers (one per vehicle)

| XCTU setting | AT | Value |
|---|---|---|
| Routing / Messaging Mode | `CE` | `3` (non-routing) |
| Node Identifier | `NI` | `VEH-01`, `VEH-02`, … |

> **`ID`, `HP` and `CM` must be identical on every radio.** A radio with any of them different does not communicate at all — it looks like dead hardware, not a weak link. In particular, a replacement radio arrives from the factory with a different channel mask and will be silent until `CM` is set to the value above. Do not change `CM` to "get more channels".

Label each module with its `NI` once written. The modules are physically identical and carry no visible serial number.

---

## Verify

With two radios on one computer, `~/uas/tools/xbee.py` reads settings and measures the link:

```bash
python3 ~/uas/tools/xbee.py --port /dev/ttyUSB0 info
python3 ~/uas/tools/xbee.py link --tx /dev/ttyUSB0 --rx /dev/ttyUSB1 --rate 100 --size 30
```

A healthy link delivers **100 %** with about **21 ms** median latency. Anything materially below that is a configuration mismatch, not a range problem — recheck `ID`, `HP` and `CM` first.

**Capacity.** Useful throughput saturates around 90 kbps, but latency climbs steeply near that limit. Keep the total offered load at or below **~25 kbps** across all vehicles to stay at 100 % delivery and ~21 ms. A 30-byte pose packet at 100 Hz is 24 kbps, so that is one vehicle at 100 Hz or four at 50 Hz (which needs measuring at a 50 Hz rate before use — latency will be higher).

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Nothing received at all | `ID`, `HP` or `CM` differs | Compare all three against the table above |
| Radio does not respond in XCTU | Another program holds the port | Close other serial tools; only one program can own a port |
| Delivery well below 100 % | Wrong `CM`, or the radios are far apart with obstructions | Check `CM` first, then move them |
| Latency in the hundreds of ms | Offered data rate too high | Reduce the pose rate or packet size |
| Receiver hears nothing once fitted to the vehicle | Flow control | `D6` must be `0` — the flight-controller UART has no RTS wired |
