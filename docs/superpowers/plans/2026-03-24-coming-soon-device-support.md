# Coming Soon Device Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add support for `DN15S`, `DN6HD`, and `MLWSB`, keep `MLWSB` diagnostic-only, and reconcile the stale `D2GF2` README state without regressing existing model classification.

**Architecture:** Extend the model/type registry in the API layer, add bridge-aware device predicates in `Device`, and reuse those predicates from entity-platform support rules so the bridge can be configured without surfacing load-control entities. Cover the new behavior with a small pure-Python `unittest` module that exercises model membership and bridge gating at the device predicate layer.

**Tech Stack:** Python 3, Home Assistant custom integration structure, standard-library `unittest`

---

### Task 1: Add failing regression tests for the new model classifications

**Files:**
- Create: `tests/test_device_support.py`
- Modify: `custom_components/leviton_decora_smart_wifi/api/const.py`
- Modify: `custom_components/leviton_decora_smart_wifi/api/device.py`
- Test: `tests/test_device_support.py`

- [x] **Step 1: Write the failing test**

```python
import importlib
import importlib.util
import sys
import unittest
from pathlib import Path
from types import SimpleNamespace

sys.modules.setdefault(
    "pyqrcode",
    SimpleNamespace(create=lambda *_args, **_kwargs: SimpleNamespace(png=lambda *a, **k: None)),
)

API_ROOT = (
    Path(__file__).resolve().parents[1]
    / "custom_components"
    / "leviton_decora_smart_wifi"
    / "api"
)

if "leviton_test_api" not in sys.modules:
    spec = importlib.util.spec_from_file_location(
        "leviton_test_api",
        API_ROOT / "__init__.py",
        submodule_search_locations=[str(API_ROOT)],
    )
    package = importlib.util.module_from_spec(spec)
    sys.modules["leviton_test_api"] = package
    spec.loader.exec_module(package)

Device = importlib.import_module("leviton_test_api.device").Device


def make_device(model: str, **data):
    residence = SimpleNamespace(id=1, rooms=[])
    base = {
        "id": 1,
        "model": model,
        "name": model,
        "power": "OFF",
        "canSetLevel": False,
        "connected": True,
    }
    base.update(data)
    return Device(api=SimpleNamespace(call=lambda **kwargs: None), residence=residence, data=base)


class DeviceSupportTest(unittest.TestCase):
    def test_new_models_have_expected_primary_types(self):
        self.assertTrue(make_device("DN15S").is_switch)
        self.assertTrue(make_device("DN6HD", canSetLevel=True).is_light)
        self.assertTrue(make_device("MLWSB").is_bridge)

    def test_gfci_and_bridge_are_not_primary_switches(self):
        self.assertFalse(make_device("D2GF2").is_switch)
        self.assertFalse(make_device("MLWSB").is_switch)
```

- [x] **Step 2: Run test to verify it fails**

Run: `python3 -m unittest tests.test_device_support -v`
Expected: FAIL because `DN15S`, `DN6HD`, and `MLWSB` are not yet recognized and `Device` has no bridge predicate.

- [x] **Step 3: Write minimal implementation**

```python
class DeviceType(StrEnum):
    BRIDGE = "bridge"


SUPPORTED_DEVICES.append(
    {DEVICE_MODEL: "DN15S", DEVICE_TYPE: [DeviceType.SWITCH], ...}
)
SUPPORTED_DEVICES.append(
    {DEVICE_MODEL: "DN6HD", DEVICE_TYPE: [DeviceType.LIGHT], ...}
)
SUPPORTED_DEVICES.append(
    {DEVICE_MODEL: "MLWSB", DEVICE_TYPE: [DeviceType.BRIDGE], ...}
)


@property
def is_bridge(self) -> bool:
    return bool(self.model in SUPPORTED_DEVICES_BRIDGE)


@property
def is_switch(self) -> bool:
    return all(
        [
            self.model in SUPPORTED_DEVICES_SWITCH,
            not self.is_fan,
            not self.is_light,
        ]
    )
```

- [x] **Step 4: Run test to verify it passes**

Run: `python3 -m unittest tests.test_device_support -v`
Expected: PASS

- [x] **Step 5: Commit**

```bash
git add tests/test_device_support.py custom_components/leviton_decora_smart_wifi/api/const.py custom_components/leviton_decora_smart_wifi/api/device.py
git commit -m "Add support for DN and bridge device models"
```

### Task 2: Add failing tests for bridge-safe load-configuration predicates

**Files:**
- Modify: `tests/test_device_support.py`
- Modify: `custom_components/leviton_decora_smart_wifi/api/device.py`
- Modify: `custom_components/leviton_decora_smart_wifi/select.py`
- Test: `tests/test_device_support.py`

- [x] **Step 1: Write the failing test**

```python
class DeviceSupportTest(unittest.TestCase):
    def test_bridge_is_diagnostic_only_for_load_configuration(self):
        bridge = make_device("MLWSB")
        dimmer = make_device("DN6HD", canSetLevel=True)

        self.assertFalse(bridge.supports_auto_shutoff)
        self.assertFalse(bridge.supports_status_led_behavior)
        self.assertTrue(dimmer.supports_auto_shutoff)
        self.assertTrue(dimmer.supports_status_led_behavior)
```

- [x] **Step 2: Run test to verify it fails**

Run: `python3 -m unittest tests.test_device_support -v`
Expected: FAIL because the new support predicates do not exist yet.

- [x] **Step 3: Write minimal implementation**

```python
@property
def supports_auto_shutoff(self) -> bool:
    return not self.is_bridge and not self.has_motion_sensor and not self.is_gfci


@property
def supports_status_led_behavior(self) -> bool:
    return not self.is_bridge and not self.is_controller and not self.is_gfci
```

```python
LevitonSelectEntityDescription(
    key="auto_shutoff",
    ...,
    is_supported=lambda device: device.supports_auto_shutoff,
)

LevitonSelectEntityDescription(
    key="status_led_behavior",
    ...,
    is_supported=lambda device: device.supports_status_led_behavior,
)
```

- [x] **Step 4: Run test to verify it passes**

Run: `python3 -m unittest tests.test_device_support -v`
Expected: PASS

- [x] **Step 5: Commit**

```bash
git add tests/test_device_support.py custom_components/leviton_decora_smart_wifi/api/device.py custom_components/leviton_decora_smart_wifi/select.py
git commit -m "Restrict bridge devices to diagnostic entities"
```

### Task 3: Update documentation to match the supported-device set

**Files:**
- Modify: `README.md`
- Test: `README.md`

- [x] **Step 1: Write the failing documentation checklist**

```text
README must:
- list D2GF2 only under supported GFCI outlets
- list DN15S under supported switches
- list DN6HD under supported lights
- list MLWSB in a dedicated supported bridge/controller section
- remove all three from Future Plans
```

- [x] **Step 2: Verify the checklist currently fails**

Run: `sed -n '1,220p' README.md`
Expected: The supported-device lists do not include all new models and Future Plans still mentions them.

- [x] **Step 3: Write minimal implementation**

```markdown
### Bridges
- MLWSB

### Lights
- DN6HD

### Switches
- DN15S
```

```markdown
## Future Plans
- Websocket support to allow for cloud push updates
- Control of night settings start/end time
```

- [x] **Step 4: Verify the checklist now passes**

Run: `sed -n '1,220p' README.md`
Expected: README supported-device sections and Future Plans match the checklist.

- [x] **Step 5: Commit**

```bash
git add README.md
git commit -m "Update README for newly supported devices"
```

### Task 4: Run full verification and prepare the branch for handoff

**Files:**
- Test: `tests/test_device_support.py`
- Test: `custom_components/leviton_decora_smart_wifi`
- Modify: `docs/superpowers/plans/2026-03-24-coming-soon-device-support.md`

- [x] **Step 1: Run the targeted unit tests**

Run: `python3 -m unittest tests.test_device_support -v`
Expected: PASS

- [x] **Step 2: Run a syntax verification pass**

Run: `python3 -m compileall custom_components tests`
Expected: PASS with no syntax errors

- [x] **Step 3: Review the diff against the spec**

Run: `git diff --stat 91615f7..HEAD`
Expected: Changes limited to the API model registry, device predicates, bridge-safe entity gating, tests, and README updates.

- [x] **Step 4: Mark completed work in the plan**

```markdown
- [x] Step 1: Run the targeted unit tests
- [x] Step 2: Run a syntax verification pass
- [x] Step 3: Review the diff against the spec
```

- [x] **Step 5: Commit**

```bash
git add docs/superpowers/plans/2026-03-24-coming-soon-device-support.md
git commit -m "Record implementation plan completion"
```
