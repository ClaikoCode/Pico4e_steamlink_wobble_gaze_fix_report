# Changes

Both changes are in `lib/arm64-v8a/libvrlink_scene.so`. Each replaces one instruction with a call to new code. The new code is written over the start of an exported, unreferenced function from the bundled Jerasure erasure-coding library.

The tables below give the addresses in Steam Link 2.0.24, whose patched builds were used on the headset, and in the 2.0.25 beta, whose patched builds were emulator-checked and used in normal play (tester report), but not in a measurement run.

## 1. Head velocity: `QSVLDeviceHmd::GetPose`

### What changes

`GetPose` locates `VIEW` in `STAGE` at the current time with an `XrSpaceVelocity` chained, and forwards the reported linear velocity. The change keeps the official pose, angular velocity and flags, and replaces only the linear velocity:

```cpp
// After the existing locate at `now` succeeds.
XrSpaceLocation future{XR_TYPE_SPACE_LOCATION};
bool ok = XRQLocateReferenceSpace(ctx, XR_REFERENCE_SPACE_TYPE_VIEW, now + 35ms, future);

velocity.velocityFlags &= ~XR_SPACE_VELOCITY_LINEAR_VALID_BIT;
if (ok
    && (future.locationFlags & XR_SPACE_LOCATION_POSITION_VALID_BIT)
    && (location.locationFlags & XR_SPACE_LOCATION_POSITION_VALID_BIT)) {
    velocity.linearVelocity = (future.pose.position - location.pose.position) / 0.035f;
    velocity.velocityFlags |= XR_SPACE_VELOCITY_LINEAR_VALID_BIT;
}
// Steam Link then sends zero linear velocity if the bit is clear.
```

Both positions come from the same locate path, so they are in the same frame whatever the recenter state.

### Why 35 ms

- The runtime's future poses advance with the requested time only up to about 44 ms. Requests for 50 and 70 ms both return about 44 ms.
- Of the single-window sources, 40 ms gave the lowest error, both replayed against recorded motion and live on screen (5.6 mm mean, against 7.8 mm for 35 ms). The rejected curve fit reached 3.4 mm; see `APPENDIX.md`. 35 ms was chosen to stay clear of the 44 ms limit: near it, the returned horizon varies, and a 40 ms request came back as up to 46.6 ms ahead at the 95th percentile.
- The value is a time added to the pose time, not a frame count. The runtime's 44 ms limit was measured only at 90 Hz.

### Binary patch

| | 2.0.24 | 2.0.25 beta |
|---|---|---|
| `GetPose` | `0x10310C` | `0x102BC0` |
| Hook, replaced instruction | `0x10318C`, `ldur q0, [sp, #0x34]` (`0x3CC343E0`) | `0x102C40`, same |
| Host | `0x1AD1B4`, `jerasure_print_matrix` (352 bytes) | `0x1B4D14`, same |
| PLT entry of `XRQLocateReferenceSpace(ctx, type, time, location)` | `0x21C670` | `0x223ED0` |

The hook becomes `bl host`. The first 164 bytes of the host are replaced with:

```asm
    stp  x29, x30, [sp, #-0x50]!
    mov  w8, #42                    // XR_TYPE_SPACE_LOCATION
    str  w8, [sp, #0x10]
    str  xzr, [sp, #0x18]           // next = NULL
    str  xzr, [sp, #0x20]           // locationFlags = 0
    ldr  x2, [sp, #0x58]            // GetPose's `now`
    movz x9, #3776
    movk x9, #534, lsl #16          // 35,000,000 ns
    add  x2, x2, x9
    add  x3, sp, #0x10
    mov  x0, x21                    // ctx
    mov  w1, #1                     // XR_REFERENCE_SPACE_TYPE_VIEW
    bl   locate
    ldur x8, [x29, #-0x30]          // velocityFlags
    and  x8, x8, #~1                // clear LINEAR_VALID
    tbz  w0, #0, finish             // locate failed
    ldr  x9, [sp, #0x20]
    tbz  w9, #1, finish             // future position invalid
    ldr  x9, [sp, #0x70]
    tbz  w9, #1, finish             // current position invalid
    ldp  s0, s1, [sp, #0x38]        // future position
    ldr  s2, [sp, #0x40]
    ldp  s3, s4, [sp, #0x88]        // current position
    ldr  s5, [sp, #0x90]
    fsub s0, s0, s3
    fsub s1, s1, s4
    fsub s2, s2, s5
    movz w9, #37449
    movk w9, #16868, lsl #16        // 28.5714 = 1 / 0.035 s
    fmov s3, w9
    fmul s0, s0, s3
    fmul s1, s1, s3
    fmul s2, s2, s3
    stp  s0, s1, [x29, #-0x28]      // linearVelocity
    stur s2, [x29, #-0x20]
    orr  x8, x8, #1                 // set LINEAR_VALID
finish:
    stur x8, [x29, #-0x30]
    ldp  x29, x30, [sp], #0x50
    ldur q0, [sp, #0x34]            // replaced instruction
    ret
locate:
    b    XRQLocateReferenceSpace    // PLT entry
```

The 2.0.25 stub is identical except for the final branch target.

## 2. Gaze: `QSVLDeviceHmd::GetInteractionGaze`

### What changes

`GetInteractionGaze` locates the eye-gaze action space and `VIEW` in `LOCAL` at the current time. It sends a point 1 m along the gaze from the eye, in head space:

```cpp
gaze = inverse(head.orientation) * (eye.position + eye.orientation * {0, 0, -1} - head.position);
```

The change sets `eye.position = head.position` before that line, so the result is the gaze direction in head space:

```cpp
gaze = inverse(head.orientation) * (eye.orientation * {0, 0, -1});
```

Once the height error is removed, the eye position this runtime reports is 1 mm in front of the head origin, so ignoring it moves the gaze by under 0.1°.

Two related gaps exist with or without this change. The head locate's return value and flags are not checked, so a failed locate gives a zero gaze vector that is still reported as valid. The eye location's flags are not checked either, only the `XrResult`; the eye pose was seen frozen while Steam Link was not in focus.

### Binary patch

| | 2.0.24 | 2.0.25 beta |
|---|---|---|
| `GetInteractionGaze` | `0x102A74` | `0x102528` |
| Hook, replaced instruction | `0x102BBC`, `ldp s0, s1, [x29, #-0x70]` (`0x2D7207A0`) | `0x102670`, same |
| Host | `0x1B4CCC`, `blaum_roth_coding_bitmatrix` (768 bytes) | `0x1BC82C`, same |

The hook is the first instruction after the head locate. Valve's code keeps the eye position in `s9`, `s14` and `s11`, and the head position is at `[x29, #-0x60]`. The first 16 bytes of the host are replaced with:

```asm
    ldp  s9, s14, [x29, #-0x60]     // eye.position.xy = head.position.xy
    ldur s11, [x29, #-0x58]         // eye.position.z  = head.position.z
    ldp  s0, s1, [x29, #-0x70]      // replaced instruction
    ret
```

`GetInteractionGaze` in 2.0.25 uses the same stack layout and registers. The differences are branch targets, string addresses, two object field offsets and two logged source-line numbers, none of which the patch touches.

## Hashes

Official APKs:

| Version | APK SHA-256 |
|---|---|
| 2.0.24, shipped with SteamVR 2.18.4 | `7eb6e4b36dbdd420501be66e6577320b19e2576a36e872d3c31279860a4c1c47` |
| 2.0.25 beta | `94766d77ca9240dd8658ba4d28548ad55d45d4c7ed3a8e24c131e5a9cc951af9` |

Patched `libvrlink_scene.so`. The APK hash depends on the signing key, so the library hash is the one to reproduce:

| Build | Library SHA-256 |
|---|---|
| 2.0.24 official | `62529b46042c3616a9206188fb8e04c47a5374fdc50b914ba0c1a8fa101a1884` |
| 2.0.24, velocity | `58f492ffd9b097df86cc070d4bd91ce05d6e4df055b5a75b456a9dce16b0ebbb` |
| 2.0.24, velocity and gaze | `f64b0540938eb9d4b951c80767fca7c6fe1ac68b6e7ab0d58258e158f1cb422d` |
| 2.0.25 beta official | `5c0d0ff95b20c017f90dee6ae1bab1cfda80298580db05163e3ff049c9671f1b` |
| 2.0.25 beta, velocity | `eb1b15ab625d93043b2ed23d7f3268e6e99c3e85efd67a41498806051cc1836e` |
| 2.0.25 beta, velocity and gaze | `f2806371e14dc2bd5acba2dbf467e68e3049c83bcca3288a941d209a5fe5ebf9` |
