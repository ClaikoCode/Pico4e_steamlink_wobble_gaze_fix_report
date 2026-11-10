# Steam Link on Pico 4 Enterprise: head wobble and misplaced foveation

This repo contains the condensed information on the identification, testing, and fixing of two bugs that Pico 4 headset users have experienced on Steam Link. Only a Pico 4 Enterprise was tested; the specific versions and setup are listed in the summary below. The head wobble has also been reported on the regular Pico 4 (not verified here), which has no eye tracking and so cannot show the gaze bug. The velocity workaround does not depend on eye tracking, so it should apply there too, but that is untested.

I have an M.Sc. in Game and Software Engineering, but Claude Opus 5.5 with thinking level High was used extensively throughout this whole process to identify the problem, attempt a fix, and then test the fix. This was an iterative process which I orchestrated throughout. 

As it is an LLM, it can make mistakes and confidently make wrong choices or draw wrong conclusions. I therefore asked it several times throughout the process to scrutinize gaps in each test, avoid overgeneralizations, retrace its steps regularly to check for consistency, document the process, and also to use agents with separate context to independently check factual claims. These "internal" tests identified several problems during every iteration, so I strongly believe it was very valuable to explicitly ask for.

These measures are the best I could do to better adhere to scientific standards, which I value, and avoid too much bloat or loss of scope. However, this repo contains the final boiled-down version of the findings of all the experiments that were done. It is still quite verbose, but I also did not want to attempt to write it myself, as Claude had a much better grasp of details that I might have missed.

I've tried to get Claude to condense this down to the essentials. Each point should be something you can either act on or quickly check and dismiss. I've also attached a license that gives anyone the freedom to look through this and use it without attribution or restrictions, which I'm alright with.

I hope that these fixes can be implemented in some way, either by Pico or by Valve, to make wireless streaming on Steam Link a valid option for users with older Pico 4 headsets.

## Summary

On a Pico 4 Enterprise, Pico's OpenXR runtime returns two values that are wrong, and Steam Link forwards both unchecked.

| | Wrong value from the runtime | Reported symptom | Workaround in Steam Link | Measured result |
|---|---|---|---|---|
| 1 | Head linear velocity is rotated relative to the head pose | World drifts in the wrong direction during head movement | Derive velocity from the runtime's pose 35 ms ahead | Pose error at display time: 24.0 → 7.8 mm |
| 2 | Eye-gaze position sits up to 1.1 m from the head when the head tilts | Image turns low quality when looking down | Aim foveation from gaze direction only | Foveation error, head down: 42.5° → 10.9° |

`CHANGES.md` has the exact patches. `APPENDIX.md` has limits and open questions.

**Setup.** One Pico 4 Enterprise, firmware 5.13.8, 90 Hz. Pico's OpenXR runtime is package version 2.1.12 and reports `runtimeVersion` 3.0.1 through `xrGetInstanceProperties`. SteamVR 2.18.4, Steam Link 2.0.24. Both patched functions have the same layout in the 2.0.25 beta.

**Method.** Logging builds of Steam Link recorded what `xrLocateSpace` returned, before any Steam Link code used it. Before use, each patch was checked by running the patched `GetPose` and `GetInteractionGaze` from Steam Link in an ARM64 emulator.

**Terms.**
- **Pipeline:** the headset sends head pose and velocity to the PC. The PC extrapolates the pose, renders, and streams video back.
- **`baseSpace`:** the space whose frame `xrLocateSpace` expresses its results in.
- **`VIEW`, `STAGE`, `LOCAL`:** OpenXR spaces. `VIEW` is the head. `STAGE` is fixed to the room. `LOCAL` is re-anchored at each recenter.
- **Recenter:** Pico's reset of the forward direction and origin, also called a tracking reset here. Pico performs one on the user's request, and also on its own (section 1.2).
- **Official:** the unpatched Steam Link build, which sends the velocity as the runtime reports it.
- **Foveation:** Steam Link encodes at full quality only around a center point, which follows the gaze.

## 1. Head linear velocity

### 1.1 What Steam Link asks for

`QSVLDeviceHmd::GetPose` locates `VIEW` in `STAGE`, with an `XrSpaceVelocity` chained. It gets a pose and a velocity, and it needs them in the same frame.

### 1.2 What the runtime returns

The linear velocity has the right speed and the wrong direction.

- **Speed.** Reported velocity matches the time derivative of the reported position, with a gain of 1.01 and about 5 ms of delay.
- **Direction.** Reported velocity is yawed away from that derivative. The angle holds steady until Pico resets tracking, and then changes.

The pose itself behaves correctly: the `STAGE` position never jumped at a reset. Only the velocity rotated.

**Pico resets tracking on its own.** In sessions A to C, `PvrTrackingService: resetTracking` lines appear after the headset is put on and before Steam Link starts streaming, around start-up and Pico's boundary confirmation. None of them is a recenter by the user. What exactly triggers them was not established. Each line records a yaw offset.

The table compares the measured error with the yaw in Pico's most recent reset line.

| Session | Resets so far | Measured velocity direction error | Yaw in Pico's last reset line | Error minus logged yaw |
|---|---|---:|---:|---:|
| A | automatic only (first of five reset states) | 9.7° | −7.5° | 17° |
| B | automatic only | 41.8° | 39.0° | 3° |
| C | automatic only | 57° | 85.4° | about −29° |
| C | plus one user recenter | −26° to −32° | 1.5° | about −29° |

Two things follow.

- **The logged yaw explains how the error changes.** When a reset changes the logged yaw, the error changes by the same amount. In session C the user recenter moved both by about 85°. Session A also had later recenters, and across its five reset states the last column stayed at 17.3° ± 0.4°.
- **The logged yaw does not explain the whole error.** The last column is the unexplained part. It is fixed within a session and differs between sessions. Its source is unknown.

In all three sessions, the error was present from the start, before any recenter by the user.

### 1.3 Why that is wrong

The [OpenXR specification](https://registry.khronos.org/OpenXR/specs/1.1/html/xrspec.html) defines `XrSpaceVelocity::linearVelocity` as "expressed in the reference frame of `baseSpace`". Here `baseSpace` is `STAGE`. Whatever causes the rotation, a velocity whose direction disagrees with the derivative of the `STAGE` position is not in that frame.

### 1.4 What Steam Link does with it

On this headset, `GetPose` forwards the velocity unchanged. It adds a head-offset term only on Pico models with a configured offset. Every frame returned by the PC carries the head pose it was rendered for. That pose fits `position + velocity × H`, with H = 78 ms, a velocity gain of 0.999 and a residual of 0.79 mm. The rotated velocity is therefore applied in full, 78 ms ahead.

### 1.5 What reaches the screen

A logging build compared the pose in each returned frame with the runtime's own head pose at that frame's display time. In session C, it cycled through five velocity sources, 6 s each, over about 160 s of head motion. The order was fixed and the run was not blind.

| Linear velocity sent | Mean error | 95th percentile |
|---|---:|---:|
| Reported by the runtime (official) | 24.0 mm | 44.6 mm |
| Position change over the next 20 ms | 9.0 mm | 16.9 mm |
| Position change over the next 35 ms | 7.8 mm | 15.8 mm |
| Position change over the next 40 ms | 5.6 mm | 11.8 mm |

Official velocity had the largest error in each of the five passes through the sources. The fifth source, a curve fit, is in the appendix.

### 1.6 Workaround

Locate `VIEW` in `STAGE` a second time at `now + 35 ms`, and send the difference:

```cpp
velocity.linearVelocity = (future.pose.position - location.pose.position) / 0.035f;
```

Both positions come from the same function and the same base space, so they share a frame in any recenter state.

The window is 35 ms and not 40 ms because the runtime's prediction stops advancing at about 44 ms ahead. Near that limit, the horizon it actually returns varies: at the 95th percentile, a 40 ms request returned a pose 46.6 ms ahead. 35 ms keeps clear of that region.

**Possible side effect, not measured.** The workaround may add a slight stutter during fast head movement. An earlier version (pose requested 70 ms ahead, zero velocity) was rated worse for stutter than official in a nominally blind test: 1.75 against 0.50 on a 0-to-4 scale, with p = 0.086. In normal play with the current 35 ms version, the tester noticed what may be a slight stutter, but could not confirm it.

### 1.7 Angular velocity has the same class of error, and Steam Link already handles it

Pico reports head angular velocity in the head's own frame, where the OpenXR specification requires `baseSpace`. The reported vector matches the head-frame derivative of orientation with a cosine of 1.000. Against the `STAGE`-frame derivative, the cosine was 0.71 in one session and 0.59 in another.

Steam Link has a rule for this. `XRQCreateXRInstance` marks the runtime when `runtimeName` contains "Pico". For a Pico runtime with `runtimeVersion` major below 4, `GetPose` sends angular velocity as reported. For every other runtime, it rotates angular velocity from the base frame into the head frame. This runtime reports 3.0.1, so its head-frame angular velocity passes through unchanged, which is what Steam Link needs.

A rule of that kind cannot fix linear velocity, because the linear error changes at every tracking reset.

## 2. Eye-gaze position

### 2.1 What Steam Link asks for

`QSVLDeviceHmd::GetInteractionGaze` locates two spaces in `LOCAL`: the `XR_EXT_eye_gaze_interaction` pose and `VIEW`. It sends the point 1 m along the gaze from the eye, expressed in head space:

```cpp
gaze = inverse(head.orientation) * (eye.position + eye.orientation * {0, 0, -1} - head.position);
```

### 2.2 What the runtime returns

The eye position is displaced from the head position. The displacement depends only on head orientation and one signed length `h`:

```text
(eye.position - head.position), in head space = h * (inverse(head.orientation) * up - up) + (0, 0, -0.001)
```

The formula was fitted to four sessions, with `h` from −0.011 to 0.70 m. It gave R² of 0.9987 to 1.0000 and a residual under 0.3 mm. `h` is constant between Pico recenters and takes a new value at each one. Within one session it went from 0.026 m to 0.48 to 0.51 m over 11 recenters, and to 0.31 m after one recenter made with the head tilted.

One logged sample, from the session with `h` = 0.70 m, with the head pitched 69.7° down:

| | Value |
|---|---|
| `eye.position − head.position`, world axes | 0.46 m up, 0.66 m back (0.80 m in total) |
| Gaze direction, relative to head forward | 6.5° down |
| Steam Link's gaze point, relative to head forward | 59.4° down |

The 1 m point lands 0.34 m in front of the head and 0.57 m below its forward axis. Steam Link's aim is therefore 59.4° − 6.5° = 53° below the gaze.

### 2.3 Why that is wrong

An eye stays within centimeters of the head. The reported eye position was up to 1.1 m from the head.

The formula matches a transform-order mistake: `h` is subtracted along the head's up axis for one pose and along the world's up axis for the other. That mechanism was read once in the runtime's disassembly but could not be found again, so it is unconfirmed. The data also do not show which of the two poses carries the error. Only their difference matters here, because the difference is what Steam Link uses.

### 2.4 What Steam Link does with it

Steam Link uses both positions as reported. From reading `driver_vrlink.dll`, the driver passes the resulting point to SteamVR as the gaze target, and the foveation center follows it.

### 2.5 What reaches the screen

A logging build alternated every 10 s between Steam Link's calculation and gaze direction only. It logged the foveation center sent with each frame. The table gives the mean angle between that center and the gaze direction from the same eye tracker. The run was not blind.

| Session `h` | Head pitch | Steam Link's calculation | Direction only |
|---|---|---:|---:|
| mostly 0.48 to 0.51 m | 55° or more down | 42.5° | 10.9° |
| mostly 0.48 to 0.51 m | 40° to 70° up | 15.0° | 9.7° |
| 0.09 m | 55° or more down | 11.1° | 7.5° |

About 10° remains in the direction-only column in every row, even with head and eyes still (9.8° and 19.6° in the two sessions). Its cause is unknown. Because this baseline appears in both columns, the bug's contribution is the difference between them. That difference is larger in the session with the larger `h`.

In the first row, the median center sat 49° below the middle of the view while the gaze was 8° below it. A center that low is at the edge of the frame, which matches the reported symptom.

### 2.6 Workaround

Replace the eye position with the head position, so only the direction is used:

```cpp
gaze = inverse(head.orientation) * (eye.orientation * {0, 0, -1});
```

The direction does not depend on `h`. Dropping the remaining 1 mm offset changes the aim by less than 0.1°.

## 3. Suggested fixes

Neither workaround is specific to Pico in principle. Neither has been tested on another runtime.

| | In Steam Link | In Pico's runtime |
|---|---|---|
| Velocity | Derive linear velocity from poses, or compare the reported velocity with the position change and fall back when they disagree | Report linear and angular velocity in the frame of `baseSpace` |
| Gaze | Aim foveation from gaze direction only | Apply `h` in the same frame for the eye-gaze pose and the head pose |

Separately, `GetInteractionGaze` ignores the result of the head locate. A failed head locate yields a zero gaze vector that is sent as valid.

## License

CC0 1.0. Everything here can be used, changed or shipped in any product without restriction or attribution. No Valve or Pico binaries are included.
