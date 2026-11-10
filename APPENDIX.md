# Appendix

## Limits

- **One device.** All measurements come from one Pico 4 Enterprise on one firmware version, at 90 Hz.
- **Runtime code only partly traced.** For velocity, `getResetAlgoResult` in `libSlamTools.pxr.so` rotates a field by the recenter offset, as confirmed in an emulator. That the field is linear velocity was inferred. The path from that field to `XrSpaceVelocity` was read once but not re-traced. For gaze, the transform-order mechanism was read once in the disassembly but could not be found again.
- **Unexplained part of the velocity error.** The part not explained by Pico's logged reset yaw (17°, 3°, −29° in the three sessions) has no known cause.
- **Ground truth.** The on-screen error in section 1.5 is measured against the runtime's own head pose at display time, not an external tracker. Both on-screen measurements ran in a fixed order and were not blind.
- **Approximate angles.** Foveation angles use an approximate pixel-to-angle mapping. Differences between columns are more reliable than absolute values.
- **Derived velocity on other runtimes.** It assumes the runtime predicts at least 35 ms ahead. This was not tested on other runtimes.
- **Other Pico models.** On PICO 4 Pro and Ultra, `GetPose` adds a head-offset term to linear velocity after the patch point. Derived velocity may count that offset twice. Not tested.
- **2.0.25 beta.** The patched 2.0.25 builds were emulator-checked and used in normal play (tester report), but not in a measurement run.

## Secondary findings

- **What sets `h`.** `h` matches the height logged at each Pico recenter (`PvrTrackingService: resetTracking input`) plus a fixed head-frame offset of about 7 cm, rotated by the logged orientation. The fit covers 15 recenters with a 0.8 mm residual and a 2.2 mm leave-one-out error. It is a fitted model.
- **Eye calibration and lens inserts.** Neither affects the gaze error. The formula has no eye term. The eye point left after removing the error was identical across two calibrations, and `h` did not change when prescription inserts were removed (calibration and insert timing per the tester).
- **Foveation center motion.** The center is unchanged in about 60% of frames, moves 16 px or less in 38%, and jumps a median of 276 px in 1%.
- **Frozen eye pose.** The runtime returned a frozen eye pose while Steam Link was not in focus. Steam Link does not check the location flags.
- **Controllers.** Controller velocity direction stayed within 2° across a recenter. Controller angular velocity is in the base frame.
- **Curve-fit velocity.** A fit through poses 20 and 40 ms ahead reached 3.4 mm mean error but added 11% to 21% frame-to-frame jitter. It was not adopted.
- **Blind A/B test.** An earlier workaround (pose requested 70 ms ahead, zero velocity) was tested against official in 8 blind trials, rated from 0 (none) to 4 (unplayable). Wobble after moving and then recentering was rated 4, 4, 4, 4 for official and 1, 2, 2, 1 for the patch (p = 0.029). Wobble on a fresh start (2.00 against 1.75) and after recentering in place (2.25 against 1.75) did not differ. Stutter was rated worse with the patch (1.75 against 0.50, p = 0.086). With four measures and 4 trials per build, p = 0.029 is the smallest value possible and would not survive correction. The rater correctly guessed the build in all 8 trials. The current 35 ms build has not been blind-tested.

## Open tests

| Test | What it would settle |
|---|---|
| Locate the eye-gaze space in `VIEW` and `STAGE` as well as `LOCAL` | Whether the eye pose is inconsistent between base spaces, and which conversion is wrong |
| Six recenters at varied head heights and tilts | The fitted model for `h` |
| Log the runtime's recenter state against the velocity direction over a cold start | The source of the unexplained part of the velocity error |
| Measurement run on the patched 2.0.25 beta | Parity with 2.0.24 |
| Blind A/B of the 35 ms build against official, including stutter | Perceived effect of the current workaround, and whether it adds stutter |
| Future-pose cap at 72 Hz | Whether the 44 ms prediction limit is time-based or frame-based |
