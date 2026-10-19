---
date: 2026-10-19
lang: en_US
status: proposed
author: Steffen Uhlig
---

# MQTT as a new device category for Stagehand

## Context and Problem Statement

Stagehand should integrate external devices into automation without their manufacturers' software. Many of these devices speak MQTT: you publish a text message to a topic, e.g. `workshop/ceiling-lamp` with the text `on`, or you subscribe to a topic and receive a message as soon as something happens, e.g. `home/garage/gate` with the text `opening` when the gate opens. This requires a broker to which all devices connect; the broker delivers each message to exactly the subscribers of the matching topics and, with wildcards (`+`, `#`) in subscriptions, to several matching topics at once. Stagehand does not have a device category for this yet. The question is: How does Stagehand integrate devices that are reachable via MQTT – in both directions?

## Decision Drivers

* External MQTT devices (lamps, gates, LED strips) should be controllable and usable as triggers without their manufacturers' software.
* The solution should work in both directions: Stagehand reacts to messages (receiving) and sends messages (sending).
* It should fit the existing model: an action connects triggers with steps, and devices live in lists as with MIDI.
* The effort should stay low; for now the message is modeled as text (UTF-8), even though MQTT payloads are actually bytes.

## Considered Options

* MQTT as a new device category in both directions
* MQTT in one direction only (only triggers or only steps)
* No MQTT

## Decision Outcome

Chosen option: "MQTT as a new device category in both directions", because only it covers receiving and sending and thus matches the role model of MIDI ("MIDI in both directions"); the alternatives solve only half the task or none at all.

Concretely this means:

1. A place that holds the connection to the broker: host and port, username, password, TLS and a client ID.
2. A new device of type MQTT that
	a) can serve as a trigger: Stagehand subscribes to a topic and reacts to incoming messages, optionally with a condition on the message text (receiving);
	b) can serve as a step: Stagehand publishes a message to a topic (sending).

If a step fails because the connection is missing, only that step does nothing and the action goes on – as with a missing device. For triggers the opposite applies: if Stagehand is not connected, messages are missed, because MQTT does not buffer messages (except as retained messages).

### Consequences

* Good, because MQTT is established, open and simple, and devices are integrated without their manufacturers' software.
* Good, because an MQTT device can have several topics, e.g. the "Feinstaub-Ampel" publishes its measured values on `state/feinstaub/sensor` and its status transitions on `state/feinstaub/monitor`.
* Bad, because a persistent network connection to the broker is required, which has to be maintained.

### Confirmation

With the example project [mqtt-led-tower](https://github.com/suhlig/mqtt-led-tower): a trigger subscribes to the tower's status topic `state/feinstaub/monitor` and reacts when a message announces the alarm status, e.g. `{"previous":"ok","new":"alert"}` after fine-dust levels rose; a step publishes the text `on` to the command topic `cmnd/feinstaub/led/red`, e.g. to switch on the red LED. This confirms both directions, the trigger with a condition on the message text.

## Pros and Cons of the Options

### MQTT as a new device category in both directions

* Good, because sending and receiving are combined in one device, as with MIDI (there, every device has a switch for sending and one for receiving).
* Good, because actions follow the familiar pattern: trigger "message on `state/feinstaub/monitor`" and step "device 'Feinstaub-Ampel', topic `cmnd/feinstaub/led/red`, text `on`".
* Neutral, because the message is modeled as text for now, although MQTT payloads are actually bytes (like MIDI's "named values instead of bytes").
* Bad, because a broker connection must be managed and monitored.

### MQTT in one direction only

* Good, because it is less effort if one direction suffices.
* Bad, because the second direction is needed anyway for the example (commanding the tower's LEDs and reacting to its fine-dust status).

### No MQTT

* Good, because no new building block is needed.
* Bad, because MQTT devices would remain unreachable and you would depend on manufacturers' software or intermediate solutions.

## More Information

* The status is "proposed"; the decision is still open and should be revisited once Stagehand ships (the app is "in the works" according to the website).
* [mqtt-led-tower](https://github.com/suhlig/mqtt-led-tower): example of an MQTT device, the "Feinstaub-Ampel" (fine-dust traffic light) driven by an SDS011 sensor.
* [Stagehand – MIDI](https://apps.metaebene.me/en/stagehand/features/midi): the role model for "sending" and "receiving" per device.
