# Brief: Zigbee Device Manager GUI for smrekar-platform

Paste this into Claude Code in the smrekar-platform repo.

---

## Context

I have a Spring Boot + React monorepo (`smrekar-platform`) that was designed
but never deployed — built to unify cabin and home locations with shared
core services, location config overlays, and a unified UI. It includes
`DeviceRegistry`, `HomeAssistantAdapter` (multi-instance aware),
`MqttBridgeService`, `KafkaEventPublisher`, `AutomationRuleService` on the
backend, and a React frontend with `MonitoringPanel`, `DeviceManagerPanel`,
`RulesEnginePanel`, `FamilyHubPanel`, `FamilyConfigPanel`.

Separately, the cabin location now has a real, deployed Home Assistant +
Zigbee2MQTT stack (M920q, Docker Compose) with paired sensors (leak
detection, freeze monitoring, door/motion sensors, a Zigbee-controlled main
water valve actuator, a mechanical room heater plug, and an intrusion
deterrence setup). The full automation logic already lives in Home
Assistant's `automations.yaml` (leak alerts, freeze-triggered heater
control, valve auto-shutoff, intrusion deterrence, RF tripwire) — **do not
duplicate this logic in the platform.** The platform's job is to give a
clean, resilient GUI on top of what Home Assistant/Zigbee2MQTT are already
doing, not reimplement the automation engine.

## Goal

Extend `smrekar-platform` with a Zigbee device management interface inside
`DeviceManagerPanel`, backed by a new `Zigbee2MqttAdapter`, that:

1. Lets a non-technical user see, add, remove, and configure Zigbee devices
   without understanding Zigbee2MQTT's native vocabulary
2. Is resilient to MQTT disconnects, Home Assistant restarts, Zigbee2MQTT
   updates changing entity shapes, and new/unrecognized device types —
   degrades gracefully instead of breaking
3. Supports theme switching (color palette + font, independently
   selectable)

## Task 1 — Backend: Zigbee2MqttAdapter

Create `platform-core/src/main/java/.../adapters/Zigbee2MqttAdapter.java`.

- Connects via the existing `MqttBridgeService` (extend it if it isn't
  already generic enough to support a second topic namespace alongside
  whatever it currently handles)
- Subscribes to `zigbee2mqtt/bridge/devices` (full device list),
  `zigbee2mqtt/bridge/state` (bridge online/offline heartbeat), and
  per-device state topics
- Publishes to `zigbee2mqtt/bridge/request/permit_join`,
  `zigbee2mqtt/{friendly_name}/set` (config changes), and device removal
  requests
- Translates raw Z2M device JSON into a `DeviceDescriptor` (existing model)
  — map Zigbee `exposes` capabilities to `DeviceCapability` entries
- **Do not assume every device has the same shape.** Some are simple
  switches, some (like the leak sensors) have a `water_leak_buzzer`
  sub-property that's independent of the main detection state — the
  adapter should expose sub-capabilities generically rather than
  hardcoding per-device-type logic wherever reasonably possible

## Task 2 — Backend: DeviceHealthMonitor

New class, `platform-core/.../device/DeviceHealthMonitor.java`.

- Tracks MQTT connection state with exponential backoff reconnect
- Caches the last-known-good device list in `DeviceRegistry` so a
  transient MQTT drop doesn't blank the UI — serve cached state with a
  `staleSince` timestamp when live data isn't available
- Exposes `GET /api/system/health` → 
  `{ "zigbee2mqtt": "healthy"|"degraded"|"offline", "lastSeen": <ISO8601> }`
- On receiving a device whose `exposes` shape doesn't match any known
  capability type, don't drop it or throw — register it as a generic
  `UnknownDevice` capability (raw state passthrough, on/off if a `state`
  property exists) so the frontend can render *something* instead of
  erroring

## Task 3 — Backend: API endpoints

Extend `DeviceController` (or add a new controller) with:

| Method | Path | Action |
|---|---|---|
| GET | `/api/devices` | List all devices + current state (See) |
| GET | `/api/devices/{id}` | Single device detail |
| PATCH | `/api/devices/{id}` | Update device config (Change) |
| POST | `/api/devices/permit-join` | Open pairing window (Add) |
| DELETE | `/api/devices/{id}` | Unpair (Remove) |
| GET | `/api/devices/{id}/config` | Get per-device settings (reporting interval, thresholds, friendly name) |
| PATCH | `/api/devices/{id}/config` | Update per-device settings — "How should the IoT children behave?" |
| GET | `/api/system/health` | Zigbee2MQTT bridge health |

## Task 4 — Frontend: DeviceManagerPanel

Extend the existing panel with this navigation structure:

- **L1 landing:** "Do things with devices"
- **L2:** four entry points — See / Change / Add / Remove
- **L3 (within See → select a device, or Change → select a device):**
  "How should the IoT children behave?" — renders the device's config
  form (from `/api/devices/{id}/config`)

Build order within this task:
1. **See only** first — get real device data rendering (list + status)
   before adding any write actions. Verify against the live cabin stack
   before proceeding.
2. Add the health badge (small, persistent, non-modal) using
   `/api/system/health`
3. Add Change (config PATCH)
4. Add Add (permit-join flow — show a countdown timer matching Z2M's
   default ~254s join window, with a live-updating "new device found"
   list as devices join). **On successful pairing of a device, present
   three explicit next-action choices — don't auto-advance or assume:**
   - **"Add another"** — re-opens the permit-join window immediately,
     stays in the Add flow
   - **"Configure it now"** — jumps straight into that device's L3
     config view (the one just paired)
   - **"Check out all devices"** — returns to the L2 landing / device
     browser (the See view, showing Active / Ready to roll / Out of
     Action status for everything, not just what was just paired)
5. Add Remove (with a confirmation step — this is destructive)

**Escape hatch:** add a low-prominence "Advanced" link that opens the raw
Zigbee2MQTT frontend in an iframe, with an overlay back button pinned to
the top of the iframe. This is for settings the panel doesn't cover yet —
not the primary path.

**Unknown/generic devices:** render `UnknownDevice` capability types with a
simple card — name, raw state as formatted JSON, an on/off toggle if a
`state` property exists. Don't let one unrecognized device type break the
list rendering for everything else.

## Task 5 — Frontend: ThemeProvider

New file `ui/src/ThemeProvider.jsx` (React context).

- CSS custom properties at root: `--bg`, `--surface`, `--accent`,
  `--font-display`, `--font-mono`, plus a couple of semantic tokens
  (`--success`, `--warning`, `--danger`) so status colors stay consistent
  across themes
- Palette and font are **independently selectable**, not locked to a
  single bundled theme
- Ship these starter presets:
  - **LCARS** (Star Trek — warm oranges/purples, bold rounded sans)
  - **Monolith** (2001: A Space Odyssey — stark black/white, minimal, high
    contrast, geometric serif or monospace)
  - **Retro-CRT** (green/amber phosphor on black, scanline texture,
    monospace)
  - **Bluefin-mono** (terminal-forward font stack matching Fedora
    Bluefin's aesthetic — JetBrains Mono or similar, cool neutral palette)
  - **Modern** (clean default — current design system, whatever that is)
- Persist selection (localStorage is fine to start; move to a per-user
  config endpoint later if multi-user preference sync matters)
- Theme picker UI: a small settings affordance, not a prominent feature —
  this is a nice-to-have layer over a working device manager, not the
  headline

## Explicit non-goals for this task

- Don't rebuild or duplicate any Home Assistant automation logic — HA
  remains the automation engine, this platform is a management UI on top
- Don't try to replicate every Zigbee2MQTT admin feature — the iframe
  escape hatch covers the long tail
- Don't block Task 1-4 on Task 5 (theming) — device management working
  correctly is the actual goal; theming is polish

## Reference

Live cabin stack for testing against: Home Assistant + Zigbee2MQTT on the
M920q, reachable via Tailscale. Zigbee2MQTT frontend already running at
port 8080 for direct comparison while building the adapter. Automations
and pairing documentation live in the `CabinAutomations` repo
(`ImpressiveLLC/CabinAutomations`) if device naming conventions or
capability details are needed for reference.
