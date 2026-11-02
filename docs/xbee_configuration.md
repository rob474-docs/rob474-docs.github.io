---
layout: default
title: XBee Configuration
parent: Radio Configuration
nav_order: 1
last_modified_at: 2026-10-12 12:00:00 -0400
---

# XBee Configuration (Motion Capture Link)
{: .no_toc }

**Modules:** Digi XBee-PRO 900HP (S3B), DigiMesh firmware  
**Purpose:** stream motion-capture pose data from the ground station to the vehicles  

This page sets up the motion-capture link end to end: the radios, the wiring, the flight-controller serial port, the pose message, and the estimator. Work through the parts in order. Each part ends with a check, and a part that does not pass its check will not be fixed by moving on.

The mocap link is separate from the ExpressLRS control link on the [previous page]({% link docs/radio_configuration.md %}). ELRS is 2.4 GHz and carries RC and MAVLink telemetry, while these radios are 900 MHz and carry pose data. They do not interfere with each other.

One radio is the **sender**, on the ground station next to the mocap PC. Every other radio is a **receiver**, one per vehicle, wired to a flight-controller UART. The traffic is deliberately lopsided: the ground station broadcasts pose data continuously, and each vehicle sends back only a 1 Hz heartbeat and the acknowledgements a waypoint upload needs.

**The vehicle's radio transmits as little as possible, and this is the single most important property of the link.** These radios are half duplex, so a radio that is transmitting cannot receive. Every packet a vehicle sends costs it inbound pose packets. On the course airframe, leaving the flight controller's default telemetry streams running on this port dropped pose delivery to 86 %. Holding the return traffic to a 1 Hz heartbeat raised it to 99.4 % at the same offered rate. Parts 4 and 5 exist for this reason.

---

## Frame conventions

**Read this before Part 6, and before creating an object in Tracker.** Every
sign error in this system lives here, and a heading error is the one failure
that reliably destroys an airframe: the position loop applies every correction
in a frame rotated from reality, so the vehicle accelerates away from its
setpoint instead of toward it.

Three frames are involved:

| Frame | Axes | Units |
|---|---|---|
| **Vicon room** | x east, y north, z up (ENU) | millimetres |
| **PX4 / your controller** | x north, y east, z down (NED) | metres |
| **Object rotation from Tracker** | rotation since the object was created, not an absolute heading | axis-angle |

The conversion the broadcaster applies, which is `--frame enu`:

```
north =  y / 1000
east  =  x / 1000
down  = -z / 1000
```

**Create every Tracker object with the nose along room +y.** Tracker aligns a
new object's axes with the room's axes at the moment of creation, so what it
reports afterwards is the rotation *since* then. Working that through, the
vehicle's NED heading is

```
psi = psi_0 - phi
```

where `phi` is the rotation Tracker reports and `psi_0` is the NED heading the
nose pointed at creation. Room +y is NED north, so creating the object facing +y
makes `psi_0 = 0` and the broadcaster's `--yaw-offset` is 0, which is what it
runs with. Create the object facing room +x instead and every heading is 90
degrees out, with no way to correct it other than deleting the object and
making it again.

There is no other room-dependent yaw term. The room convention is carried
entirely by the axis swap above and by the yaw sign, so do not add an offset to
compensate for something else.

> **The room is not aligned to the earth's frame.** Magnetic headings are not
> useful here. Room +y is roughly north and +x roughly east, and "north" in your
> controller means room +y, not magnetic north. The vehicle has no
> magnetometer, so mocap yaw is its only absolute heading.

Verifying this is not optional, and it takes two minutes. Carry the vehicle by
hand, disarmed, and run the five direction checks in
[Using Motion Capture]({% link docs/motion_capture.md %}#the-direction-checks-before-you-fly).
Two of the five were wrong when this system was first set up, and each would
have destroyed the vehicle on its first hover.

---

## Part 1: Configure the Radios with XCTU

Connect each radio to a USB adapter, open it in XCTU, set the values below, and click **Write** to store them permanently. Every radio gets the common settings. The role settings differ.

Power down every other radio in the room while you configure one. A transparent-mode radio pushes everything it hears over the air out its serial port, which collides with the configuration dialogue and makes a working radio look unresponsive.

### 1.1 Common to every radio

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

### 1.2 Sender (ground station)

| XCTU setting | AT | Value |
|---|---|---|
| Routing / Messaging Mode | `CE` | `0` (standard) |
| Node Identifier | `NI` | `MOCAP-TX` |

### 1.3 Receivers (one per vehicle)

| XCTU setting | AT | Value |
|---|---|---|
| Routing / Messaging Mode | `CE` | `3` (non-routing) |
| Node Identifier | `NI` | `VEH-01`, `VEH-02`, … |

> **`ID`, `HP` and `CM` must be identical on every radio.** A radio with any of them different does not communicate at all, and looks like dead hardware rather than a weak link. In particular, a replacement radio arrives from the factory with a different channel mask and will be silent until `CM` is set to the value above. Do not change `CM` to "get more channels".

**Leave `MT` at 0.** Setting `MT` to 1 makes the sender transmit every broadcast twice, which does raise delivery to 99.98 %. It also doubles the airtime, and at 50 Hz the sender then queues: measured latency rose from 20 ms median to 62 ms median, 120 ms at the 95th percentile and 158 ms worst case. A position loop cannot use that. `EKF2_EV_DELAY` compensates a fixed delay, not 100 ms of jitter. Take 99.4 % at 20 ms over 100 % at 62 ms.

Label each module with its `NI` once written. The modules are physically identical and carry no visible serial number.

---

## Part 2: Verify the Radios on the Bench

With two radios on one computer, `~/uas/tools/xbee.py` reads settings and measures the link:

```bash
python3 ~/uas/tools/xbee.py --port /dev/ttyUSB0 info
python3 ~/uas/tools/xbee.py link --tx /dev/ttyUSB0 --rx /dev/ttyUSB1 --rate 100 --size 30
```

A healthy link delivers **100 %** with about **21 ms** median latency. Anything materially below that is a configuration mismatch, not a range problem. Recheck `ID`, `HP` and `CM` first.

Pass `--baud 115200` to skip baud autodetection once you know the radio is set correctly. Autodetection writes to the radio at eight wrong baud rates before reaching the right one, and each of those writes is transmitted over the air as noise.

---

## Part 3: Wire the Receiver to the Vehicle

On the MicoAir H743V2-AIO the ELRS receiver occupies UART1 and the MTF-01 optical flow module occupies UART4, so the mocap receiver goes on the **GPS header, UART3**. PX4 calls that port **GPS 1**, and NuttX calls it `/dev/ttyS2`. The full map for this board:

| Silkscreen | PX4 name | Device | Used by |
|---|---|---|---|
| UART1 | TELEM1 | `/dev/ttyS0` | ELRS receiver |
| UART3 (GPS header) | GPS 1 | `/dev/ttyS2` | **mocap receiver** |
| UART4 | TELEM2 | `/dev/ttyS3` | MTF-01 optical flow |
| UART5 | TELEM3 | `/dev/ttyS4` | free |
| UART8 | TELEM4 | `/dev/ttyS7` | free |

A reversed or over-voltage power line will destroy the radio. The module itself runs on 3.3 V, and a carrier board may or may not regulate for you. Confirm the header's power pin voltage against the rating of the board you are fitting before the first power-up.

1. Connect the radio's **GND** and power to the header, observing both polarity and voltage.
2. Connect the radio's **DOUT** to the flight controller's **RX3**.
3. Connect the radio's **DIN** to the flight controller's **TX3**.

The vehicle needs `DIN` to send its heartbeat and to acknowledge a waypoint upload, so wire it. It is also the reason Part 5 exists. With `DIN` wired, everything the flight controller streams on this port goes out over the air, and because the radio is half duplex, each of those transmissions costs the vehicle inbound pose packets. PX4's default stream set on this port measured 86 % pose delivery. Part 5 brings it to 99.4 % by cutting the return traffic to the heartbeat and the mission messages.

Leaving `DIN` disconnected is the alternative. It removes the contention entirely and physically guarantees the radio cannot transmit, which no parameter change or reflash can undo. It also gives up the heartbeat and makes waypoint upload impossible, because the mission protocol is a handshake. Wire `DIN` unless a vehicle will only ever receive poses.

`D6` must be `0`, as set in Part 1. The flight-controller header has no RTS line, so a receiver expecting RTS flow control withholds data forever and the link looks dead.

---

## Part 4: Configure the Flight-Controller Serial Port

Four parameters. The first releases the port from the GPS driver, which otherwise holds it and swallows the pose stream while hunting for a GPS that is not there.

| Parameter | Value | Reason |
|---|---|---|
| `GPS_1_CONFIG` | 0 (Disabled) | Release GPS 1 from the GPS driver |
| `MAV_2_CONFIG` | 201 (GPS 1) | Run a MAVLink instance on the mocap port |
| `SER_GPS1_BAUD` | 115200 | Match the radio's `BD` = 7 |
| `MAV_2_MODE` | 1 (Custom) | Stream nothing by default. See Part 5 |
| `MAV_2_FORWARD` | 0 (Disabled) | Do not forward other links' traffic out of this one. See below |

**`MAV_2_FORWARD` is not cosmetic.** With it enabled the vehicle forwards
everything it receives on its other links out of this one. Measured on the
course airframe that was 6531 bytes per second, and it dropped pose delivery
from 99 % to about 6 %. It looks exactly like a hardware fault: the radios test
fine, the link is simply full of the vehicle's own retransmissions, and because
the radio is half duplex each one deafens its own receiver.

`MAV_2_CONFIG`, `MAV_2_MODE` and `MAV_2_FORWARD` take effect only after a restart.

**Power cycle the board. Do not use a software reboot.** On this flight controller an NSH `reboot` after saving parameters leaves the board silent and requires a power cycle anyway.

**Check.** With the board back up:

```bash
python3 ~/uas/tools/nsh.py 'mavlink status'
```

The instance on `/dev/ttyS2` must report `mode: Custom` and a transmit rate near 39 B/s. A transmit rate in the hundreds or thousands of B/s means `MAV_2_MODE` did not take, and pose delivery will suffer in proportion.

---

## Part 5: Hold Down the Return Traffic

Custom mode streams nothing by default, but PX4 always configures three streams on every instance, and the mission manager sends one message of its own. Measured on the air with the ground station silent, the vehicle's return traffic is **44.7 B/s**, made up of:

| Message | Rate | Cost | Removable |
|---|---|---|---|
| `HEARTBEAT` | 1 Hz | 22 B/s | No, and it is worth keeping |
| `MISSION_CURRENT` | 1 Hz | 18 B/s | No. The mission manager sends it on every channel, independent of stream configuration |
| `STATUSTEXT` | on event | 0 idle | Yes |
| `COMMAND_LONG` | on event | 0 idle | Yes, but it is already idle |

`STATUSTEXT` idles at zero and is not worth keeping on this link. Each message is up to 66 bytes, and PX4 emits several in a burst on arming and on mode changes, which is exactly when losing pose packets matters most. The same text reaches you over USB and ELRS.

Stream rates set with `mavlink stream` do not survive a reboot, so put the command in the startup addon script on the SD card. PX4 runs `/fs/microsd/etc/extras.txt` at boot, after the MAVLink instances have started:

```bash
python3 ~/uas/tools/nsh.py \
  'echo "mavlink stream -d /dev/ttyS2 -s STATUSTEXT -r 0" >> /fs/microsd/etc/extras.txt' \
  'cat /fs/microsd/etc/extras.txt'
```

Apply it to the running system as well, or wait for the next power cycle:

```bash
python3 ~/uas/tools/nsh.py 'mavlink stream -d /dev/ttyS2 -s STATUSTEXT -r 0'
```

**Check.**

```bash
python3 ~/uas/tools/nsh.py 'mavlink status streams'
```

The block for instance on `/dev/ttyS2` should list `HEARTBEAT` at 1.00 Hz and `COMMAND_LONG` as unlimited, and nothing else.

**What you keep.** The heartbeat tells the ground station the vehicle is alive. Waypoint traffic needs no configuration at all: `MISSION_ACK`, `MISSION_REQUEST_INT` and `MISSION_ITEM_REACHED` come from the mission manager directly rather than from a stream, so the standard mission upload handshake works over this link. Upload missions on the ground. An upload is a burst of return traffic, and every packet of it costs pose packets.

---

## Part 6: Send Pose Data from the Ground Station

In normal use `~/uas/tools/mocap_stream.sh` does this for you: it reads the
Vicon stream, converts frames, shares the link between vehicles and broadcasts
flight plans alongside the poses. See
[Using Motion Capture]({% link docs/motion_capture.md %}#part-2-run-the-broadcaster)
for running it. This part documents what it sends and why, which is what you
need in order to debug it or to replace it.

Send **`VISION_POSITION_ESTIMATE`** (message 102). The choice matters more than it looks:

| Message | Wire size | Verdict |
|---|---|---|
| `VISION_POSITION_ESTIMATE` (102) | 44 B | **Use this.** PX4 decodes it to `vehicle_visual_odometry`, the only odometry topic EKF2 subscribes to |
| `ATT_POS_MOCAP` (138) | 48 B | Avoid despite the name. It publishes `vehicle_mocap_odometry`, which EKF2 does not read. It is logged and ignored |
| `ODOMETRY` (331) | 242 B | Does not fit the radio. The quaternion sits mid-message and blocks MAVLink v2 truncation |
| `VISION_SPEED_ESTIMATE` (103) | 28 B | Optional addition if you want to supply velocity rather than let the estimator derive it |

Fill the message as follows:

1. **Position in metres, NED.** Tracker reports millimetres in the room's own frame, which in this lab is ENU, so `north = y`, `east = x`, `down = -z`. See [Frame conventions](#frame-conventions) above for the full derivation including yaw, and keep the conversion in one function so there is one place to be wrong. Test it by walking the vehicle north and watching the sign, not by reading the code again.
2. **Attitude as roll, pitch, yaw in radians.** The vehicle has no magnetometer, so mocap yaw is the only absolute heading available to it, and `EKF2_EV_CTRL` must include the yaw bit or position fusion never starts (Part 8). Compose the full rotation rather than sending heading only with roll and pitch zeroed: zeros are a claim that the vehicle is always level, and they cost nothing to fill in correctly.
3. **Covariance all zeros.** MAVLink v2 truncates trailing zeros, so a zero covariance array costs nothing on the wire and Part 8 sets the noise from parameters instead. Writing NaN into the first element to mean "unknown" costs 4 more bytes and buys nothing.
4. **Send the timestamp, and do not rely on it.** No `TIMESYNC` stream runs on this instance, so the board's clock offset never converges and PX4 stamps each sample with its arrival time. Running timesync over this link would make it converge, at the cost of return traffic in both directions for no gain. Declare the link delay in `EKF2_EV_DELAY` in Part 8 instead.

Rate and load, measured on the course hardware with a 44-byte packet at 8N1:

| Rate | Load on the wire | Measured delivery |
|---|---|---|
| 25 Hz | 10.7 kbps | 99.4 % |
| 50 Hz | 21.5 kbps | 99.4 % |
| 100 Hz | 43.0 kbps | Not recommended, see below |

**The link, not the capture system, is the scarce resource, and it is shared.**
The room's cameras run at 120 Hz, but 21.5 kbps is the load that holds 99.4 % at
20 ms, so that figure is the budget for *all* vehicles together rather than for
each. The broadcaster divides it by however many objects Tracker is currently
tracking, and gives the airtime back when one leaves:

Flight plans share the radio with the poses, so the broadcaster reserves about
4.5 kbps for them before dividing what is left. Measured through
`mocap_stream.sh` against a 120 Hz capture, with that reserve in place:

| Vehicles tracked | Frames per pose | Pose rate each | Poses on the wire |
|---|---|---|---|
| 1 | 4 | 30 Hz | 13.2 kbps |
| 2 | 7 | 17 Hz | 15.1 kbps |
| 3 | 10 | 12 Hz | 15.8 kbps |
| 4 | 13 | 9 Hz | 16.3 kbps |

Add the plan reserve and each row sits just inside the 21.5 kbps budget. The
rate is snapped to a whole number of capture frames, because a pose can only go
out on a frame that arrived, which is why the rates are not exactly the budget
share.

Thirty Hz for one vehicle is well above what a position loop needs. Loss at these
rates is a fixed per-packet probability rather than a bandwidth effect: halving
the rate from 50 to 25 Hz did not change the percentage delivered. Twelve Hz is
usable too. A back-to-back comparison on the course airframe measured the same
hold error and the same altitude hold at 10 Hz as at full rate, with about 7 cm
more lag along a 1 m/s leg.

You do not have to implement this sharing. `mocap_stream.sh` does it, and
reports the rate it has chosen on every change.

Airtime is dominated by per-packet overhead, not payload. The sender's `BC`
counter advanced about 110 bytes per 40-byte packet transmitted, and the
`MT` = 1 experiment in Part 1 began queueing at roughly 43 kbps of nominal load.
Packet rate therefore matters more than packet size when planning for several
vehicles. One accidental run at 116 Hz, which is 51 kbps and well over budget,
still delivered 98.5 % to a single vehicle at short range, so the budget is
conservative for one. It has not been measured with four vehicles spread across
the room, and latency at that load was never characterised. Measure at the rate
and vehicle count you intend to fly before relying on it.

---

## Part 7: Verify the Link End to End

Keep `EKF2_EV_CTRL` at 0 until this part passes. With it at 0 the pose reaches the uORB topic and the estimator ignores it, so the plumbing can be exercised with no possibility of moving the vehicle's estimate.

`~/uas/tools/mocap_tx.py` stands in for the mocap feed, so the link can be
proved before Vicon is involved at all. It generates a synthetic square, sends
it as `VISION_POSITION_ESTIMATE` out the sender radio, and reports what arrived.
It is a bench instrument, not the thing you fly with: the live path is
`mocap_stream.sh` from Part 6. Use this one when you need to know whether the
radios and the serial port are good, with no capture system in the loop.

It defaults to 50 Hz and one vehicle, which is the single-vehicle budget, so the
numbers below are directly comparable to Part 6's table.

```bash
python3 ~/uas/tools/mocap_tx.py --yaw-follow --secs 120
```

With the vehicle on USB it measures latency from the flight controller's own arrival timestamps, mapped into the laptop's clock with MAVLink `TIMESYNC` over USB, so the USB round trip does not bias the figure. Delivery comes from the per-component sequence counts PX4 keeps for the stream. Expected on the course hardware, one vehicle, radios in the same room:

| Quantity | Expected |
|---|---|
| Delivery | 99.4 % (roughly 30 packets lost per 5400) |
| Latency | 20 ms median, 15 ms minimum, 24 ms at the 95th percentile |
| Rate | 50.0 Hz sustained, 21.5 kbps |

Pass `--via usb` to exercise the measurement chain with no radio in it, which separates a tool problem from a link problem.

The residual loss tracks the vehicle's transmit packet rate rather than its byte rate: roughly 2 return packets per second costs about 0.7 % of inbound poses, which is about 3 ms of deafness per vehicle transmission. If your delivery is materially worse than 99 %, the vehicle is transmitting more than Part 5 allows.

---

## Part 8: Enable Fusion in EKF2

A pose source with a wrong sign or a stale value will fly the vehicle into a wall under position control. Complete Part 7 first, keep the vehicle disarmed and props off for the first fused test, and confirm the position the estimator reports matches where the vehicle actually is before arming.

| Parameter | Value | Reason |
|---|---|---|
| `EKF2_EV_CTRL` | 11 (bits 0, 1, 3) | Fuse horizontal position, vertical position and yaw |
| `EKF2_EV_DELAY` | 20 ms | The measured ground-to-vehicle link latency from Part 7 |
| `EKF2_EV_NOISE_MD` | 1 (EV noise parameters) | Take noise from parameters, since the message carries zero covariance |
| `EKF2_EVP_NOISE` | 0.1 m | Position measurement noise. **Leave at the default. Do not lower it**, see below |
| `EKF2_EVA_NOISE` | 0.1 rad | Angle measurement noise |

`EKF2_EV_CTRL` is a bitmask: bit 0 horizontal position, bit 1 vertical position, bit 2 3D velocity, bit 3 yaw. Bit 2 is for sources that report velocity directly, which `VISION_POSITION_ESTIMATE` does not. Leave it clear and let the estimator derive velocity.

**Bit 3, yaw, is not optional on this airframe.** There is no magnetometer
(`SYS_HAS_MAG = 0`), so heading is unobservable without an external source, and
EKF2 will not fuse external vision position until its heading is aligned. Leave
bit 3 out and position fusion silently produces NaN rather than an error.

**Do not lower `EKF2_EVP_NOISE` to "trust mocap more".** It is tempting, because
the measurement really is good to a millimetre, and it is wrong.
`EKF2_EVP_GATE` is expressed in **standard deviations**, so the acceptance gate
shrinks with the noise you declare. Lowering this to 0.02 on the course airframe
collapsed the gate to about +-0.15 m; the estimator then rejected vision during
a climb, and the vehicle flew at 0.1 m while believing it was at 1.0 m. It
nearly crashed twice before the cause was found. The measurement being precise
is not a reason to tell the estimator to reject anything that disagrees with its
prediction.

`EKF2_EV_DELAY` takes effect only after a restart, so power cycle again.

**Check.** On the bench, with the pose stream running and the vehicle disarmed, carry the vehicle a metre north and confirm `vehicle_local_position` follows:

```bash
python3 ~/uas/tools/nsh.py 'listener vehicle_visual_odometry' 'listener vehicle_local_position'
```

The position in `vehicle_visual_odometry` is what arrived over the link. The position in `vehicle_local_position` is what the estimator made of it. If the first is right and the second is not, the problem is in the parameters above. If the first is wrong, the problem is in the frame conversion in Part 6.

**A stale pose must time out, not persist.** If the link drops, the last pose stays valid-looking while the vehicle moves away from it, and a position loop will chase a point the vehicle has already left. Whatever consumes the pose must treat a receive gap of a few frames as invalid rather than holding the last value.

---

## Capacity

Useful throughput saturates around 90 kbps, but latency climbs steeply near that
limit. A single vehicle at 50 Hz offers 21.5 kbps and delivers 99.4 % at 20 ms,
with headroom. Adding vehicles multiplies the packet rate, and because airtime is
dominated by per-packet overhead, the practical ceiling arrives sooner than
payload arithmetic suggests.

This is why 21.5 kbps is treated as a budget for the whole room rather than for
each vehicle, and why `mocap_stream.sh` divides it by the number of objects being
tracked (Part 6). Flight plans ride on the same link, and the broadcaster holds
about 4.5 kbps back for them, so the poses get what is left.

Measure with `mocap_tx.py` at the intended rate and vehicle count before
relying on it, and treat a rise in median latency, not a fall in delivery, as
the signal that the link is saturating. Delivery held at 98.5 % on an accidental
51 kbps run; latency at that load was never measured, and latency is what a
position loop cannot tolerate.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Nothing received at all | `ID`, `HP` or `CM` differs | Compare all three against Part 1 |
| Radio does not respond in XCTU | Another program holds the port | Close other serial tools; only one program can own a port |
| Delivery around 40 %, and traffic on the link with the ground tools stopped | Another program also has the sender radio open and is writing to it. QGroundControl's serial auto-connect grabs the FTDI adapter and talks MAVLink to it | Turn off **Application Settings → AutoConnect** for serial devices, or close QGC. Check with `fuser -v /dev/ttyUSB0` |
| Radio looks dead while another radio is powered | Its serial output carries over-the-air traffic, which collides with the configuration dialogue | Power down other radios while configuring, and pass `--baud` to skip autodetection |
| `rx: 0.0 B/s` on the flight-controller instance | Wrong UART, or the GPS driver holds the port | Part 4: `GPS_1_CONFIG` = 0 and `MAV_2_CONFIG` = 201 |
| `vehicle_visual_odometry: never published` but the radio link tests fine | Pose sent as `ATT_POS_MOCAP` | Send `VISION_POSITION_ESTIMATE`, Part 6 |
| Delivery around 86 % | The vehicle is transmitting its default telemetry on this port | Part 5. Check the instance's transmit rate is near 39 B/s |
| Delivery below 99 % with the return traffic already minimal | Radios too close on the bench, or interference | Separate them by a few metres and repeat |
| Latency in the tens of ms above 20 ms, or heavy jitter | Offered load too high, or `MT` is not 0 | Reduce the rate; confirm `MT` = 0 |
| `rx loss` reports an absurd percentage such as 254 % | The sender is not incrementing MAVLink sequence numbers | In pymavlink, `seq` is incremented by `MAVLink.send()`, not by `msg.pack()`. Increment it yourself if you pack and write the bytes |
| Estimator rejects the pose, or position jumps | Frame conversion, or `EKF2_EV_DELAY` far from the measured latency | [Frame conventions](#frame-conventions), then Part 8 |
| Estimator rejects vision during a climb, and the vehicle flies low while reporting the commanded height | `EKF2_EVP_NOISE` lowered, which shrinks `EKF2_EVP_GATE` with it | Put it back to 0.1 (Part 8). This nearly crashed the course airframe twice |
| Position fusion produces NaN, or EKF2 never accepts vision at all | `EKF2_EV_CTRL` missing bit 3, with no magnetometer to align the heading | Set `EKF2_EV_CTRL` = 11 (Part 8) |
| Delivery around 6 % | `MAV_2_FORWARD` = 1, so the vehicle floods the link with forwarded traffic | Set it to 0 and power cycle (Part 4) |
| Position is right but the vehicle flies at a fixed angle to every command | Tracker object created facing the wrong way | Recreate it with the nose along room +y. There is no software fix ([Frame conventions](#frame-conventions)) |
| Pose rate lower than expected | More vehicles are being tracked, so the link budget is shared | Expected. See the table in Part 6 |
| Parameters saved but nothing changed | `MAV_2_CONFIG`, `MAV_2_MODE` and `EKF2_EV_DELAY` all need a restart | Power cycle. A software reboot leaves this board silent |
