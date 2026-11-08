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

### Mixed installation: alternative switches and stock relays

The button endpoint of a switch running the alternative firmware can also be bound to a stock relay (any device that accepts On/Off commands): in the Bind tab, source endpoint 1 → the stock device, its On/Off endpoint, cluster `genOnOff`. A press then switches the stock relay directly over Zigbee, without Home Assistant.

The reverse path (stock relay → alternative switches) cannot be done by a binding, because a stock relay sends no command when it is switched. It is done by `multicontrol_slave.yaml`: stock relays in `relays`, relays of the alternative switches in `slave_relays`.

Use `toggle_smart_sync` on the alternative switches: the stock relay then receives an explicit ON or OFF, never a TOGGLE, so every command is harmless when the target is already in that state (see [Echoes, case by case](#echoes-case-by-case)).

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

With a single entity in `relays`, there is no masking delay: every change of that relay is sent immediately to the slave relays. The relay never receives a command from the automation and the slave relays are never read, so the loop is open and cannot oscillate.

### `multicontrol_hybrid.yaml`

Inputs: `relays`, `switch_press`, `slave_relays`, `state_bit`.

- `switch_press`: the "Switch press action" entities of the switches running the alternative firmware.
- `state_bit`: an `input_boolean` helper, one per group, used as internal state bit.

Behaviour:

- A press (`press`) toggles the internal bit. The resulting on or off is sent to the `relays` only; the slave relays follow by themselves through the Zigbee bindings. A press is never masked, but it starts the masking delay.
- The first unmasked change of a relay sets the bit and is sent immediately to the other relays and to the slave relays.
- 1 s after the last state change (press or relay), the slave relays are forced to the state of the bit (anti-desynchronisation).
- The state of the slave relays is never used.

## Echoes, case by case

### What an echo is

An **echo** is a state report that comes back to Home Assistant because of a command, not because of a person. Home Assistant (or a Zigbee binding) sends ON to a relay, the relay switches, and τ later (the pure delay, typically a few hundred ms) its new state arrives in Home Assistant as a state change, exactly like a manual change would.

An echo is harmless in itself. It becomes a problem only when it triggers a new command back towards the device it came from: then each echo produces a command, which produces an echo, and so on. That is a **closed loop**. A loop needs an echo at both ends: each device in it must be both a source (its changes trigger commands) and a consequence (it receives commands).

How each kind of device behaves:

| Device | Receives commands | Its state changes trigger commands | Role in a loop |
|---|---|---|---|
| Stock relay in `relays` / `linked_entities` | yes | yes | echoes at both ends: can be part of a loop |
| Relay of an alternative switch in `slave_relays` | yes (Home Assistant or binding) | no: the blueprints never read it, and the firmware sends no bound command when the relay is switched remotely | absorbs the echo: the chain stops there |
| Button of an alternative switch | no | yes, but only on a physical press | starts a chain, never continues one |

### How a loop oscillates

Two stock relays S1 and S2, both OFF, synchronised by Home Assistant with no masking, τ = 300 ms. Someone switches S1 ON then OFF quickly (a double press):

1. t = 0: S1 → ON. Home Assistant sends ON to S2.
2. t = 50 ms: S1 → OFF. Home Assistant sends OFF to S2.
3. t = 300 ms: echo of S2 → ON. Home Assistant sees "S2 changed to ON" and sends ON to S1.
4. t = 350 ms: echo of S2 → OFF. Home Assistant sends OFF to S1.
5. t = 600 ms: echo of S1 → ON, Home Assistant sends ON to S2. t = 650 ms: echo of S1 → OFF, Home Assistant sends OFF to S2.
6. t = 900 ms: echoes of S2 again, and so on.

The ON/OFF pulse bounces between the two relays every τ forever: the lights blink on their own until someone stops the automation. One quick double action is enough; with more relays and crossing actions, the pattern gets more chaotic but never dies out.

The masking delay breaks this: after the first change acts, every change within the next second is ignored. The echoes of steps 3 and 4 arrive within that second and send nothing. (The second change of S1, at step 2, is masked too: this is the limitation described below.)

### `multicontrol.yaml`: only stock relays

Every entity is both source and consequence, so the loop above is always possible. The 1 s masking is required.

### `multicontrol_slave.yaml` with one stock relay

Typical mixed installation: one stock relay S, alternative switches X, Y bound to each other and to S.

- **Press on X**: X switches its relay and sends ON over Zigbee to Y and S. S reports ON → Home Assistant sends ON to the slave relays X and Y. This is the echo on the alternative side: X and Y receive the state they already have. They switch nothing and send nothing to their bindings. The chain stops.
- **S switched from Home Assistant or by hand**: S reports → Home Assistant sends the state to X and Y. Nothing comes back: X and Y are never read, and S never receives a command from the automation.

S is a source but never a consequence: the loop is open at the alternative end, and there is no masking delay. Every change of S is sent immediately.

**Late echo.** Two quick presses on X:

1. Press 1: X, Y → ON through the bindings; S → ON.
2. S reports ON → Home Assistant sends ON to X and Y.
3. Press 2: X, Y → OFF through the bindings; S → OFF.
4. The ON command of step 2 arrives at X and Y after press 2: X and Y go back ON while S is OFF.
5. S reports OFF → Home Assistant sends OFF to X and Y: the group is back in sync.

The late echo creates a short desync, never an oscillation, because the echo goes towards devices that do not answer. Each report of S triggers a command with the state of S at that moment, and with `mode: restart` the last report of S always wins. The group ends in the state of S, provided the last report of S reaches Home Assistant.

**Why `toggle_smart_sync`.** The echo is harmless only because it is idempotent: Home Assistant sends `turn_on` / `turn_off`, an explicit state. The binding from X to S must be explicit too. With `toggle_simple`, X sends TOGGLE: if S is not in the state X expects (late echo, lost command, change made from Home Assistant), the toggle moves S away from X, and nothing brings it back.

### `multicontrol_slave.yaml` with several stock relays

Stock relays S1, S2 and alternative switches X, Y.

- The alternative side still absorbs its echoes, as above.
- The stock side is a closed loop again: S1 changes → Home Assistant commands S2 → S2 echoes → Home Assistant would command S1 → … This is the loop of `multicontrol.yaml`, between S1 and S2 only.

The 1 s masking is required, and only because of the stock relays: this is why it is applied when `relays` holds more than one entity.

### `multicontrol_hybrid.yaml`

A press goes through Home Assistant: the internal bit is toggled and sent to the stock relays, which echo after τ.

- With several stock relays, their echoes form the loop of `multicontrol.yaml`: the masking is required.
- With a single stock relay R, the echo cannot loop: it sets the bit to the value it already has, is sent to the slave relays (which absorb it), and nothing is sent back to R. The masking is still required, for another reason: a late echo would overwrite the bit with an old state. Two quick presses send ON then OFF to R; without masking, the ON echo arriving after the second press sets the bit back to ON and sends ON to the slave relays. If the OFF echo is then lost, the bit stays wrong: the final resynchronisation forces the slave relays to the wrong state, and the next press starts from the wrong bit and seems to do nothing. With the masking, the bit changes only on presses and on genuine relay changes.
- The final command to the slave relays (1 s after the last event) is absorbed like any command to slave relays.

### Summary

| Setup | Closed loop | Masking |
|---|---|---|
| `multicontrol`, any number of stock relays | yes | required |
| `slave`, one stock relay | no (open at the alternative end) | none |
| `slave`, several stock relays | yes, between the stock relays | required |
| `hybrid`, several stock relays | yes, between the stock relays | required |
| `hybrid`, one stock relay | no, but late echoes would corrupt the bit | required |
| Alternative switches bound to each other only | no | none (no automation) |

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
- In `multicontrol_slave.yaml` with a single relay, that relay never receives a command from the automation: there is no echo to mask, and no delay.

No command can cause another command: for any pure delay below 1 s, the loop is open. The details, case by case, are in [Echoes, case by case](#echoes-case-by-case).

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
