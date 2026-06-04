# Cascade / multi-zone Gree support (uncommitted local patch)

Local working-tree changes adding **central/cascade Gree system** support to the
library (one wifi gateway proxying several indoor heads). Not committed — this
file documents the diff so it can be reconstructed or turned into a PR later.

## The problem

The library assumes a flat topology: **one wifi module = one AC**, with each
device reachable on its own MQTT topic. Cascade/multi-zone systems break that.

Per home, the Gree cloud returns a mix of:

- **gateways** — the wifi modules actually connected to the cloud broker. Real
  Gree OUI MAC `58:0D:0D…`, 12 hex chars, e.g. `580d0d37023e`.
- **heads** — individual indoor units behind a gateway. 14-char MACs ending in
  `00`, e.g. `9abbc91e000000`. No independent cloud presence; everything is
  relayed via their gateway. All devices behind a gateway share its **encryption
  key**.

`_filter_duplicate_devices()` keeps the heads and drops the gateways; then
`CloudDevice._detect_parent_mac()` derives a head's parent as `headMac[:-2]` — a
topic nobody serves. Every status request times out and no state is ever read.

Verified on a real system (3 gateways → 9 heads): subscribing/publishing on the
**gateway** topic makes head state stream in on `status/<gatewayMac>/<headMac>`,
and commands to `request/<gatewayMac>` with `tcid=<headMac>` reach the right head.

## Changes in THIS repo (`greeclimate`)

- **`greeclimate/cloud_api.py`**
  - `CloudDeviceInfo` gains an optional `parent_mac: Optional[str] = None` field.
  - `GreeCloudApi.get_all_devices()`: after dedup, build a `key → gateway MAC`
    map (gateway = the member of each encryption-key group whose MAC is **not** a
    14-char `…00` head) and stamp `parent_mac` onto each head. Heads with no
    distinct gateway (true standalone units) are left untouched (`parent_mac`
    stays `None`).

- **`greeclimate/cloud_device.py`**
  - `_handle_mqtt_message()`: child-scoped routing guard. When the device is a
    cascade head (`self._is_cascade`, set by the consumer after construction),
    only accept `status/`/`response/` messages whose topic's 3rd path segment
    equals `self._child_mac`. Heads behind one gateway share a parent topic **and**
    a decryption key, so without this guard a sibling would happily decrypt and
    apply another head's state.

- **`greeclimate/device.py`**
  - `target_temperature` getter: previously returned `None` if **either** `SetTem`
    **or** `TemRec` was missing. Cascade/commercial heads report `SetTem` but **not**
    `TemRec` (the Fahrenheit half-degree bit) nor `TemUn`, so the setpoint never
    surfaced and the Home Assistant climate card hid the temperature control
    entirely. Now only `SetTem` is required; a missing `TemRec` defaults to `0`
    (it is ignored in Celsius by `_convert_to_units`, and `0` is the natural
    default in Fahrenheit). Standalone units that do report `TemRec` are
    unaffected. Note: these heads also omit `TemSen`, so `current_temperature`
    falls back to the setpoint (existing library behaviour) — display-only.

The consumer is responsible for setting `device._parent_mac` (to the stamped
gateway MAC) and `device._is_cascade = True` before `bind()`. Once `_parent_mac`
is a gateway MAC, the existing topic logic — `subscribe_to_device(parent)` →
`status/<parent>/#` etc., and `publish_command(parent, …, target=child)` →
`request/<parent>` with `tcid=child` — is already correct; no MQTT-client changes
are needed.

## Companion changes (separate repo: `homeassistant-gree-cloud`)

The Home Assistant integration consumes the above, see
`../homeassistant-gree-cloud/CASCADE_FIX.md`:

- `coordinator.py` overrides `device._parent_mac` / sets `_is_cascade` from the
  new `parent_mac` field before bind, and downgrades the now-expected per-cycle
  "timeout" warning to debug.
- `const.py` fixes the North American broker host (`mqtt-us` → `mqtt-na.gree.com`).

## Behaviour notes / known edge

- Gateways **push** state on `status/` (initial retained snapshot + on change);
  they do **not** ack the explicit status request on `response/`. State is applied
  via the unsolicited path, so the request's wait legitimately times out each
  cycle (the integration logs this at debug).
- The initial retained snapshot can race for sibling heads that bind after the
  first head on a gateway (their handler isn't registered yet when the retained
  message is delivered), but they populate within a poll cycle / on next change.
  A clean upstream fix would subscribe each head to its child-specific topic
  (`status/<gateway>/<child>`) so retained delivery is per-head.

## How to turn this into a PR

This library change should land **first**; the integration PR bumps its pinned
greeclimate ref to pick it up. `origin` = your fork (GTez), `upstream` = davo22;
`master` was identical to the deployed tag `1.0.3`, so the diff applies cleanly.
