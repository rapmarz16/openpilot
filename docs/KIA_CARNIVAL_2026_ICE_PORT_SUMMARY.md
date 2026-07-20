# 2026 Kia Carnival SX ICE port summary

## Purpose

This fork adds support for the Canadian-market 2026 Kia Carnival SX with the gasoline engine, CAN-FD, and Highway Driving Assist II (HDA II). It is intentionally based on Sunnypilot's comma four release branch and keeps the vehicle's factory Smart Cruise Control and Forward Collision-Avoidance Assist stack in charge of longitudinal control.

The supported installer branch is `kia-carnival-2026-ice-stock` in [`rapmarz16/openpilot`](https://github.com/rapmarz16/openpilot/tree/kia-carnival-2026-ice-stock).

## Baseline

- Stock baseline: Sunnypilot `release-mici` commit `af744c85e7c971e7bfbc8e6ee9e2bd75452a6f00`
- Vehicle support commit: `ba78094ed6bc11c7dc70668794dfe18e119bbb9e`
- Factory-longitudinal safeguard commit: `6ccde157c11e70505c79b32d54931556479dc3d8`
- Vehicle-code delta through the safeguard: eight embedded-opendbc files, 150 insertions, and 9 deletions

Documentation and update-automation commits added after those two commits do not change on-road vehicle behavior.

## High-level changes

### Vehicle recognition

- Adds a separate `KIA_CARNIVAL_2025` platform covering the 2025-26 HDA II Carnival.
- Adds the exact forward-camera and forward-radar firmware strings recorded from this Canadian 2026 SX.
- Keeps it separate from the existing 2022-24 Carnival platform so the newer CAN layout is not incorrectly applied to older vehicles.

### Physical and steering configuration

- Reuses the proven fourth-generation Carnival mass, wheelbase, and steering-ratio values.
- Reuses the existing fourth-generation Carnival torque-controller tuning rather than introducing unvalidated steering constants.
- Declares the HDA II vehicle as using harness Q.

### CAN-FD message layout

- Detects alternate HDA II steering messages, including `LKAS_ALT` at address `0x110`.
- Detects the Carnival's alternate cruise-button receive message at address `0x1AA` instead of the usual `0x1CF`.
- Uses the ICE gear message detected at address `0x40`.
- Reads the newer alternate left/right turn-signal lamp fields.
- Retains blind-spot monitoring detection through address `0x1BA`.

### Cruise cancellation and Panda safety

- Sunnypilot does not synthesize `0x1AA` cruise-button presses on this vehicle.
- When cancellation is requested while factory SCC is active, it may send `SCC_CONTROL` at address `0x1A0`.
- Panda permits that message only when it is a cancellation frame with `ACCMode = 4` and both acceleration requests equal to zero.
- Acceleration, resume, set, and arbitrary alternate-button transmissions remain rejected by Panda safety.

### Factory FCA/AEB preservation

- `alphaLongitudinalAvailable` is forced to false for `KIA_CARNIVAL_2025`.
- The vehicle therefore remains in stock longitudinal mode with `pcmCruise = true` and `openpilotLongitudinalControl = false`.
- Sunnypilot cannot enter the code path that disables the HDA II ADAS ECU at diagnostic address `0x730` for this platform.
- The Alpha Longitudinal setting is hidden/cleared for this vehicle.
- This safeguard remains in place until a validated CAN-FD longitudinal solution, such as a suitable eSCC architecture, is available and separately reviewed.

## What remains stock Sunnypilot

- Driving model, planning, UI, driver monitoring, logging, updater, and all non-Hyundai vehicle code are unchanged by the vehicle port.
- Sunnypilot's standard Hyundai CAN-FD steering limits, driver-torque override rules, brake/gas disengagement logic, and relay-malfunction protection remain in force.
- ABS, stability control, airbags, parking ADAS, and blind-spot ECUs are not disabled by the Carnival-specific changes.

## Important behavior and limitations

- Factory SCC/FCA/AEB remains responsible for acceleration and braking, but a software review cannot prove a real emergency-braking event. Dashboard warnings, diagnostic trouble codes, and initial post-install routes should still be checked.
- As with the stock Sunnypilot HDA II integration, factory LFA/LKAS steering commands are suppressed and replaced by Sunnypilot steering commands to prevent two steering controllers from fighting. Factory lane-centering behavior is therefore not concurrently available while Sunnypilot is controlling the steering path.
- Firmware matching is exact. A Carnival with different camera or radar firmware may not identify as this platform until its logs are reviewed and a new fingerprint is deliberately added.
- Steering tuning is inherited from the 2022-24 fourth-generation Carnival. It is not a new tune derived from destructive or limit-seeking vehicle tests.

## Validation completed

- Exact camera/radar firmware pair uniquely identifies `KIA_CARNIVAL_2025`.
- Synthetic route parameters identify the vehicle as ICE, HDA II, alternate steering, alternate buttons, alternate gears, and BSM-equipped.
- Hyundai parameter tests pass, including a regression test that requests Alpha Longitudinal with an ADAS ECU present and confirms it remains disabled.
- The complete compiled Hyundai CAN-FD Panda safety suite passed during the original port review.
- The comma four installer endpoint returned HTTP 200 and a valid ELF executable after the safeguard was published.

## Updating from Sunnypilot

Sunnypilot's `release-mici` branch is published as a snapshot without the normal `master` ancestry. GitHub's large "behind master" count is therefore not a useful measure of this port's freshness.

The updater in `.github/workflows/sync-sunnypilot-release-mici.yml` watches `release-mici`. When a new snapshot appears, it replays the reviewed fork commits onto that snapshot, runs the Hyundai parameter and compiled CAN-FD safety tests, and updates the installer branch only when every step succeeds. Failures leave the installer branch unchanged and create a GitHub notification.

For the full code-level explanation, see [KIA_CARNIVAL_2026_ICE_PORT_TECHNICAL.md](KIA_CARNIVAL_2026_ICE_PORT_TECHNICAL.md).
