---
layout: default
title: Lab 4
nav_order: 4
parent: Labs
last_modified_at: 2026-09-18 12:00:00 -0400
---

# Lab 4: Velocity and Position Control

**Course:** Uncrewed Aerial Systems  
**Prerequisites:** [Lab 3]({% link docs/labs/lab3.md %}) complete — your `uas_control` module flying with rate, attitude and altitude loops (`UAS_LOOP_EN = 7`), landing in your own mode; optical flow configured and verified per [Lab 2 Part 12]({% link docs/labs/lab2.md %}#part-12-configure-optical-flow)  
**Estimated Time:** TBD  
**Hardware Required:**
- Your quadrotor, with the MTF-01 optical flow / rangefinder module
- RC transmitter, flashed and bound per [Radio Configuration]({% link docs/radio_configuration.md %})
- 2S LiPo batteries
- **Propellers removed** for all bench work
- Personal laptop with the PX4 toolchain from Lab 2

---

## Overview

At the end of Lab 3 your vehicle holds attitude and height and **drifts**. With the sticks centred it holds itself level and at 1 m, and wanders off across the room, because nothing is closing a loop on where it is going. That was correct behaviour for Lab 3. This lab closes that loop, in two stages:

1. **Velocity**, from optical flow. The MTF-01 under the vehicle watches the floor slide past; EKF2 turns that into a horizontal velocity estimate. Your velocity controller turns a velocity error into a tilt command for the attitude loop you already have. With this loop closed, sticks centred means *stop*, not *hold level*.
2. **Position**, from motion capture. The flow estimate integrates to a position that drifts; a motion-capture system gives ground truth. Your position controller turns a position error into a velocity command for the loop from stage 1. With this loop closed the vehicle holds a point in the room, and can be sent along a trajectory.

The module you built in Lab 3 already has the two files (`VelocityController.cpp`, `PositionController.cpp`) and the two `UAS_LOOP_EN` bits (`15`, `31`) waiting for this. Nothing in the inner loops changes.

> **Scope note:** Part 1 (velocity from optical flow) is written. Parts 2 onward are placeholders and will be filled in as the lab is developed.

---

## Part 1: Velocity Control from Optical Flow

**File:** `VelocityController.cpp`

### 1.1 What optical flow gives you, and what it does not

The flow sensor reports how fast the image of the floor moves across its lens, in radians per second, about two axes. On its own that is not a velocity: a fast-moving vehicle high up and a slow one low down produce the same flow rate. EKF2 combines the flow with the **gyro** (to remove the part of the image motion caused by the vehicle rotating) and the **rangefinder** (to scale by height) and fuses the result into its velocity estimate. That is what arrives in your controller as `state.vx` and `state.vy`:

| Field | Frame | Meaning |
|---|---|---|
| `state.vx`, `state.vy` | local **NED** (north, east) | fused horizontal velocity, m/s |
| `state.velocity_valid` | — | EKF2's `v_xy_valid`: it currently trusts this estimate |
| `state.yaw` | — | heading, needed to relate NED to the vehicle's forward/right |

Three things follow from how the estimate is made, and every one of them will bite you at some point in this lab:

- **It needs texture and light.** A featureless floor, a glossy one, or a dark room gives the sensor nothing to track, and `velocity_valid` goes false. Check the floor before you fly.
- **It needs height, and not too much of it.** The rangefinder scales the flow; below ~0.3 m the ground is too close and too blurred, above ~3 m the flow is too slow to resolve. The MTF-01 is happiest between 0.5 and 2 m.
- **It is a velocity, integrated into a position that drifts.** `state.x`, `state.y` exist and are dead-reckoned from this velocity. Over a minute they can be off by metres. That is why the position loop waits for motion capture in Part 2.

### 1.2 Verify the estimate before you close a loop on it

A velocity loop closed on a wrong-signed estimate is positive feedback: the vehicle accelerates into the wall and every correction makes it worse. Before writing any control code, prove the estimate is right.

1. `UAS_LOOP_EN` unchanged from Lab 3 (`7`). Hover in Stabilized at about 1 m over a textured floor.
2. Push the vehicle **forward** with the stick for a second, then **right**, then land.
3. In the log, plot `vehicle_local_position.vx`, `.vy` alongside the attitude. Moving forward while pointed north must show positive `vx`; moving right, positive `vy`. If the vehicle was pointed elsewhere, rotate accordingly — or simplest, take off pointed north.
4. Also plot `v_xy_valid`. It should be true for the whole hover. Where it drops, note the height and what the floor looked like.

If a sign is wrong, the usual culprit is the flow sensor's mounting orientation (`SENS_FLOW_ROT`), not your code. Fix it there; do not compensate in the controller.

> **A trap, learned the hard way.** You might think any flight log proves the sign: the flow velocity should change in the direction the vehicle tilts. It does — *even when the sensor is mounted backwards* — because the EKF's velocity is driven by the accelerometers between flow updates, and that test only sees the accelerometers. The instructor's vehicle passed it with `SENS_FLOW_ROT` 180° wrong, and three flights "held velocity" in the log while sliding across the room. The check that catches it is the one above — push the vehicle and look at `vx`/`vy` — or, from a log, compare the *raw* flow against the EKF: in `estimator_aid_src_optical_flow`, the observation and the EKF's prediction (`observation + innovation`) must correlate **positively**; negative means reversed, and a rejection rate above ~30 % is the same symptom. A reversed flow also makes EKF2 re-align its heading mid-flight, which a yaw-hold loop will then chase — the instructor's crashed on a 196° reset.

### 1.3 Implement

The template walks you through it in two steps.

**Step 1 — rotate the error into the body frame.** `sp.vx`, `sp.vy` and `state.vx`, `state.vy` are all NED. Your tilt commands are forward/right. Compute the velocity error in NED and rotate it by `state.yaw`:

```
error_forward =  error_n * cos(yaw) + error_e * sin(yaw)
error_right   = -error_n * sin(yaw) + error_e * cos(yaw)
```

Skip this and the loop works only while the vehicle points north; yaw it 90° and it corrects sideways.

**Step 2 — PID each body-frame error into a tilt angle.** For small angles the horizontal acceleration is proportional to tilt, so a PID whose output you *read as an angle* is fine at these speeds. Then the signs, which are the part that bites:

- To accelerate **forward** the nose goes **down**, which is **negative pitch**: `sp.pitch` takes the *opposite* sign of `error_forward`.
- To accelerate **right** the vehicle rolls **right**, which is **positive roll**: `sp.roll` takes the *same* sign as `error_right`.

Clamp both to `±_max_tilt` (`UAS_MAX_TILT`). Start with **P only** and a small gain; a velocity loop that commands 20° for a 1 m/s error is already aggressive. Add D (on measured velocity, not error — same argument as the rate loop) if it overshoots, and a little I only if it settles with a steady drift. Respect `velocity_valid`: the template's guard commands level and resets when the estimate is not trusted. Keep it.

### 1.4 A switch for the loops, and what the sticks do now

PX4 has exactly one Offboard mode, so there is no second flight-mode slot for "Offboard with the velocity loop". Instead, put your spare 3-position switch to work: map it to an AUX channel (`RC_MAP_AUX1` = its channel number, found the same way as in Lab 2) and set `UAS_LOOP_SW = 1`. The mode switch still selects Offboard / Stabilized / Altitude; the spare switch selects how much of your cascade runs while in Offboard:

| Switch | Loops | `UAS_LOOP_EN` equivalent |
|---|---|---|
| down | rate + attitude | 3 |
| centre | + altitude | 7 |
| up | + velocity | 15 |

You can move it in flight. A loop switched on mid-air starts from the vehicle's current state (the altitude loop latches the current height, the velocity loop starts clean), so stepping up is smooth; stepping down hands you back the simpler behaviour instantly. This is the same switch the gain-tuning feature (`UAS_TUNE_SEL`) uses — leave that at 0 while the loop switch is on.

**When there is no velocity estimate the loop steps itself down.** Whenever `velocity_valid` is false — on the ground, where the flow sees nothing at 5 cm; over a bad patch of floor; too high — the module drops the velocity (and position) loop and the sticks go back to commanding tilt, exactly as in the centre position. QGC shows `no flow estimate, sticks are tilt`, and `flow valid, velocity/position loops on` when it returns. Two consequences: you can **arm and take off with the switch up** — it is an attitude/altitude take-off until the flow comes good at about 0.5 m, then the velocity loop takes over — and a stick you are holding for *velocity* becomes, for the duration of a dropout, a *tilt* of the same fraction. Centre the sticks when you hear the message.

With the velocity loop running, the roll/pitch sticks command **velocity in the heading frame**, up to `UAS_MAX_VXY` (default 1 m/s) at full deflection; the module rotates that into NED for you. Sticks centred means zero velocity — the loop actively stops the vehicle. Throttle and yaw sticks are unchanged from Lab 3.

### 1.5 Bench Test

Props off, loop switch **up** (or `UAS_LOOP_EN = 15`), arm in Offboard, sticks centred. The velocity setpoint is zero, so any velocity you impose by hand should produce a tilt command *opposing* it.

1. `uas_control status` — confirm `velocity … valid: yes`. If it is `NO` on the bench that is normal (no flow on a static floor at 5 cm); lift the vehicle to ~0.5 m over a textured surface and it should come good within a second. Do the rest of the test at that height.
2. Carry the vehicle **forward** at walking pace, pointed north: `listener vehicle_attitude_setpoint` (or the log) should show a **nose-up** (positive pitch) command — the loop trying to slow you down.
3. Carry it **right**: a **roll-left** (negative roll) command.
4. Turn the vehicle to face **east** and repeat step 2. The command must still be nose-up. If it becomes a roll command, your Step 1 rotation is missing or reversed.

### 1.6 Flight Test

Take off in Stabilized, hand over at a hover as in Lab 3 with the loop switch at **centre** (altitude) — confirm it still behaves. Then, in the hover, move the switch **up**.

- **Sticks centred:** the Lab 3 drift should stop. Expect a gentle correction as the loop catches the initial velocity, then a hover that stays within a metre or so, wandering slowly as the flow estimate breathes.
- **A slow, growing sway** (period of a few seconds) is the classic velocity-loop failure: too much P for the lag in the estimate. Halve it. Flip to Stabilized if it grows past a couple of metres of travel.
- **Stick inputs** should feel like steering a velocity: push forward, it accelerates to a speed and holds it; release, it stops.
- **Unlearn the attitude-mode reflex.** In attitude mode you stop a drift by tilting against it — stick opposite the motion. In velocity mode that same stick means "go the other way at up to `UAS_MAX_VXY`", and the vehicle will. **Centring the stick is the brake.** The first velocity-loop flight on the instructor's vehicle "zoomed off" for exactly this reason: a backward drift, a stick pushed back, and a loop faithfully delivering −1 m/s.
- **Gains that are too soft feel like no loop at all.** A P gain of 0.12 rad per m/s answers a 0.25 m/s drift with 1.5° of tilt — you will not notice it working. PX4 flies this airframe at `MPC_XY_VEL_P_ACC` = 1.8 m/s² per m/s, which is 0.18 rad per m/s once you divide by *g*, with an integrator ten times larger than instinct suggests. Convert PX4's gains before deciding yours are wrong.
- **Over a bad patch of floor** `velocity_valid` will drop and the module hands you an attitude-hold vehicle (sticks are tilt) until it returns — see 1.4. Learn what that looks like.

### 1.7 Deliverable

From one flight: `vx`/`vy` setpoint against measured, the roll/pitch commands the loop produced, and `v_xy_valid`, over a window that includes sticks-centred hover and at least one commanded velocity step. Plus a comparison of horizontal wander with `UAS_LOOP_EN = 7` and `15` over the same duration.

---

## Part 2: Motion Capture

*TBD.*

---

## Part 3: Position Control

**File:** `PositionController.cpp`

*TBD.*

---

## Part 4: Trajectories

*TBD.*

---

## Lab Deliverables

*TBD.*

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
