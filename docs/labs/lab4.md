---
layout: default
title: Lab 4
nav_order: 4
parent: Labs
last_modified_at: 2026-09-18 12:00:00 -0400
---

# Lab 4: Velocity and Position Control

**Course:** Uncrewed Aerial Systems  
**Prerequisites:** [Lab 3]({% link docs/labs/lab3.md %}) complete, with your `uas_control` module flying with rate, attitude and altitude loops (`UAS_LOOP_EN = 7`), landing in your own mode; optical flow configured and verified per [Lab 2 Part 12]({% link docs/labs/lab2.md %}#part-12-configure-optical-flow)  
**Estimated Time:** 6 hours across two sessions. Part 2 is a 45-minute
configuration you do once  
**Hardware Required:**
- Your quadrotor, with the MTF-01 optical flow / rangefinder module
- RC transmitter, flashed and bound per [Radio Configuration]({% link docs/radio_configuration.md %})
- 2S LiPo batteries
- **Propellers removed** for all bench work
- Personal laptop with the PX4 toolchain from Lab 2
- Access to the motion-capture room, per [Using Motion Capture]({% link docs/motion_capture.md %})

---

## Overview

At the end of Lab 3 your vehicle holds attitude and height and **drifts**. With the sticks centred it holds itself level and at 1 m, and wanders off across the room, because nothing is closing a loop on where it is going. That was correct behaviour for Lab 3. This lab closes that loop, in two stages:

1. **Velocity**, from optical flow. The MTF-01 under the vehicle watches the floor slide past; EKF2 turns that into a horizontal velocity estimate. Your velocity controller turns a velocity error into a tilt command for the attitude loop you already have. With this loop closed, sticks centred means *stop*, not *hold level*.
2. **Position**, from motion capture. The flow estimate integrates to a position that drifts; a motion-capture system gives ground truth. Your position controller turns a position error into a velocity command for the loop from stage 1. With this loop closed the vehicle holds a point in the room, and can be sent along a trajectory.

The module you built in Lab 3 already has the two files (`VelocityController.cpp`, `PositionController.cpp`) and the two `UAS_LOOP_EN` bits (`15`, `31`) waiting for this. Nothing in the inner loops changes.

> **What is yours to write:** `VelocityController.cpp` (Part 1),
`PositionController.cpp` (Part 3) and `FlightPlan::evaluate()` (Part 4). The
module already handles receiving poses and plans off the radio, selecting
between the two position sources, and translating a plan onto the vehicle.
Nothing in the inner loops from Lab 3 changes.

---

## Part 1: Velocity Control from Optical Flow

**File:** `VelocityController.cpp`

### 1.1 What optical flow gives you, and what it does not

The flow sensor reports how fast the image of the floor moves across its lens, in radians per second, about two axes. On its own that is not a velocity: a fast-moving vehicle high up and a slow one low down produce the same flow rate. EKF2 combines the flow with the **gyro** (to remove the part of the image motion caused by the vehicle rotating) and the **rangefinder** (to scale by height) and fuses the result into its velocity estimate. That is what arrives in your controller as `state.vx` and `state.vy`:

| Field | Frame | Meaning |
|---|---|---|
| `state.vx`, `state.vy` | local **NED** (north, east) | fused horizontal velocity, m/s |
| `state.velocity_valid` | n/a | EKF2's `v_xy_valid`: it currently trusts this estimate |
| `state.yaw` | n/a | heading, needed to relate NED to the vehicle's forward/right |

Three things follow from how the estimate is made, and every one of them will bite you at some point in this lab:

- **It needs texture and light.** A featureless floor, a glossy one, or a dark room gives the sensor nothing to track, and `velocity_valid` goes false. Check the floor before you fly.
- **It needs height, and not too much of it.** The rangefinder scales the flow; below ~0.3 m the ground is too close and too blurred, above ~3 m the flow is too slow to resolve. The MTF-01 is happiest between 0.5 and 2 m.
- **It is a velocity, integrated into a position that drifts.** `state.x`, `state.y` exist and are dead-reckoned from this velocity. Over a minute they can be off by metres. That is why the position loop waits for motion capture in Part 2.

### 1.2 Verify the estimate before you close a loop on it

A velocity loop closed on a wrong-signed estimate is positive feedback: the vehicle accelerates into the wall and every correction makes it worse. Before writing any control code, prove the estimate is right.

1. `UAS_LOOP_EN` unchanged from Lab 3 (`7`). Hover in Stabilized at about 1 m over a textured floor.
2. Push the vehicle **forward** with the stick for a second, then **right**, then land.
3. In the log, plot `vehicle_local_position.vx`, `.vy` alongside the attitude. Moving forward while pointed north must show positive `vx`; moving right, positive `vy`. If the vehicle was pointed elsewhere, rotate accordingly. The simplest approach is to take off pointed north.
4. Also plot `v_xy_valid`. It should be true for the whole hover. Where it drops, note the height and what the floor looked like.

If a sign is wrong, the usual culprit is the flow sensor's mounting orientation (`SENS_FLOW_ROT`), not your code. Fix it there; do not compensate in the controller.

> **A check that appears to work but does not.** It is tempting to verify the sign from any flight log, on the reasoning that flow velocity should change in the direction the vehicle tilts. It does so *even when the sensor is mounted backwards*, because the EKF's velocity is driven by the accelerometers between flow updates, and that test only sees the accelerometers. The instructor's vehicle passed it with `SENS_FLOW_ROT` 180° wrong, and three flights "held velocity" in the log while sliding across the room. The check that catches it is the one above: push the vehicle and look at `vx`/`vy`. Alternatively, from a log, compare the *raw* flow against the EKF: in `estimator_aid_src_optical_flow`, the observation and the EKF's prediction (`observation + innovation`) must correlate **positively**; negative means reversed, and a rejection rate above ~30 % is the same symptom. A reversed flow also makes EKF2 re-align its heading mid-flight, which a yaw-hold loop will then chase. On the instructor's vehicle this produced a crash after a 196° reset.

### 1.3 Implement

The template walks you through it in two steps.

**Step 1: rotate the error into the body frame.** `sp.vx`, `sp.vy` and `state.vx`, `state.vy` are all NED. Your tilt commands are forward/right. Compute the velocity error in NED and rotate it by `state.yaw`:

```
error_forward =  error_n * cos(yaw) + error_e * sin(yaw)
error_right   = -error_n * sin(yaw) + error_e * cos(yaw)
```

Skip this and the loop works only while the vehicle points north; yaw it 90° and it corrects sideways.

**Step 2: PID each body-frame error into a tilt angle.** For small angles the horizontal acceleration is proportional to tilt, so a PID whose output you *read as an angle* is fine at these speeds. Then the signs, which are the part that bites:

- To accelerate **forward** the nose goes **down**, which is **negative pitch**: `sp.pitch` takes the *opposite* sign of `error_forward`.
- To accelerate **right** the vehicle rolls **right**, which is **positive roll**: `sp.roll` takes the *same* sign as `error_right`.

Clamp both to `±_max_tilt` (`UAS_MAX_TILT`). Start with **P only** and a small gain; a velocity loop that commands 20° for a 1 m/s error is already aggressive. Add D (on measured velocity, not error, for the same reason as in the rate loop) if it overshoots, and a little I only if it settles with a steady drift. Respect `velocity_valid`: the template's guard commands level and resets when the estimate is not trusted. Keep it.

### 1.4 A switch for the loops, and what the sticks do now

PX4 has exactly one Offboard mode, so there is no second flight-mode slot for "Offboard with the velocity loop". The right switch you set up in [Lab 3 Part 4]({% link docs/labs/lab3.md %}#part-4-assign-your-controller-to-the-mode-switch) already does this job: the left switch selects Stabilized, Position or Offboard, and the right switch selects how much of your cascade runs while you are in Offboard. All this part changes is what the top position means, by raising `UAS_LOOP_EN` from 7 to 15:

| Switch | Loops | `UAS_LOOP_EN` equivalent |
|---|---|---|
| down | rate + attitude | 3 |
| centre | + altitude | 7 |
| up | + velocity | `UAS_LOOP_EN`, now 15 |

You can move it in flight. A loop switched on mid-air starts from the vehicle's current state (the altitude loop latches the current height, the velocity loop starts clean), so stepping up is smooth; stepping down hands you back the simpler behaviour instantly. This is the same switch the gain-tuning feature (`UAS_TUNE_SEL`) uses, so leave that at 0 while the loop switch is on.

**The velocity loop only runs when the flow can actually see.** The module enables it only while airborne (after the automatic take-off, Lab 3 Part 8.3), above 0.2 m, with `velocity_valid` true. On the ground, during take-off, over a bad patch of floor or too high, it drops the velocity (and position) loop and the sticks go back to commanding tilt, exactly as in the centre position. QGC shows `no flow estimate, sticks are tilt`, and `flow valid, velocity/position loops on` when it comes back. Two consequences: you can **arm and take off with the switch up**, because it is the same automatic take-off and the velocity loop joins as the vehicle passes 0.2 m on its way to the `UAS_TKO_ALT` hover. Second, a stick held for *velocity* becomes, for the duration of a dropout, a *tilt* of the same fraction. Centre the sticks when you hear the message. Do not trust PX4's `v_xy_valid` on the ground: it stayed true on the instructor's vehicle while the estimated velocity drifted to 0.7 m/s with the vehicle sitting still.

With the velocity loop running, the roll/pitch sticks command **velocity in the heading frame**, up to `UAS_MAX_VXY` (default 1 m/s) at full deflection; the module rotates that into NED for you. Sticks centred means zero velocity, and the loop actively stops the vehicle. Throttle and yaw sticks are unchanged from Lab 3.

### 1.5 Bench Test

Props off, loop switch **up** (or `UAS_LOOP_EN = 15`), arm in Offboard, sticks centred. The velocity setpoint is zero, so any velocity you impose by hand should produce a tilt command *opposing* it.

1. `uas_control status`: confirm `velocity … valid: yes`. If it is `NO` on the bench that is normal (no flow on a static floor at 5 cm); lift the vehicle to ~0.5 m over a textured surface and it should come good within a second. Do the rest of the test at that height.
2. Carry the vehicle **forward** at walking pace, pointed north: `listener vehicle_attitude_setpoint` (or the log) should show a **nose-up** (positive pitch) command, which is the loop acting to slow the vehicle.
3. Carry it **right**: a **roll-left** (negative roll) command.
4. Turn the vehicle to face **east** and repeat step 2. The command must still be nose-up. If it becomes a roll command, your Step 1 rotation is missing or reversed.

### 1.6 Flight Test

Take off in Stabilized, hand over at a hover as in Lab 3 with the loop switch at **centre** (altitude), and confirm it still behaves. Then, in the hover, move the switch **up**.

- **Sticks centred:** the Lab 3 drift should stop. Expect a gentle correction as the loop catches the initial velocity, then a hover that stays within a metre or so, wandering slowly as the flow estimate breathes.
- **A slow, growing sway** (period of a few seconds) is the classic velocity-loop failure: too much P for the lag in the estimate. Halve it. Flip to Stabilized if it grows past a couple of metres of travel.
- **Stick inputs** should feel like steering a velocity: push forward, it accelerates to a speed and holds it; release, it stops.
- **The stick convention changes.** In attitude mode a drift is arrested by tilting against it, with the stick opposite the motion. In velocity mode that same stick means "go the other way at up to `UAS_MAX_VXY`", and the vehicle will. **Centring the stick is the brake.** On the first velocity-loop flight of the course airframe this produced a rapid departure: a backward drift, a stick pushed back, and the loop delivering the commanded −1 m/s.
- **Gains that are too soft feel like no loop at all.** A P gain of 0.12 rad per m/s answers a 0.25 m/s drift with 1.5° of tilt, which you will not notice working. PX4 flies this airframe at `MPC_XY_VEL_P_ACC` = 1.8 m/s² per m/s, which is 0.18 rad per m/s once you divide by *g*, with an integrator ten times larger than instinct suggests. Convert PX4's gains before deciding yours are wrong.
- **Over a bad patch of floor** `velocity_valid` will drop and the module hands you an attitude-hold vehicle (sticks are tilt) until it returns (see 1.4). Learn what that looks like.

### 1.7 Deliverable

From one flight: `vx`/`vy` setpoint against measured, the roll/pitch commands the loop produced, and `v_xy_valid`, over a window that includes sticks-centred hover and at least one commanded velocity step. Plus a comparison of horizontal wander with `UAS_LOOP_EN = 7` and `15` over the same duration.

---

## Part 2: Motion Capture

Your velocity loop stops the vehicle drifting, but it still does not know
*where* it is: the flow estimate integrates to a position that walks away at a
few centimetres a second. This part gives the vehicle a position it can trust.

The room procedure, creating your object in Vicon Tracker, running the
broadcaster and loading plans, is in
[Using Motion Capture]({% link docs/motion_capture.md %}). Do that first. This
part is the configuration your airframe needs.

### 2.1 Two position sources that do not mean the same thing

| Source | Where it comes from | What it is good for |
|---|---|---|
| Optical flow | downward camera, fused by EKF2 | velocity. Position only as a drifting integral |
| Motion capture | the room's cameras, over the XBee | position in the room, to a centimetre |
| Rangefinder | downward laser | height above whatever is under it |

The two horizontal sources differ in a way that matters to your controller. The
flow integral's origin is wherever the loop was switched on, so zero means
"here". Motion capture reports the room's frame, so zero is a *place*, and it is
usually metres away. A plan can only name a point in the second. `state.x` and
`state.y` carry whichever is selected, and `state.mocap` carries the room's own
measurement untouched by the estimator.

### 2.2 Your vehicle's identity on the shared link

Every vehicle in the room hears every pose and every plan, because the messages
carry no destination field. `UAS_MOCAP_ID` is the only thing that makes your
airframe ignore everyone else's.

**Set it to your vehicle's number. Getting it wrong means flying to another
team's position, or flying another team's plan.** The numbers are in
[Using Motion Capture]({% link docs/motion_capture.md %}#what-to-name-it).

### 2.3 Module parameters

```
param set UAS_MOCAP_ID   <your number>
param set UAS_LOOP_EN    31
param set UAS_MODE_SLOT  14
param set UAS_SRC_SLOT   5
param set UAS_ALT_SRC    2
param set UAS_POS_SRC    1
param set UAS_PLAN_EN    1
param set UAS_PLAN_REL   1
param save
```

| Parameter | Value | Why |
|---|---|---|
| `UAS_LOOP_EN` | 31 | full cascade: rate, attitude, altitude, velocity, position |
| `UAS_MODE_SLOT` | 14 | your module takes over in Offboard |
| `UAS_SRC_SLOT` | 5 | mode slots at or above 5 select motion capture, so slot 4 is flow and slot 6 is mocap |
| `UAS_ALT_SRC` | 2 | altitude follows the same switch as position |
| `UAS_POS_SRC` | 1 | ignored while `UAS_SRC_SLOT` is non-zero. Set for clarity |
| `UAS_PLAN_EN` | 1 | accept flight plans (Part 4) |
| `UAS_PLAN_REL` | 1 | fly a plan from wherever you are, so it repeats from any start |

Limits. Start here and raise them only once the loops behave:

```
param set UAS_MAX_VXY  1.0
param set UAS_MAX_VZ   0.5
param set UAS_MAX_ALT  2.0
param save
```

### 2.4 Flight-mode switch

**Arming is refused in Offboard.** Without a Stabilized position on the switch
you cannot arm at all, and you have no abort path.

PX4 divides the mode channel into six slots, and a three-position switch reaches
slots 1, 4 and 6. Lab 3 set the left switch up already; this part uses the two
Offboard positions to select the position source:

| Parameter | Value |
|---|---|
| `COM_FLTMODE1` | 8 (Stabilized) |
| `COM_FLTMODE4` | 7 (Offboard) |
| `COM_FLTMODE6` | 7 (Offboard) |

| Switch | Slot | Mode | Horizontal source | Altitude source |
|---|---|---|---|---|
| **DOWN** | 1 | Stabilized | PX4's own controller | PX4's own |
| **MIDDLE** | 4 | Offboard, your module | optical flow | rangefinder |
| **UP** | 6 | Offboard, your module | **motion capture** | **motion capture** |

**DOWN is your abort.** Arm there, take off there, and return there if anything
surprises you.

Verify the mapping with the propellers off before you fly:

```
python3 ~/uas/tools/nsh.py 'listener manual_control_switches'
```

Move the switch and confirm `mode_slot` reads 1, 4 and 6. If it reads something
else, your transmitter is sending a different range and the table above will not
do what it says.

### 2.5 Estimator parameters

```
param set EKF2_EV_CTRL     11
param set EKF2_HGT_REF     3
param set EKF2_EV_DELAY    20
param set EKF2_EV_NOISE_MD 1
param set EKF2_EVP_NOISE   0.1
param set EKF2_GPS_CTRL    0
param set EKF2_OF_CTRL     1
param set EKF2_RNG_CTRL    1
param save
```

**`EKF2_EV_CTRL` and `EKF2_HGT_REF` need a power cycle.** Pull the battery and
reconnect. A software reboot leaves this board silent.

| Parameter | Value | Why |
|---|---|---|
| `EKF2_EV_CTRL` | 11 | fuse external vision: horizontal position (bit 0), vertical position (bit 1), yaw (bit 3) |
| `EKF2_HGT_REF` | 3 | vision is the height reference |
| `EKF2_EV_DELAY` | 20 | ms. The measured median latency of the radio link |
| `EKF2_EV_NOISE_MD` | 1 | take vision noise from parameters, not from the message |
| `EKF2_EVP_NOISE` | 0.1 | m. **Do not lower this to "trust mocap more"**, see below |
| `EKF2_GPS_CTRL` | 0 | no GPS indoors |
| `EKF2_OF_CTRL` | 1 | keep optical flow fused. It carries your velocity estimate |
| `EKF2_RNG_CTRL` | 1 | keep the rangefinder for EKF2's terrain estimate, which optical flow needs |

**Why bit 3, yaw, is not optional.** This airframe has no magnetometer
(`SYS_HAS_MAG = 0`), so heading is unobservable without an external source. EKF2
will not fuse external vision position until its heading is aligned, and with no
magnetometer the only thing that can align it is the vision yaw. Leave bit 3 out
and position fusion silently produces NaN rather than an error.

**Why not to tighten `EKF2_EVP_NOISE`.** `EKF2_EVP_GATE` is expressed in standard
deviations, so tightening the noise tightens the rejection gate with it. Lowering
this to 0.02 on the course airframe collapsed the gate to +-0.15 m, the estimator
locked vision out during a climb, and the vehicle flew at 0.1 m while believing
it was at 1.0 m. It nearly crashed twice before the cause was found.

**Optical flow still has to work.** The module enables the velocity and position
loops only while `velocity_valid` is true, and with `EKF2_EV_CTRL = 11` the
vision velocity is not fused, so that flag is carried by flow. Motion capture
supplies position; flow still supplies velocity. This is why `EKF2_OF_CTRL` stays
at 1 rather than being only a fallback.

### 2.6 Radio link

The XBee carries the poses to your vehicle on the GPS 1 port, so that port is a
MAVLink link rather than a GPS. Configured per
[XBee Configuration]({% link docs/xbee_configuration.md %}):

```
param set GPS_1_CONFIG   0
param set MAV_2_CONFIG   201
param set SER_GPS1_BAUD  115200
param set MAV_2_MODE     1
param set MAV_2_FORWARD  0
param save
```

**`MAV_2_MODE` and `MAV_2_FORWARD` are not cosmetic.** The radio is half duplex,
so anything the vehicle transmits deafens its own receiver. Mode 1 holds the
return traffic to a heartbeat. With `MAV_2_FORWARD = 1` the vehicle forwards
everything it receives on other links out of this one, which measured 6531 bytes
per second and dropped pose delivery from 99 % to about 6 %. It looks exactly
like a hardware fault.

Needs a power cycle.

### 2.7 Confirm it before you fly

Run the checks in
[Using Motion Capture]({% link docs/motion_capture.md %}#part-3-check-your-vehicle-is-receiving),
including all five hand-carried direction checks. **Do not skip them.** A heading
error puts every position correction in a frame rotated from reality, so the
vehicle accelerates away from its setpoint instead of toward it. Two of the five
were wrong when this was first set up, and each would have destroyed the vehicle
on its first hover.

### 2.8 Deliverable

A short log excerpt with the vehicle carried by hand through a known path,
showing `vehicle_visual_odometry` position against the path you walked, plus the
measured pose rate and the fraction of poses delivered.

---

## Part 3: Position Control

**File:** `PositionController.cpp`

### 3.1 Implement

The outermost loop, and the simplest. Position error becomes a velocity command
for the loop you built in Part 1.

- **P only.** An integrator on position is almost always a mistake: the velocity
  loop below already has one, and two integrators in series produce a slow
  oscillation that is hard to attribute.
- **Stay in NED.** Both the error and the output are NED. Do **not** rotate into
  the body frame here; the velocity controller below you already does that
  rotation, and doing it twice gives a vehicle that circles its target instead
  of converging on it.
- **Clamp the magnitude, not the axes.** Clamping `vx` and `vy` separately
  distorts the direction of travel, so a diagonal move heads somewhere other
  than where you pointed it.

Starting gain, measured on the course airframe: `UAS_POS_P` = 0.9, with
`UAS_VEL_P` = 0.4 under it. The velocity gain matters more than instinct
suggests: 0.2 left the vehicle lagging the target by tens of centimetres on
every leg.

### 3.2 Altitude from motion capture

Set `UAS_ALT_SRC = 2` in Part 2.3, which puts the altitude source on the same
switch as the position source. In the up position your height comes from the
pose rather than the rangefinder.

**Why the rangefinder is not good enough in this room.** The floor is laid with
rubber tiles in several colours, and an optical rangefinder returns a different
range from each. Measured on the course airframe, the error jumped between
+0.06 and +0.34 m within a single one-metre square: a vehicle commanded to
1.00 m sat at 1.24 m, climbed over the black tiles and sank over the coloured
ones. That is a property of the surface, not a calibration, and no scale or
offset fixes it. Reading the pose directly took the true height to 1.00 m and cut
the bob from 0.124 m to 0.018 m standard deviation.

**Which velocity to damp against.** Your altitude loop from
[Lab 3 Part 8]({% link docs/labs/lab3.md %}#part-8-altitude-controller) damps on
`state.vz`, which comes from EKF2 and blends every height sensor it has. On the
course airframe it correlates about 0.2 with the vehicle's real vertical motion,
and damping against a signal that wrong puts its error straight back into the
thrust. `state.mocap.vz` is differentiated from a pose whose noise is under a
millimetre. Switching the D term to it, while motion capture is selected, is the
difference between holding a centimetre and holding five. Make that change and
measure both.

### 3.3 Bench Test

`UAS_BENCH = 1` runs the outer loops on the ground against the real pose stream,
so you can test the position loop without flying.

> **`UAS_BENCH = 1` means the outer loops run with the vehicle on the floor.
> PROPELLERS OFF. Set it back to 0 before you fly.** The module prints a warning
> every time you enter the mode with it on.

1. `uas_control status` shows `mocap OK` and sensible room coordinates.
2. Arm in Offboard with the switch **up**, propellers off. The position setpoint
   latches where the vehicle is.
3. Carry the vehicle **north** by half a metre. The commanded velocity should be
   **south**, back toward the latch, and should grow with the distance.
4. Carry it **east**. The commanded velocity should be **west**.
5. Return it to the latch point. The commanded velocity should fall to near zero.
6. Confirm the commanded speed never exceeds `UAS_MAX_VXY` however far you carry
   it.

### 3.4 Flight Test

Take off in Stabilized, hand over at a hover with the switch at **middle**
(flow), confirm Part 1 still behaves, then move the switch **up**.

- **At the moment of the switch** the vehicle should hold the point it is at. The
  module re-latches on a source change, because carrying the old setpoint across
  the frame change would be a step input of exactly the distance to the room
  origin. You should see `position source -> motion capture, holding here` in
  QGC.
- **Sticks centred:** expect the wander of Part 1 to stop dead. The course
  airframe held a point to 6 cm median, 11 cm at the ninetieth percentile.
- **Sticks** now walk the *target* around at up to `UAS_MAX_VXY`, rather than
  commanding a velocity directly. Centring them leaves the target where it is and
  the vehicle converges on it.
- **Flip back to middle** and the vehicle returns to the flow frame, re-latching
  at its current position. Practise this handover before you need it.
- **If the pose stream stops**, EKF2's horizontal position goes invalid and the
  module drops the position loop, leaving you the velocity loop on flow. Expect
  the message, not a fall.

### 3.5 Deliverable

From one flight: commanded against measured position for both sources over the
same duration, with the handover visible, and the hold error quoted as median and
ninetieth percentile with the sticks centred. Plus the altitude hold from 3.2
with both choices of damping signal, as a standard deviation about the commanded
height.

---

## Part 4: Trajectories

**File:** `FlightPlan.cpp`

A plan is a list of timed waypoints. The module receives one over the radio and
holds it; your job is to execute it, turning the time since the plan started into
the target for this instant.

### 4.1 What is provided, and what is yours

Provided: assembly of the items off the link (`ingest()`), the translation of the
plan onto the vehicle (`start()`), and the start conditions. Yours:
`evaluate()`, which is the executor.

The file format and how to load a plan are in
[Using Motion Capture]({% link docs/motion_capture.md %}#part-4-load-a-flight-plan).

### 4.2 Implement

`FlightPlan::evaluate()` has the full specification in its TODO. Five pieces:

1. **The two held ends.** Before the first item's time and after the last, hold
   that end item with zero feedforward. Hold rather than extrapolate, or a plan
   whose last point is away from the floor has the vehicle fly on past it.
2. **The leg lookup.** Find the pair with `a.t <= t < b.t`. Times are guaranteed
   to increase strictly, so the span is never zero.
3. **The interpolation.** The straight line between the two, in `x`, `y` and
   `alt`. This is what makes the item times set the speed.
4. **The heading**, with two traps. Wrap the difference before interpolating, or
   a leg from 170 to -170 degrees becomes a 340 degree turn instead of 20. And
   only write `sp.yaw` when both headings are finite, because a NaN yaw becomes a
   NaN attitude setpoint.
5. **The feedforward.** The leg's own velocity. Without it the position loop has
   to build an error before it commands any speed, so the vehicle lags the target
   all the way round and the corners come out rounded.

> **On the NaN.** The module checks the plan's output before anything downstream
> sees it, so a non-finite setpoint abandons the plan with
> `plan produced a non-finite setpoint, abandoned, holding` rather than reaching
> the motors. That guard exists because this exact bug, a running plan restoring
> NaN headings, zeroed the thrust and dropped the course airframe 0.93 m. You
> will see a stopped plan. Do not rely on it; get the finiteness check right in
> your own code.

### 4.3 Bench Test

With `UAS_BENCH = 1` and **propellers off**, the plan runs on the ground.

1. Load a square: `plan_load.sh <OBJECT> --size 1.0 --alt 1.0`.
2. `uas_control status` should show the plan held, with its id, item count and
   duration.
3. Arm in Offboard, switch **up**. `status` should show the plan running and the
   leg number advancing.
4. Watch `vehicle_local_position_setpoint` in the log. The commanded position
   should trace the square at the speed the item times imply, and the commanded
   altitude should hold.
5. Confirm the setpoint stops at the last point rather than continuing.
6. Flip out of motion capture and back in. The plan should restart from the
   beginning.

Getting the square right on the bench is most of the work. Do not take an
unverified executor into the air.

### 4.4 Flight Test

Set `UAS_BENCH = 0`. Take off in Stabilized, hover, hand over at **middle**, then
switch **up**. The plan starts on the switch.

- Expect it to climb to the plan's altitude, hold briefly, fly the square and
  hover at the end.
- **Flip to middle at any time to abandon the plan.** Moving a stick also
  abandons it, which is deliberate: the pilot taking hold of the vehicle must
  always win.
- Flip out and back to run it again. `UAS_PLAN_REL = 1` flies it from wherever
  the vehicle is, so you do not have to return to a particular spot.
- The course airframe flew a 1 m square at 0.75 m/s with 18 cm of leg error at
  40 Hz. Corner rounding is the feedforward and the velocity gain, not the
  position gain.

### 4.5 Deliverable

A 3D trace of at least two circuits of your plan flown from different starting
points, with the commanded setpoint overlaid, plus the leg error and the altitude
hold quoted as numbers. Comment on where the vehicle departs from the commanded
path and why.

---

## Lab Deliverables

1. **Source code:** your team's `uas_control` fork, pushed, with completed
   `VelocityController.cpp`, `PositionController.cpp` and `FlightPlan::evaluate()`.
2. **Bench test evidence:** the plots and status output from Parts 1.5, 3.3 and
   4.3.
3. **Flight logs:** your best flight from each of Parts 1.6, 3.4 and 4.4.
4. **Written analysis (3-4 pages):**
   - The cascade you now have, and the bandwidth separation between the position,
     velocity and attitude loops. Justify your gain ratios with measurements
   - Optical flow against motion capture as a position source: what each one
     measures, how each one fails, and what you observed of both
   - Your altitude hold with `state.vz` and with `state.mocap.vz` as the damping
     signal, with the measured standard deviations, and an explanation of why the
     difference is as large as it is
   - Your trajectory tracking error, separated into what the feedforward
     accounts for and what the position loop has to correct

---

## Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| `velocity … valid: NO` on the bench | No flow on a static floor at 5 cm | Normal. Lift to ~0.5 m over texture |
| `velocity_valid` drops in flight | Featureless or glossy floor, too low, too high, poor light | Fly 0.5–2 m over a textured surface; check `estimator_status_flags` `cs_opt_flow` |
| Vehicle accelerates away when the loop is enabled | Velocity estimate sign wrong, or tilt sign wrong in your controller | Part 1.2 first (`SENS_FLOW_ROT`), then the signs in 1.3 |
| Corrects sideways when yawed | Missing body-frame rotation | Part 1.3 Step 1 |
| Slow growing sway | Velocity P too high for the estimate's lag | Halve `UAS_VEL_P` |
| Sticks do nothing in velocity mode | `UAS_MAX_VXY` tiny, or loop not enabled | `uas_control status` should say `loops enabled: 0x0f`; check the switch is up and `UAS_LOOP_SW` names its AUX channel |
| Switch is up but `loops enabled` shows `0x07 (selected 0x0f)` | No velocity estimate, module stepped down | Normal on the ground and over bad floor (1.4). Lift to ~0.5 m over texture |
| `rangefinder rejected … altitude on vz` in QGC | Rangefinder past its reach (outdoors, ~1.5 m over grass) or a large terrain step | Descend; the sample is re-accepted when it agrees again. Keep `UAS_MAX_ALT` at 1.5 m outdoors |
| `mocap STALE` in `uas_control status` | No poses arriving | [Using Motion Capture]({% link docs/motion_capture.md %}#troubleshooting). Check `UAS_MOCAP_ID` matches your vehicle |
| Position fusion produces NaN, or EKF2 never accepts vision | `EKF2_EV_CTRL` missing bit 3 | Set it to 11. With no magnetometer, vision yaw is the only thing that can align the heading (2.5) |
| Vehicle holds 0.1 m while believing it is at 1.0 m | `EKF2_EVP_NOISE` lowered, collapsing `EKF2_EVP_GATE` with it | Put it back to 0.1. Do not tighten it (2.5) |
| Pose delivery around 6 % instead of 99 % | `MAV_2_FORWARD = 1` flooding the half-duplex link | Set it to 0 and power cycle (2.6) |
| Vehicle climbs over dark floor tiles and sinks over light ones | Altitude coming from the rangefinder | `UAS_ALT_SRC = 2` and fly with the switch up (3.2) |
| Vehicle lurches a metre when you switch to motion capture | Setpoint carried across the frame change | The module re-latches on a source change. Check for `position source -> motion capture, holding here`; if it is absent, `UAS_SRC_SLOT` is not set (2.3) |
| Plan held but never starts | Mode switch not on motion capture, or a stick is off centre | A stick move abandons a plan deliberately. Flip out and back in to restart |
| `plan produced a non-finite setpoint, abandoned, holding` | Your `evaluate()` emitted a NaN, usually an unguarded yaw | Part 4.2, trap 4 |
| Vehicle flies the square but rounds every corner | Missing or weak leg feedforward, or velocity P too low | Part 4.2, piece 5. The course airframe needed `UAS_VEL_P` = 0.4 |
| Outer loops will not engage at all | `velocity_valid` false, so flow is not working | Motion capture gives position, not velocity. Fix the flow first (1.2, and 2.5) |
