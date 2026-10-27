# Shelly thermostat (Signal K plugin) – specification

Replaces the Node-RED flow `2025_10_04_shelly_engine_temperature_signalk_flow.json` (kept in the repo for reference).
Status: **draft – main decisions agreed, all questions answered; two assumptions and one device check remain (see end). Nothing is built yet.**

## What the existing flow does

Purpose: connect the charging path for the hotel battery when the engine is warm ("hot enough to start charging").

- **Device:** Shelly Gen2+ (device type `S4SW-001X16EU`), host `10.10.10.155`, switch id 0. Commands via `Switch.Set {id:0,on:true|false}`; the real state comes back from the Shelly as events (`switch0.output`).
- **Temperature:** `environment.inside.engine.temperature` (Kelvin), subscribed with 1 s period, converted to °C.
- **Rule:** threshold **40 °C**, *one* threshold, no hysteresis.
  - temperature >= 40 °C and relay off -> relay **on**
  - temperature < 40 °C and relay on -> relay **off**
  - only while `automatic` is true.
- **Signal K paths (read and PUT):**
  - `electrical.batteries.hotel.shelly` (boolean): relay state, published from the Shelly events. PUT `true`/`false` switches the relay manually.
  - `electrical.batteries.hotel.automatic` (boolean): PUT `true` = automatic, `false` = manual. Published as state.
- **At start (INIT):** `automatic = true` and the relay is switched **on** regardless of the temperature. The next temperature value then switches it off if it is below 40 °C.
- **Internal state:** the flow remembers `charging` (relay) in flow context and sets it when the command is *sent*, not when the Shelly confirms.

## Observations (weak points of the flow)

1. No hysteresis: around 40 °C the relay can flap, one switching command per crossing.
2. No handling of a missing/old temperature value: the relay simply keeps its last state.
3. A manual ON/OFF in automatic mode is overridden by the next temperature value (within about a second). Manual control only sticks when `automatic` is false.
4. The internal state can differ from the real relay if a command fails; there is no error or alarm when the Shelly is unreachable.
5. After a restart `automatic` is true again (not persisted); the relay is switched on at start.

## Decisions (agreed 2026-10-27)

1. **Hysteresis:** relay **on at 45 °C or more**, **off at 40 °C or less**; between 40 and 45 °C the relay keeps its state. (Boundary values 45 / 40 inclusive are an assumption, to be confirmed.)
2. **Start behaviour:** the relay is **off at start** ("charging only with a warm engine"). In automatic mode the plugin switches the relay off when it starts and only then lets the temperature logic run. The Shelly itself also has "off at power-up" configured.
3. **Sensor loss:** temperature missing/too old -> **charging off** (relay off). The relay stays off until valid temperature data arrives and the on-threshold is reached again.
4. **Paths stay exactly as in the old flow** so existing KIP buttons keep working:
   - `electrical.batteries.hotel.shelly` (boolean, state + PUT)
   - `electrical.batteries.hotel.automatic` (boolean, state + PUT)
5. **Operating mode is a plugin setting** (decided in the plugin configuration, not at runtime):
   - **Manual only:** the plugin only provides the paths and the switch. The relay is switched on/off by PUT (e.g. KIP switch). No temperature logic, no subscription to the temperature, no `automatic` path (or it is not offered).
   - **Automatic:** the temperature logic runs with the hysteresis above, plus the `automatic` path to switch between automatic and manual operation at runtime.

6. **Sensor loss timeout: 60 s** without a valid temperature -> charging off (setting).
7. **Minimum time between two switching commands: 30 s** (setting). It applies to the temperature logic; an explicit manual command is not delayed (to be confirmed when specifying the details).
8. **Shelly unreachable / command failed:** a **warning notification** (state `warn`, no alarm) on a notification path, cleared when the Shelly answers again. Text and retry interval are details to specify.
9. **Manual command in automatic mode:** a manual command (PUT on `...hotel.shelly`) switches to manual mode by itself (`automatic = false`). `automatic` is **not persisted**: after a restart the **plugin setting** applies (setting "automatic" -> automatic again, relay off at start, then the logic runs).
10. **Manual-only mode at start:** the relay is switched **off** at start.

11. **Shelly access:** address `10.10.10.155`, no password. The plugin uses the Shelly's **events** (websocket) like the Node-RED node did. The exact API calls and the event format are to be checked against the Shelly documentation (and against the real device) before coding.
12. **Plugin settings:** mode (manual only / automatic at start), on threshold 45 °C, off threshold 40 °C, temperature path, sensor timeout 60 s, minimum switching interval 30 s, Shelly address, switch id, credentials if needed.
13. **Relay state and failures:** `electrical.batteries.hotel.shelly` shows the **real state reported by the Shelly**. A difference between the commanded and the real state, or a failed command, is retried and raises a warning: **3 attempts, 10 s apart**, then a `warn` notification (cleared when the Shelly answers/confirms again). Attempts and interval become settings with these defaults.

## Open points

- Check the Shelly API/events against the documentation and the real device before the first implementation (decision 11).
- Boundary values of the hysteresis (45 °C and 40 °C inclusive, decision 1) and whether a manual command bypasses the 30 s minimum interval (decision 7) are assumptions to confirm during review.
