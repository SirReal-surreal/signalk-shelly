# Shelly thermostat (Signal K plugin) – specification

Replaces a Node-RED flow that switches a 12 V Shelly depending on a temperature.
Status: **draft, open questions below** – to be filled in together with the user.

## Goal

A Signal K plugin that reads a temperature from Signal K and switches a Shelly relay with on/off thresholds (hysteresis).
The decision logic is pure and tested; the Shelly access is a thin, replaceable layer.

## Open questions

1. Old Node-RED flow (file) – thresholds, timings, special cases.
2. Shelly model and generation (Gen1 `/relay/0?turn=on` vs. Gen2+ `/rpc/Switch.Set`), IP address, authentication.
3. Temperature source: Signal K path (e.g. `environment.inside.*.temperature`), unit (K in Signal K), `$source`.
4. Purpose (heating / fan / other) – heating or cooling logic (on below / on above the threshold).
5. Failsafe: sensor value missing for N s -> switch off / on / keep last state? Alarm notification?
6. Shelly unreachable: retry, notification, state path for KIP.
7. Manual override (PUT path), minimum on/off times to protect the relay.

## Decisions

(none yet)
