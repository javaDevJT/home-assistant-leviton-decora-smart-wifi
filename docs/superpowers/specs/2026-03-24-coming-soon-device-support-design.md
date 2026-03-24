# Coming Soon Device Support Design

Date: 2026-03-24
Branch: `codex/support-coming-soon-devices`

## Summary

Add support for the remaining README "coming soon" devices by:

- recognizing `DN15S` as a supported switch device
- recognizing `DN6HD` as a supported light device
- recognizing `MLWSB` as a supported bridge device with diagnostic/update-only exposure
- reconciling `D2GF2` support and README documentation
- fixing tightly related classification bugs that would cause the new support to surface incorrect entities

## Goals

- Allow `DN15S`, `DN6HD`, and `MLWSB` to be selected during config flow.
- Expose normal Home Assistant control entities for `DN15S` and `DN6HD`.
- Expose only meaningful bridge-facing entities for `MLWSB`.
- Keep entity behavior driven by existing API capability fields where possible.
- Update the README so supported devices and future plans are accurate.

## Non-Goals

- Refactor the entire device typing system away from model-based gating.
- Add websocket support.
- Add new entity concepts that are not already represented in the API wrapper unless required to prevent incorrect bridge behavior.

## Research Notes

Official Leviton product documentation indicates:

- `DN15S` is a no-neutral Decora Smart switch that depends on `MLWSB`.
- `DN6HD` is a no-neutral Decora Smart dimmer that depends on `MLWSB` and should align with existing light/dimmer behavior.
- `MLWSB` is a Wi-Fi bridge, not a load-controlling wall device.
- `D2GF2` is a smart GFCI receptacle focused on protection, status, and audible alerts rather than normal switched-load control.

These product semantics match the current integration structure:

- config flow only offers `device.is_supported` devices
- device model lists decide whether a device is configurable at all
- most entity behavior is already capability-driven through fields like `canSetLevel`, `fault`, `matterQRCode`, and firmware/update attributes

## Current Code Observations

### Supported-model gate

`custom_components/leviton_decora_smart_wifi/api/const.py` controls support with the `SUPPORTED_DEVICES` table and the derived model lists.

### Entity creation

- `light.py` creates light entities when `device.is_light`
- `switch.py` creates primary switch entities when `device.is_outlet` or `device.is_switch`
- `number.py`, `select.py`, `sensor.py`, `button.py`, `image.py`, and `update.py` create secondary entities from device capability helpers and generic per-device support rules

### Tight coupling bug

`Device.is_switch` currently checks the truthiness of `SUPPORTED_DEVICES_SWITCH` instead of checking membership in that list. As written, any supported non-fan/non-light device can be treated as a switch. This is especially risky for:

- GFCI models such as `D2GF1` and `D2GF2`
- `MLWSB`, which should not surface as a primary switch entity

## Design Decisions

### 1. Add explicit bridge typing

Introduce a bridge-specific device type for `MLWSB`.

Reasoning:

- `MLWSB` needs to be selectable and registered as a supported device
- but it should not inherit switch, outlet, fan, or light behavior
- a dedicated type is the smallest change that keeps the existing model-table approach intact

### 2. Keep DN devices on existing behavior paths

- Map `DN15S` to switch behavior
- Map `DN6HD` to light behavior

Reasoning:

- the integration already derives most behavior from shared helpers
- `DN15S` should align with existing switch-style entities
- `DN6HD` should align with existing dimmer/light entities when `canSetLevel` is true
- avoiding device-specific one-off code reduces regression risk

### 3. Restrict bridge entity exposure

`MLWSB` should expose only generic diagnostic/update surfaces that make sense for a bridge device.

Expected bridge behavior:

- no primary switch entity
- no light entity
- no fan entity
- no load-control configuration entities that assume a switched or dimmed load
- diagnostic/update entities may remain available when backed by API fields

### 4. Fix related classification bugs while in scope

The implementation should include tightly related fixes needed for coherent support:

- fix `Device.is_switch` membership logic
- ensure bridge detection prevents `MLWSB` from being treated as a controllable device
- remove the stale `D2GF2` entry from README future plans and keep it only under supported devices

## Proposed Code Changes

### `api/const.py`

- add a new `DeviceType.BRIDGE`
- add `MLWSB` to `SUPPORTED_DEVICES` with the bridge type
- add `DN15S` to `SUPPORTED_DEVICES` with switch type
- add `DN6HD` to `SUPPORTED_DEVICES` with light type
- derive a `SUPPORTED_DEVICES_BRIDGE` list alongside the existing derived model lists

### `api/device.py`

- add `is_bridge`
- correct `is_switch` to check `self.model in SUPPORTED_DEVICES_SWITCH`
- update `is_supported`/type helpers only as needed to keep bridge models out of normal load-control paths
- preserve existing capability-driven behavior for DN devices

### Entity platforms

Update platform-level support predicates where needed so bridge devices only receive meaningful entities.

Expected result:

- `switch.py`: no primary switch entity for `MLWSB`
- `light.py` / `fan.py`: no control entities for `MLWSB`
- `select.py` / `number.py`: no load configuration entities for `MLWSB`
- `sensor.py` / `button.py` / `update.py`: bridge-safe generic entities can remain if API-backed
- `image.py`: unchanged unless the API actually reports pairing assets for the new models

### `README.md`

- remove `D2GF2` from future plans
- add `DN15S`, `DN6HD`, and `MLWSB` to the supported sections in the appropriate categories
- keep future plans focused on still-unimplemented work only

## Validation Plan

- verify config flow now offers `DN15S`, `DN6HD`, and `MLWSB`
- verify `DN15S` creates a main switch entity and does not create light/fan entities
- verify `DN6HD` creates a main light entity with brightness support when the API reports `canSetLevel`
- verify `MLWSB` creates no primary load-control entity
- verify `D2GF2` and other GFCI devices are not misclassified as primary switch devices after the `is_switch` fix
- run the project’s available verification commands and any targeted static checks

## Risks

- The exact API payload for `MLWSB` is not represented in this repository, so bridge-safe entity filtering should be implemented conservatively.
- Some secondary entities are currently created from broad predicates plus always-present Python properties; bridge handling must account for that to avoid exposing meaningless controls.
- If Leviton uses additional model-specific flags for the DN devices, follow-up adjustments may still be needed after real-world testing.

## Implementation Approach

Use the smallest safe patch set:

1. extend the model/type table
2. add bridge detection and fix switch classification
3. harden entity exposure predicates for bridge devices
4. update README
5. run verification
