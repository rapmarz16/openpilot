# 2026 Kia Carnival SX ICE technical diff walkthrough

## Scope of this walkthrough

This document explains every runtime and test block changed by the two vehicle commits relative to stock Sunnypilot `release-mici` at `af744c85e7c971e7bfbc8e6ee9e2bd75452a6f00`:

1. [`ba78094ed`](https://github.com/rapmarz16/openpilot/commit/ba78094ed6bc11c7dc70668794dfe18e119bbb9e) — adds 2026 Kia Carnival ICE support.
2. [`6ccde157c`](https://github.com/rapmarz16/openpilot/commit/6ccde157c11e70505c79b32d54931556479dc3d8) — disables Alpha Longitudinal for the Carnival to preserve factory FCA/AEB.

The combined vehicle diff changes eight files under `opendbc_repo`; it does not change Sunnypilot's model, planner, UI, driver monitoring, or core controls code.

## 1. Platform declaration: `values.py`

```python
KIA_CARNIVAL_2025 = HyundaiCanFDPlatformConfig(
  [
    HyundaiCarDocs("Kia Carnival (with HDA II) 2025-26", "Highway Driving Assist II",
                   car_parts=CarParts.common([CarHarness.hyundai_q])),
  ],
  KIA_CARNIVAL_4TH_GEN.specs,
)
```

- `KIA_CARNIVAL_2025` creates a distinct platform identity. The internal name says 2025 because that is the start of this vehicle generation; the documentation range includes the 2026 model.
- `HyundaiCanFDPlatformConfig` places the vehicle in the CAN-FD Hyundai architecture. The base class automatically supplies the CAN-FD platform flag and generated CAN-FD DBC mapping.
- `HyundaiCarDocs(...)` supplies the human-readable vehicle/model-year and HDA II package shown in generated vehicle documentation.
- `CarHarness.hyundai_q` identifies the required physical harness routing.
- `KIA_CARNIVAL_4TH_GEN.specs` reuses mass, wheelbase, steering ratio, and related physical parameters from the existing fourth-generation Carnival instead of inventing new values.
- No EV, hybrid, radar-SCC, high-torque, or experimental-longitudinal flag is statically added.
- One blank line was removed near the platform dataclasses. That is formatting only and has no runtime effect.

## 2. Firmware identification: `fingerprints.py`

```python
CAR.KIA_CARNIVAL_2025: {
  (Ecu.fwdCamera, 0x7c4, None): [
    b'\xf1\x00KA4 MFC  AT CAN LHD 1.00 1.00 99210-R0700 250324',
  ],
  (Ecu.fwdRadar, 0x7d0, None): [
    b'\xf1\x00KA4_ RDR -----      1.00 1.01 99110-R0510         ',
  ],
},
```

- The dictionary key binds the firmware pair to the new platform.
- `Ecu.fwdCamera` at diagnostic address `0x7C4` records the exact Canadian left-hand-drive camera firmware from the route.
- `Ecu.fwdRadar` at diagnostic address `0x7D0` records the exact radar firmware from the same vehicle.
- Requiring the pair prevents these strings from being silently folded into the older 2022-24 Carnival platform.
- The bytes, spaces, part numbers, versions, and build date are deliberately exact because firmware fingerprinting compares byte strings.

## 3. Runtime feature detection and longitudinal guard: `interface.py`

### Alternate cruise-button detection

```python
# This HDA II Carnival receives cruise buttons on 0x1AA instead of 0x1CF.
if candidate == CAR.KIA_CARNIVAL_2025 and 0x1aa in fingerprint[CAN.ECAN] and 0x1cf not in fingerprint[CAN.ECAN]:
  ret.flags |= HyundaiFlags.CANFD_ALT_BUTTONS.value
```

- The check runs only in the LKA-steering/HDA II path.
- `candidate == CAR.KIA_CARNIVAL_2025` prevents the route-specific rule from changing other Hyundai vehicles.
- `0x1AA in fingerprint[CAN.ECAN]` verifies that the observed electronic CAN bus actually contains the alternate button frame.
- `0x1CF not in fingerprint[CAN.ECAN]` prevents selecting the alternate parser when the standard button frame exists.
- Setting `CANFD_ALT_BUTTONS` changes receive parsing and Panda safety configuration to the `CRUISE_BUTTONS_ALT` layout.
- This does not grant permission to transmit synthetic `0x1AA` button messages.

### Alpha Longitudinal block

```python
if candidate == CAR.KIA_CARNIVAL_2025:
  # Keep the factory FCA/AEB stack active until longitudinal control is validated on this platform.
  ret.alphaLongitudinalAvailable = False
```

- The candidate check limits the safeguard to the new Carnival platform.
- `alphaLongitudinalAvailable = False` overrides the generic HDA II availability result even when firmware discovery finds the ADAS ECU.
- Later stock interface code computes `openpilotLongitudinalControl = alpha_long and alphaLongitudinalAvailable`; therefore the result is always false for this vehicle.
- Stock interface code then sets `pcmCruise = True`, leaving acceleration and braking with factory SCC.
- The Panda `LONG` safety bit is not set.
- The interface initialization condition that sends diagnostic communication-control to ADAS ECU `0x730` cannot be reached for this platform.
- UI/state cleanup removes the persistent Alpha Longitudinal parameter when the reported capability is unavailable.

## 4. Turn-signal parsing: `carstate.py`

```python
# ccNC cars use the alternate lamp signals.
left_blinker_sig, right_blinker_sig = "LEFT_LAMP", "RIGHT_LAMP"
if self.CP.carFingerprint in (CAR.HYUNDAI_KONA_EV_2ND_GEN, CAR.KIA_CARNIVAL_2025):
  left_blinker_sig, right_blinker_sig = "LEFT_LAMP_ALT", "RIGHT_LAMP_ALT"
```

- The comment now describes why the alternate signals exist: newer connected-car Navigation Cockpit (ccNC) vehicles use the alternate lamp fields.
- The default signal names remain unchanged for every other Hyundai platform.
- Adding `CAR.KIA_CARNIVAL_2025` to the tuple makes this vehicle read `LEFT_LAMP_ALT` and `RIGHT_LAMP_ALT`.
- `update_blinker_from_lamp(...)` remains stock and still performs the normal debounce/hold behavior.

## 5. Steering-torque substitution: `substitute.toml`

```toml
"KIA_CARNIVAL_2025" = "KIA_CARNIVAL_4TH_GEN"
```

- The left side is the new platform that needs lateral torque parameters.
- The right side selects the existing fourth-generation Carnival values: lateral acceleration factor `1.75` and friction `0.15` in this baseline.
- This avoids failing torque-parameter lookup and avoids adding an unvalidated tune.
- It does not alter Panda's hard steering-torque/rate limits; those remain enforced separately in safety code.

## 6. Panda CAN-FD safety configuration: `hyundai_canfd.h`

The safety changes are conditional on the existing `CANFD_LKA_STEER_MSG`, `CANFD_LKA_STEER_MSG_ALT`, and `CANFD_ALT_BUTTONS` safety bits. The Carnival route produces all three bits.

### Stock-longitudinal transmit lists for alternate buttons

```c
static const CanMsg HYUNDAI_CANFD_LKA_STEER_MSG_ALT_BUTTONS_TX_MSGS[] = {
  HYUNDAI_CANFD_LKA_STEER_MSG_COMMON_TX_MSGS(0, 1)
  HYUNDAI_CANFD_SCC_CONTROL_COMMON_TX_MSGS(1, false)
};

static const CanMsg HYUNDAI_CANFD_LKA_STEER_MSG_ALT_ALT_BUTTONS_TX_MSGS[] = {
  HYUNDAI_CANFD_LKA_STEER_MSG_ALT_COMMON_TX_MSGS(0, 1)
  HYUNDAI_CANFD_SCC_CONTROL_COMMON_TX_MSGS(1, false)
};
```

- The first array covers normal LKA steering plus alternate buttons.
- The second covers the Carnival's `LKAS_ALT` steering plus alternate buttons.
- The common steering macros retain stock steering and camera-suppression transmit permissions.
- `SCC_CONTROL` address `0x1A0` is added on E-CAN bus 1 because the controller cancels factory SCC using the SCC frame when alternate buttons are present.
- The `false` argument means this is not openpilot longitudinal mode and disables relay checking for that cancellation-only entry.
- This whitelist entry is not sufficient by itself to send arbitrary SCC frames; the transmit hook below applies content checks.

### Existing transmit hook relied upon by the new list

```c
if (msg->addr == 0x1a0U) {
  ...
  if (hyundai_longitudinal) {
    // Normal openpilot longitudinal acceleration limits.
  } else {
    const int acc_mode = (msg->data[8] >> 4) & 0x7U;
    if (acc_mode != 4) {
      violation = true;
    }
    if ((desired_accel_raw != 0) || (desired_accel_val != 0)) {
      violation = true;
    }
  }
  ...
}
```

- This block existed in stock Sunnypilot; the Carnival port makes use of it by adding `0x1A0` to the applicable transmit list.
- In stock-longitudinal mode, `ACCMode` must equal 4, the Hyundai cancellation state.
- Both encoded acceleration requests must decode to zero.
- Any non-cancel mode or nonzero acceleration marks the frame as a violation and Panda rejects it.

### Alternate receive checks

```c
static RxCheck hyundai_canfd_lka_steer_msg_alt_buttons_rx_checks[] = {
  HYUNDAI_CANFD_ALT_BUTTONS_RX_CHECKS(1)
  HYUNDAI_CANFD_SCC_ADDR_CHECK(1)
};

if (hyundai_canfd_alt_buttons) {
  SET_RX_CHECKS(hyundai_canfd_lka_steer_msg_alt_buttons_rx_checks, ret);
} else {
  SET_RX_CHECKS(hyundai_canfd_lka_steer_msg_rx_checks, ret);
}
```

- The new receive table watches alternate button frame `0x1AA` on bus 1 and the factory `SCC_CONTROL` status frame on bus 1.
- `hyundai_canfd_alt_buttons` selects the new table only when the interface supplied the matching safety flag.
- Standard-button HDA II cars continue using the original table and `0x1CF` parser.

### Conditional transmit-list selection

```c
if (hyundai_canfd_lka_steer_msg_alt) {
  if (hyundai_canfd_alt_buttons) {
    SET_TX_MSGS(HYUNDAI_CANFD_LKA_STEER_MSG_ALT_ALT_BUTTONS_TX_MSGS, ret);
  } else {
    SET_TX_MSGS(HYUNDAI_CANFD_LKA_STEER_MSG_ALT_TX_MSGS, ret);
  }
} else {
  if (hyundai_canfd_alt_buttons) {
    SET_TX_MSGS(HYUNDAI_CANFD_LKA_STEER_MSG_ALT_BUTTONS_TX_MSGS, ret);
  } else {
    SET_TX_MSGS(HYUNDAI_CANFD_LKA_STEER_MSG_TX_MSGS, ret);
  }
}
```

- This replaces two unconditional stock assignments with a two-flag matrix.
- The outer condition selects normal `LKAS` versus `LKAS_ALT` steering layout.
- The inner condition selects standard versus alternate button receive behavior.
- Only the alternate-button combinations receive the constrained `0x1A0` cancellation permission.
- Existing platforms without both flags retain their original transmit lists.

### Longitudinal receive branch

The generic HDA II longitudinal initialization was also expanded to select alternate button receive checks when `CANFD_ALT_BUTTONS` is set. This keeps the safety implementation internally complete for future CAN-FD platforms.

For `KIA_CARNIVAL_2025`, that branch is currently unreachable because the interface forces `alphaLongitudinalAvailable = False` and never sets the Panda `LONG` bit. It must not be treated as a working eSCC or openpilot-longitudinal implementation.

## 7. Vehicle regression test: `test_hyundai.py`

The new `test_carnival_2025_ice_route` test performs these checks:

- Defines the exact camera and radar firmware bytes and proves their pair maps only to `KIA_CARNIVAL_2025`.
- Builds the observed HDA II CAN fingerprint with `0x110`, `0x40`, `0x1A0`, `0x1AA`, and `0x1BA`.
- Supplies a discovered ADAS ECU and deliberately requests Alpha Longitudinal.
- Confirms mass is `2223 kg`, which is the stock platform mass plus standard cargo.
- Confirms BSM is available and the software radar interface is unavailable for the observed route.
- Confirms Alpha Longitudinal remains unavailable despite being requested.
- Confirms `pcmCruise` is true and `openpilotLongitudinalControl` is false.
- Confirms the HDA II alternate steering, alternate buttons, and ICE gear flags.
- Confirms the hybrid flag is absent because `0xFA` is not present.
- Confirms the exact Panda safety parameter contains the three CAN-FD layout bits and does not contain `LONG`.

## 8. Panda regression test: `test_hyundai_canfd.py`

`TestHyundaiCanfdLKASteeringAltButtonsICE` models the Carnival's safety combination.

- It inherits the existing alternate-LKAS steering tests so stock steering torque, request-bit timing, forwarding, and relay-malfunction checks still run.
- It switches the gas parser from EV to the ICE `ACCELERATOR_BRAKE_ALT` signal.
- `_button_msg(...)` constructs received `CRUISE_BUTTONS_ALT` frames for safety-state testing.
- `_acc_cancel_msg(...)` constructs `SCC_CONTROL` cancellation candidates.
- `_lkas_button_msg(...)` maps the LDA button used by the shared MADS tests.
- `test_button_sends` proves every synthetic alternate-button value is rejected regardless of controls state.
- `test_acc_cancel` proves only `ACCMode = 4` with zero acceleration passes, while acceleration and non-cancel modes fail.
- `test_longitudinal_uses_alternate_button_rx` verifies the generic future longitudinal safety branch reads alternate buttons. It does not enable longitudinal control for the Carnival platform.

The import change adding `Buttons` exists only so the final test can refer to symbolic `SET` and `NONE` values.

## 9. Effective behavior compared with stock Sunnypilot

| Area | Stock `release-mici` | Carnival fork |
| --- | --- | --- |
| Vehicle identity | No distinct 2025-26 HDA II Carnival | Exact Canadian camera/radar pair identifies `KIA_CARNIVAL_2025` |
| Physical parameters | No parameters for this identity | Reuses fourth-generation Carnival specifications |
| Steering message | Generic detection, vehicle not identified | Detects and configures HDA II `LKAS_ALT` (`0x110`) |
| Cruise buttons | HDA II LKA-steering path assumes standard `0x1CF` | Receives alternate `0x1AA`; does not synthesize it |
| Cruise cancellation | No applicable transmit list for this combination | Allows constrained `0x1A0` cancellation only |
| Turn signals | Standard lamp fields, except Kona EV 2nd gen | Uses alternate lamp fields for this Carnival |
| Gear parsing | Generic CAN-FD detection | Observed ICE `0x40` selects alternate gears |
| Torque tuning | No entry for new identity | Reuses fourth-generation Carnival tune |
| Longitudinal control | Generic HDA II Alpha Longitudinal may be offered | Permanently unavailable for this platform |
| Factory FCA/AEB ECU | Could be disabled if Alpha Longitudinal were enabled | `0x730` disable path is unreachable for this platform |

## 10. Relevant inherited behavior not introduced by this diff

- Stock Sunnypilot's HDA II controller blocks/replaces factory steering messages and sends a lane-line suppression message so factory LFA does not fight Sunnypilot steering.
- Stock Panda safety continues enforcing steering torque, torque-rate, driver-override, brake/gas disengagement, heartbeat, and relay-malfunction rules.
- Stock controller code creates the `0x1A0` cancellation frame by copying required factory SCC fields, setting `ACCMode = 4`, and setting both acceleration requests to zero.
- No Carnival-specific code transmits braking or acceleration commands.

## 11. Automated upstream maintenance

The workflow `.github/workflows/sync-sunnypilot-release-mici.yml` tracks only Sunnypilot `release-mici`, not `master`.

Because `release-mici` is an unrelated snapshot, the workflow cannot safely run a conventional `git merge`. Instead it:

1. Compares the latest upstream snapshot with the commit recorded in `.github/upstream-release-mici-base`.
2. Replays the fork-only commits onto the new snapshot in a detached test state.
3. Updates the recorded base commit.
4. Runs the complete Hyundai parameter test and compiled Hyundai CAN-FD Panda safety test files.
5. Creates a recovery tag for the previous installer tip.
6. Uses `--force-with-lease` to update the installer branch only if the remote tip has not changed unexpectedly.
7. Leaves the installer branch untouched and alerts `@rapmarz16` through the repository's persistent sync-status issue on any conflict or test failure.

This update mechanism preserves the small, reviewable Carnival delta while avoiding a false merge of Sunnypilot's full `master` history.
