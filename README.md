# Multicontrol

Home Assistant blueprints that keep the on/off state of several relays in sync without self-sustained oscillation, and how to synchronise switches natively over Zigbee with an alternative firmware.

## The problem

When a relay is both the source of the information (its state change triggers the synchronisation) and the consequence of it (it receives the commands), the automation is a closed loop with a pure delay inside (command → device → state report). With asynchronous reports and bursts of user actions, such a loop can oscillate: the lights keep blinking on their own.

Two ways out:

1. Decouple source and consequence in the device itself: native Zigbee synchronisation with the alternative firmware below.
2. Where that is not possible, mask the state changes that are consequences of the automation: the blueprints of this repository.

## Native Zigbee synchronisation (alternative firmware)

The custom firmware [romasku/tuya-zigbee-switch](https://github.com/romasku/tuya-zigbee-switch) for Tuya-based Zigbee switches separates the button from the relay. For an N-gang switch, endpoints 1…N are the switch endpoints (On/Off client) and endpoints N+1…2N the relay endpoints (On/Off server). For a 1-gang switch: endpoint 1 = button, endpoint 2 = relay.

The firmware sends On/Off commands to its bindings only when the physical button is used, not when its relay is switched by a received command. Switches bound to each other therefore cannot form a loop.

Setup in Zigbee2MQTT, for two switches A and B:

- Flash the firmware (see the [supported devices](https://github.com/romasku/tuya-zigbee-switch/blob/main/docs/supported_devices.md) and the project documentation) and load the project's external converters.
- On each switch, set `Switch action mode` to `toggle_smart_sync`: a press toggles the relay and sends ON or OFF, matching the relay state, to the bindings.
- In the Bind tab of A: source endpoint 1 → B, destination endpoint 2 (relay), cluster `genOnOff`. Same from B to A.
- Set `Switch relay mode` to `press_start` (keep `Switch binded mode` at `short_press`). With `short_press` and a direct binding, the pressed switch toggles its relay but its new state is not reported: Zigbee2MQTT and Home Assistant keep the old state until it is read again (refresh). The bound switches report normally ([issue #548](https://github.com/romasku/tuya-zigbee-switch/issues/548), same symptom as [#74](https://github.com/romasku/tuya-zigbee-switch/issues/74)). With `press_start`, the relay toggles at press and the bound command is sent at release, and the state is reported.
- Side effect of `press_start`: a long press toggles the local relay but sends nothing to the bindings, so the switches stay out of sync until the next short press. A long `Switch long press duration` (for example 5000 ms) makes this unlikely.
- For more than two switches, the firmware documentation describes binding to a Zigbee group. It also lists that end devices do not respond to group commands: use the router build.

A command sent from Home Assistant to one relay is not propagated by the bindings: only physical presses are. This is what the `slave` and `hybrid` blueprints below take care of. They never read the state of the slave relays, so a missing state report (issue #548 above) does not affect them.

## The blueprints

All three share the same masking rule:

- The first relay state change that is not masked acts immediately.
- Every later state change re-arms a 1 s masking delay and is ignored, until there has been 1 s without any state change.

This is implemented with `mode: restart`: every state change restarts the run, and only a run started outside the masking delay sends commands.

### `multicontrol.yaml`

Input: `linked_entities`. Every entity is both source and consequence. The first unmasked change to on or off is sent to all the other linked entities.

### `multicontrol_slave.yaml`

Inputs: `relays`, `slave_relays`.

- `relays`: devices whose state changes are processed.
- `slave_relays`: relays of switches synchronised among themselves by native Zigbee bindings. Their state is never used by the automation and their changes trigger nothing.

The first unmasked change of a relay is sent immediately to the other relays and to the slave relays.

### `multicontrol_hybrid.yaml`

Inputs: `relays`, `switch_press`, `slave_relays`, `state_bit`.

- `switch_press`: the "Switch press action" entities of the switches running the alternative firmware.
- `state_bit`: an `input_boolean` helper, one per group, used as internal state bit.

Behaviour:

- A press (`press`) toggles the internal bit. The resulting on or off is sent to the `relays` only; the slave relays follow by themselves through the Zigbee bindings. A press is never masked, but it starts the masking delay.
- The first unmasked change of a relay sets the bit and is sent immediately to the other relays and to the slave relays.
- 1 s after the last state change (press or relay), the slave relays are forced to the state of the bit (anti-desynchronisation).
- The state of the slave relays is never used.

## Why not filter on the origin reported by Home Assistant

The original blueprint had a commented-out condition comparing `trigger.to_state.context` with `this.context`, to ignore the state changes caused by the automation itself. This is not reliable:

- `this` is the state of the automation captured when the trigger fires, before the context of the new run is applied. `this.context` is the context of an earlier state write of the automation, not the context of the run that sent the commands (each run gets a new context).
- A state change is linked to a command only through the context of the entity: when a service call targets an entity, the entity keeps that context for 5 s (`CONTEXT_RECENT_TIME_SECONDS` in Home Assistant). A device report arriving more than 5 s after the command gets a new context and looks like a manual change; a manual change arriving within 5 s of a command inherits the command's context and looks like a consequence of the automation.

The context tells which call last touched the entity, not what caused the reported state. Hence the time-based masking below.

## Why it does not oscillate

Let τ be the pure delay between a command sent by Home Assistant and the reception of the state change it causes.

- Commands to the relays are sent only at an event that starts the 1 s masking delay: the first unmasked relay change, or a press.
- Their echo comes back after τ. If τ < 1 s, it arrives while the masking delay is running: it is ignored and only re-arms the delay.
- Commands to the slave relays never come back as information: their state is not watched, and the firmware does not send bound commands when its relay is switched remotely.

No command can cause another command: for any pure delay below 1 s, the loop is open.

## Limitation

A genuine relay change that arrives during the masking delay is ignored: no oscillation, but the group can stay out of sync. One new action outside the delay re-synchronises the whole group; if the resulting state is the opposite of the expected one, a second press gives it. In the hybrid blueprint a press is never ignored.

## Installation

- Requires Home Assistant 2024.10.0 or newer (automation syntax with `triggers:` / `actions:`).
- Import each blueprint in Home Assistant (Settings → Automations & scenes → Blueprints → Import blueprint) with the URL of its file in this repository.
- Hybrid: create one `input_boolean` helper per group and set it to the current state of the group before enabling the automation.

## Credits

- Original blueprint: [Synchronize the on/off state of 2 entities](https://community.home-assistant.io/t/synchronize-the-on-off-state-of-2-entities/259010) by [@adchevrier](https://community.home-assistant.io/u/adchevrier), template by [@hebus](https://community.home-assistant.io/u/hebus) ([post 38](https://community.home-assistant.io/t/synchronize-the-on-off-state-of-2-entities/259010/38)).
- Firmware: [romasku/tuya-zigbee-switch](https://github.com/romasku/tuya-zigbee-switch).

## License

MIT. The original blueprint templates come from the Home Assistant community thread linked above.
