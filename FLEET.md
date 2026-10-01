# Zendure Fleet — second SolarFlow 800 Plus alongside Gielz1986 zenSDK

> ## ⚠️ Day-1 requirement — do this BEFORE enabling the layer
> Set the vendor's `input_number.zendure_setting_max_charge_power` **and**
> `input_number.zendure_setting_max_discharge_power` to **800 W**.
> Unit 1 must never try to serve more than itself — everything above 800 W is
> unit 2's job now. (Defense in depth: the split calculation independently
> clamps unit 1's capability to min(vendor helper, device-reported inverter
> limit, 800 W), so a misconfigured helper can no longer skew the split — but
> the vendor's own loop still chases whatever the helper says, so set it.)

Two HA packages that add a second 800 Plus **without touching** the vendor
package or its automation (verified against vendor `v20260824`).

| File | Role |
|---|---|
| `packages/zendure2.yaml` | Unit 2 I/O: IP helper, REST sensors, REST commands, RAM-mode + idle-standby housekeeping |
| `packages/zendure_fleet.yaml` | Coordinator: desired-state sensor, shared write script, slow/fast loops, neutralizer, fleet sensors, contract interlock |

See `MAINTAINING.md` for the fork/upstream-sync workflow.

## Control architecture

One desired-state function, one writer, three small automations:

- **`sensor.zendure2_target_power`** computes unit 2's complete desired power
  for *every* mode — Standby, dynamic price windows, Quick, Manual overflow,
  and the Smart Matching family. If this sensor reads 0, unit 2 must be idle;
  there is no second opinion anywhere else.
- **`script.zendure2_apply`** is the only code path that writes to unit 2.
  Every caller inherits: stop-before-reverse (a direction change always passes
  through a stop first), the write deadband, flash→RAM handling, and a
  re-check of both gates (enabled + dry run) immediately before every REST
  write — including after internal delays.
- **`zendure2_slow_loop`** (default 20 s, tunable) is the *only* path allowed
  to increase unit-2 power in smart-matching modes: unit 1 (vendor, 5 s) stays
  the fast regulator, unit 2 the slow baseload setter.
- **`zendure2_fast_reconcile`** reacts immediately — to every mode change,
  dynamic-flag edge, manual-power change, enable/go-live edge, contract
  failure, and to sustained meter breach (export, or import while charging,
  beyond `zendure2_shed_threshold_w` for 5 s). In smart modes it may only
  shed or stop (**fast shed, slow add**); in quick/manual/window contexts it
  applies the full desired state.
- **`zendure2_neutralize`** makes the kill switch and dry run *interlocks*:
  flipping `zendure2_enabled` off or `zendure2_dry_run` on while unit 2 is
  actively commanded sends one immediate stop.

Smart-matching split (`fleet_target = (z1 + z2) − P1`, an absolute setpoint):
- **|fleet target| ≤ single-unit limit (default 700 W):** one unit runs.
  Unit 1 by default; unit 2 takes over completely when its SoC advantage
  exceeds the lead band (default 8%). **Hysteresis:** the lead engages at the
  full band and releases at half the band; the two-unit split engages above
  the limit and releases 150 W below it. The slow loop additionally holds a
  release (target 0 → stop) for 30 s before writing it.
- **Above the limit:** proportional split by usable headroom using *each
  unit's own* min/max SoC (`soc − min_soc_i` for discharge,
  `max_soc_i − soc` for charge), per-unit clamp 0–800 W with spill-over.
- Per-unit SoC/limit guards; unit 2 covers everything when unit 1 is
  SoC-limited, and vice versa. If the vendor contract is broken or unit-2
  telemetry is missing, the target is 0: **failure means unit 2 idles.**

Mode handling (all through the same target sensor):
- **Dynamic Trading windows** — the vendor's own
  `dynamic_charging_running`/`dynamic_discharging_running` flags take
  precedence over everything except Standby. An *ineligible* window (unit 2
  already full/empty) yields target 0, which the fast reconciler turns into an
  immediate stop — unit 2 can never keep running in the wrong direction
  through a price window.
- **Quick Charge / Quick Discharge** — mirrored to unit 2 **only if**
  `input_boolean.zendure2_mirror_quick_modes` is on (default off). Leave it
  off if existing automations (e.g. a morning sweep) set quick modes with
  single-battery semantics in mind; turn it on for fleet-wide 1600 W quick
  actions.
- **Manual** — unit 2 takes the overflow above 800 W of
  `input_number.zendure_manual_power`; entering Manual with a preset value
  reconciles immediately (no stuck-at-zero).
- **Standby** — immediate stop; full standby follows via the idle-standby
  automation after the vendor's standby delay.

## Coexisting automations that drive the mode select

The fleet layer **never writes** `input_select.zendure_operation_mode` — it
only reads it, so it adds no new writer. Known writers in this install:
the user/dashboard (`zendure_roi_card.yaml` buttons: Quick Charge, Quick
Discharge, Dynamic Trading, Standby) and the morning sweep (sets Quick
Discharge). Every mode edge triggers an immediate full reconciliation of
unit 2, so rapid mode changes converge on the last written mode; there is no
feedback from this layer back into the select, hence no race.
`zendure_strategy_d_shadow.yaml` is log-only and `zendure_helpers.yaml` is
read-only metrics — neither writes to the mode select, the dynamic flags, or
the device. **Action for the morning sweep:** decide whether it should sweep
one unit (leave `zendure2_mirror_quick_modes` off) or both (turn it on).

## Safe bring-up (observe before control)

Two switches gate every write to unit 2; both are created **OFF** by HA, so a
fresh install behaves exactly like the single-battery setup until you act:

- `input_boolean.zendure2_enabled` — master kill switch, and an interlock:
  turning it OFF while unit 2 is active sends an immediate stop.
- `input_boolean.zendure2_dry_run` — observe mode. While ON, the target is
  computed and every would-be action (loop writes and mode/window mirrors) is
  logged with 🧪, but no REST command is sent; turning it ON mid-operation
  also neutralizes unit 2 first. (HA cannot default a toggle to ON without
  re-forcing it on every restart — which would silently disable live control
  in production — so dry run is OFF at creation and the enabled switch
  provides first-load safety instead. Follow the order below.)

Bring-up sequence:
1. Day-1 requirement above (vendor max powers → 800 W).
2. Configure unit 2 (steps below), leave `zendure2_enabled` OFF.
3. Turn `zendure2_dry_run` **ON**, then `zendure2_enabled` **ON**.
4. Run a day or two: compare the 🧪 DRY RUN logbook entries and
   `sensor.zendure2_target_power` against real P1 behavior
   (`sensor.zendure2_applied_power` stays 0 — nothing is written).
5. When the intended splits look right, turn `zendure2_dry_run` **OFF**
   (this itself triggers an immediate reconcile). Unit 2 is now live.

### Hardware shakedown (first live days)

Static review and simulation cannot establish these — watch for them live:
loop stability at 20 s vs 5 s with real ramp/latency; visible hunting at the
700 W or SoC-lead boundaries (raise deadband/interval or widen the band);
behavior of direct `acMode` reversal (the layer always stops first, but
confirm the device settles); prompt reaction to zero-limit and standby
writes; export duration after a sudden load drop (tune
`zendure2_shed_threshold_w`); and what the device does with its last limit if
unit 2 becomes unreachable (the watchdog notifies after 5 minutes, but HA
cannot stop a device it cannot reach).

## Install

1. Keep the vendor package + automation exactly as they are (this fork carries
   them unmodified except the nordpool string shim, see `MAINTAINING.md`).
2. Copy `packages/zendure2.yaml` and `packages/zendure_fleet.yaml` into
   `config/packages/` (packages must be enabled under
   `homeassistant: packages:`). Restart HA.
3. Collision check (one-time): the new layer only creates `zendure2_*` and
   `zendure_fleet_*` names — verified against the vendor package and against
   `zendure_helpers.yaml` / `zendure_strategy_d_shadow.yaml`. Re-confirm
   against the live config (covers files not seen at review time, e.g. the
   morning sweep):
   ```sh
   grep -rn "zendure2_\|zendure_fleet_" config/packages/ config/automations.yaml \
     config/scripts.yaml --include="*.yaml" -l 2>/dev/null \
     | grep -v "zendure2.yaml\|zendure_fleet.yaml"
   ```
   No output = no collisions.
4. Set `input_text.zendure2_setting_ip_address` to unit 2's IP (give it a DHCP
   reservation).
5. Helpers (fresh installs start at the value in parentheses):
   - `zendure2_setting_max_charge_power` / `..._discharge_power` → 800 (0)
   - `zendure_fleet_soc_lead_band` → 8 (2)
   - `zendure_fleet_single_unit_limit` → 700 (100)
   - `zendure2_loop_interval_s` → 20 (20, also the floor)
   - `zendure2_write_deadband_w` → 25 (25, also the floor)
   - `zendure2_shed_threshold_w` → 200 (100, more sensitive = safer)
   - `zendure2_mirror_quick_modes` → your choice (off = quick modes stay
     single-unit; see the mode-select section above)
6. Follow the bring-up sequence above.

## Surviving a Gielz version bump

- Vendor files stay pristine (one isolated shim commit — see `MAINTAINING.md`
  for the tag-by-tag upstream merge workflow).
- This layer only reads the vendor entities listed at the top of
  `zendure_fleet.yaml` (the "contract").
- `binary_sensor.zendure_fleet_vendor_contract_ok` re-checks that list
  continuously and is a **control interlock**: while it is off, the target is
  forced to 0 and the fast reconciler stops unit 2 — unit 2 idles, unit 1
  keeps working. The `missing` attribute names the culprit and the watchdog
  sends a persistent notification after 5 minutes.

## Fleet sensors

- `sensor.zendure_fleet_power` — combined signed battery power (+charge/−discharge)
- `sensor.zendure_fleet_state_of_charge` — capacity-weighted SoC
- `sensor.zendure_fleet_capacity` — combined kWh
- `sensor.zendure_fleet_available_energy` — usable kWh above each unit's own min SoC
- `sensor.zendure_fleet_required_energy` — kWh of headroom to each unit's own max SoC

## Tuning

- Oscillation at the 1-unit/2-unit boundary → raise `zendure2_write_deadband_w`
  or `zendure2_loop_interval_s`; the built-in hysteresis (150 W on the
  single-unit limit, half-band on the SoC lead, 30 s release hold) should make
  this rare.
- Units ping-ponging lead at low power → widen `zendure_fleet_soc_lead_band`.
- Second unit engaging too late in evening peaks → lower
  `zendure_fleet_single_unit_limit` (e.g. 500 W).
- Export spikes after load drops lasting too long → lower
  `zendure2_shed_threshold_w`.
