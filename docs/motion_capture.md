---
layout: default
title: Using Motion Capture
nav_order: 5
last_modified_at: 2026-10-21 12:00:00 -0400
---

# Using Motion Capture
{: .no_toc }

How to get your vehicle tracked by the lab's Vicon system, how to send it a
flight plan, and how to check it is receiving both. This is the practical
procedure. The parameters your airframe needs are in
[Lab 4]({% link docs/labs/lab4.md %}#part-2-motion-capture).

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## What you get, and what you have to do

The room's cameras track your vehicle and a laptop broadcasts its position over
the XBee radio. Your vehicle fuses that position and can then hold a point in
the room to a centimetre, which optical flow alone cannot do because its
position is a drifting integral.

Three things have to be true before any of that works:

1. Your vehicle exists as an **object** in Vicon Tracker, named correctly.
2. The **broadcaster** is running on the lab laptop.
3. Your airframe's `UAS_MOCAP_ID` matches your vehicle number.

---

## Part 1: Create your object in Vicon Tracker

Do this once. The object persists between sessions, so check whether yours
already exists before making a new one.

**Set the vehicle down facing room +y before you create the object.** Tracker
aligns a new object's axes with the room's axes at the moment you create it, so
the rotation it reports afterwards is the rotation *since creation*, not an
absolute heading. The broadcaster assumes the nose was pointing along room +y,
which is north in the vehicle's frame. Create it facing any other way and your
vehicle flies at a fixed angle to every command it is given, which is the one
failure that reliably destroys an airframe.

The room's axes: **+y is north and +x is east**, with z up. They are marked on
the floor. The room is not aligned to the earth's frame, so magnetic headings
are not useful here; +y is roughly north and +x roughly east, and that is as
precise as it needs to be.

1. Put the vehicle in the middle of the capture volume, the right way up, nose
   along room +y.
2. In Tracker, switch to the **Objects** pane and confirm the markers on your
   vehicle appear as unlabelled markers. You need at least four.
3. Drag a selection box around your vehicle's markers only. Check the count
   matches the number of markers on the airframe. A stray marker from the floor
   or another vehicle will distort the object.
4. Type the name (see below) in the **Create Object** box and create it.
5. Confirm the object appears in the object list with a tick beside it, which
   means it is enabled and being tracked.
6. Lift the vehicle and move it around the volume. The object should stay
   tracked. If it drops out, you probably captured too few markers.

### What to name it

The name must be exactly one of these, matching your vehicle number. Nothing
else is broadcast.

| Vehicle | Object name | `UAS_MOCAP_ID` |
|---|---|---|
| 0 | `UAS_00` | 1 |
| 1 | `UAS_01` | 2 |
| 2 | `UAS_02` | 3 |
| 3 | `UAS_03` | 4 |

**Getting `UAS_MOCAP_ID` wrong means flying to another team's position, or
flying another team's plan.** Every vehicle in the room hears every pose and
every plan, because the messages carry no destination field. That parameter is
the only thing that makes your airframe ignore the rest.

### If your object already exists but the heading is wrong

Delete it and create it again facing room +y. There is no offset to apply after
the fact, and nothing to correct in software: it is a two-minute job in Tracker
and it is the right fix. Confirm with the direction checks in Part 3 before you
fly it.

---

## Part 2: Run the broadcaster

One laptop in the lab broadcasts to every vehicle. It is normally already
running, started once at the beginning of the session.

```
~/uas/tools/mocap_stream.sh
```

**Do not start a second one.** The radio is a byte pipe with room for exactly
one writer, so a second program sending on it corrupts both streams and every
vehicle in the room loses position. If you think it is not running, check with
whoever is at the laptop before starting anything.

It broadcasts whichever objects Tracker is currently tracking, up to four, and
shares the radio between them, so the rate falls as more vehicles appear:

| Vehicles tracked | Pose rate each |
|---|---|
| 1 | 40 Hz |
| 2 | 24 Hz |
| 3 | 15 Hz |
| 4 | 12 Hz |

You do not need to restart it when you load a plan, and you do not need to
restart it when your vehicle enters or leaves the volume.

If it exits within ten seconds saying it saw no packets, Tracker is not
streaming. Check that Tracker's **UDP Object Stream** output is enabled and that
at least one object is being tracked.

---

## Part 3: Check your vehicle is receiving

With the vehicle powered, inside the volume, and the broadcaster running:

```
python3 ~/uas/tools/nsh.py 'uas_control status'
```

Look for the `mocap` line:

```
mocap   OK    n/e/alt [m]:   0.91   0.61   0.07  rpy [deg]:   -0.8    0.8    2.2
```

| What you see | What it means |
|---|---|
| `mocap OK` | poses are arriving and fresh. This is what you want |
| `mocap STALE` | nothing arriving. Vehicle outside the volume, object not tracked, broadcaster not running, or `UAS_MOCAP_ID` wrong |
| `n`/`e` match where you are standing | the frame is right |
| `alt` about 0.07 on the floor | correct. The object origin sits a few centimetres up the airframe |
| `rpy` near zero sitting flat | attitude is right |

### The direction checks, before you fly

**Do these every session, with the propellers off.** A heading or sign error
puts every position correction in a frame rotated from reality, so the vehicle
accelerates away from its target instead of toward it. Two of these five were
wrong when the system was first set up, and each would have destroyed the
vehicle on its first hover.

Carry the vehicle by hand, disarmed, and confirm each one:

| Move it | `uas_control status` should show |
|---|---|
| toward room north (+y) | `n` increases |
| toward room east (+x) | `e` increases |
| upward | `alt` increases |
| rotate clockwise seen from above | `yaw` increases |
| tilt right wing down | `roll` increases |

If any of these is backwards or 90 degrees out, stop. Do not fly it. Ask.

---

## Part 4: Load a flight plan

A plan is a list of timed waypoints your vehicle flies when you select motion
capture on the mode switch. Run this **at the broadcaster laptop**, because that
is where the plan files live.

Generate a square:

```
~/uas/tools/plan_load.sh UAS_01 --size 1.0 --alt 1.0
```

Load a plan you wrote yourself:

```
~/uas/tools/plan_load.sh UAS_01 myplan.txt
```

See what your vehicle will fly, or clear it:

```
~/uas/tools/plan_load.sh UAS_01 --show
~/uas/tools/plan_load.sh UAS_01 --clear
```

The plan is checked before it is installed, so a file that will not fly is
refused at the laptop with the line number rather than silently doing nothing in
the air. The broadcaster picks up the new plan within a second and sends it under
your vehicle's id only.

### Where the files live

```
~/uas/plans/UAS_01.txt      your vehicle's plan
~/uas/plans/vehicles.txt    object name to vehicle id map, do not edit
```

### Writing your own plan

Four or five columns, whitespace separated, `#` starts a comment:

```
# t      x      y     alt   yaw
   0.0    0.0    0.0   1.00    0
   5.0    0.0    0.0   1.00    0
   8.0    1.0    0.0   1.00    0
  11.0    1.0    1.0   1.00   90
  17.0    1.0    1.0   1.00   90
```

| Column | Unit | Meaning |
|---|---|---|
| `t` | s | time from plan start. Must increase strictly down the file |
| `x` | m | north |
| `y` | m | east |
| `alt` | m | height above the floor |
| `yaw` | deg | optional. Write `-` or leave the column off to hold the previous heading |

The times set the speed, because the vehicle flies the straight line between
consecutive waypoints: 1 m over 4 s is 0.25 m/s. To pause at a point, repeat it
with a later time. Before the first waypoint and after the last, the vehicle
holds that end point. The limit is 32 waypoints.

With `UAS_PLAN_REL = 1` the plan is flown from wherever your vehicle is when you
flip the switch, so `x` and `y` are a shape rather than a place and the same plan
repeats from any starting point. **Height and heading are not translated**: `alt`
is above the floor, and `yaw` is a room heading, so a plan written with yaw
angles turns the vehicle to those room headings whichever way it was pointing
when it started. Write the first waypoint at `0 0` and keep the rest relative to
it.

### Running the plan

Take off and hover under your own control first, then select motion capture on
the mode switch. The plan starts there. Flip out of motion capture to abandon it,
and back in to run it again from the beginning.

`--clear` stops the broadcast but does not erase the plan your vehicle is already
holding. That is replaced when a different plan arrives, or dropped when the
vehicle is power cycled.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `mocap STALE` | object not tracked, or wrong `UAS_MOCAP_ID` | check the object has a tick in Tracker and that the vehicle is in the volume |
| `mocap STALE` for everyone | broadcaster not running, or two writers on the radio | one broadcaster only. Ask at the laptop |
| Pose arrives but `n`/`e` are nonsense | object created facing the wrong way | delete the object and create it again facing room +y (Part 1) |
| `yaw` is out by a constant, 90 or 180 degrees | object created facing +x, or facing backwards | same fix. Tracker takes the zero heading from creation, so it can only be corrected by recreating the object |
| Vehicle drifts away as soon as position control engages | heading error | run the five direction checks. Do not fly until they pass |
| Pose stream dies when QGroundControl is open | QGC grabbed the radio's serial adapter | turn off QGC's serial auto-connect |
| Plan does not start | no plan loaded, or the mode switch is not on motion capture | `plan_load.sh <OBJECT> --show` |
| Plan refused with a line number | the file cannot be flown as written | fix that line. Times must increase strictly |
| Pose rate lower than expected | more vehicles are being tracked | expected. The radio is shared, see the table in Part 2 |
