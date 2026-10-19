---
date: 2026-10-19
lang: en_US
status: proposed
author: Steffen Uhlig
---

# Webhooks as a new device category for Stagehand

## Context and Problem Statement

Stagehand should integrate external devices into automation without their manufacturers' software. Many of these devices and services speak webhooks: you register a URL that the service calls as soon as something happens, e.g. `https://stagehand.example/webhook/garage-gate` with the body `{"state":"opening"}` when the gate opens, or you call a URL yourself to trigger something, e.g. you POST `on` to `https://bridge.example/cmnd/ceiling-lamp`. Webhooks are plain HTTP: no broker, no persistent connection, each call is an independent request/response exchange, and the caller learns from the status code whether the call succeeded. Stagehand does not have a device category for this yet. The question is: How does Stagehand integrate devices that are reachable via webhooks – in both directions?

## Decision Drivers

* External webhook-capable devices and services (smart-home bridges, GitHub, monitoring tools) should be controllable and usable as triggers without their manufacturers' software.
* The solution should work in both directions: Stagehand reacts to incoming calls (receiving) and sends calls itself (sending).
* It should fit the existing model: an action connects triggers with steps, and devices live in lists as with MIDI.
* The effort should stay low; for now the payload is modeled as text (UTF-8), even though webhook payloads are usually structured JSON.
* The solution should stay simple enough to test and invoke with plain `curl`, without client libraries or protocol tooling.

## Considered Options

* Webhooks as a new device category in both directions
* Webhooks in one direction only (only triggers or only steps)
* WebSocket as a new device category
* gRPC as a new device category
* No webhooks

## Decision Outcome

Chosen option: "Webhooks as a new device category in both directions", because only it covers receiving and sending and thus matches the role model of MIDI ("MIDI in both directions"); the alternatives solve only half the task or none at all, and the more powerful alternatives (WebSocket, gRPC) buy their extra possibilities with complexity that the current scope does not need.

Concretely this means:

1. A place that holds the endpoint configuration: for incoming webhooks the path Stagehand listens on, the HTTP method it accepts (POST) and a secret to verify callers; for outgoing webhooks the target URL, HTTP method (POST by default), headers and authentication.
2. A new device of type Webhook that
	a) can serve as a trigger: Stagehand exposes an HTTP endpoint, external systems call it, and Stagehand reacts to the request, optionally with a condition on the payload (receiving);
	b) can serve as a step: Stagehand sends an HTTP request to a URL (sending).

If a step fails because the call fails (unreachable host or non-2xx status), only that step does nothing and the action goes on – as with a missing device. For triggers the opposite applies: if Stagehand is not reachable, calls are missed, because webhooks do not buffer messages (some services retry on their own).

A device that is reachable both via MQTT and via webhooks appears as two separate devices in Stagehand, one per category, each with its own connection and endpoint configuration. The categories are independent building blocks that follow the same MIDI-inspired model; the user picks the transport that fits the device, and nothing forces the two to be unified.

### Consequences

* Good, because webhooks are established, open and simple, and devices are integrated without their manufacturers' software.
* Good, because a webhook device can have several endpoints, e.g. the "Feinstaub-Ampel" exposes one endpoint for its measured values and one for its status transitions.
* Good, because HTTP gives feedback: a step sees the status code of its call and can retry, and a trigger can answer the caller.
* Bad, because for incoming webhooks Stagehand must be reachable from the outside (public URL, port forwarding or tunnel), and callers must be verified so that strangers cannot trigger actions.

### Confirmation

With the example project [mqtt-led-tower](https://github.com/suhlig/mqtt-led-tower): a trigger exposes an endpoint that the tower's status monitor calls and reacts when a message announces the alarm status, e.g. `{"previous":"ok","new":"alert"}` after fine-dust levels rose; a step POSTs the text `on` to the command URL of the red LED, e.g. to switch it on. This confirms both directions, the trigger with a condition on the payload.

## Security

The HTTP server that receives incoming webhooks must not become an attack surface.

* **Listening address**: bind to `127.0.0.1` (localhost) by default, so only local processes can reach the server. If external services must call Stagehand directly, bind to `0.0.0.0` – but then TLS and authentication must be handled at the edge. The recommended setup is a reverse proxy (nginx, Caddy, Traefik) or a tunnel (ngrok, Tailscale Funnel, Cloudflare Tunnel) in front of Stagehand: it terminates TLS, forwards to the localhost port, and keeps Stagehand itself unexposed.
* **Port**: configurable, with a default above 1024 so Stagehand can run unprivileged (ports below 1024 need root); a default like `8765` is a reasonable starting point, and the reverse proxy forwards to it.
* **Authentication**: a shared secret that the caller passes as a custom HTTP header, e.g. `X-Stagehand-Token: <secret>`. Stagehand compares it in constant time to avoid timing attacks. For stronger integrity, an HMAC signature over the body (like GitHub's `X-Hub-Signature-256`) additionally proves the payload was not tampered with. The secret is long, random, and configured per endpoint.
* **Transport**: the header is only as safe as the transport. On localhost, plain HTTP is acceptable because traffic never leaves the machine; anything exposed beyond localhost must use HTTPS so the secret is not sent in cleartext.
* **Outgoing webhooks** carry the same idea in reverse: Stagehand sends a bearer token or shared secret header to the target service, and uses HTTPS where possible.

## Message Authoring

The body of an outgoing webhook (and the expected payload of a trigger) is modeled as text for now. How is it authored?

* **Plain text editor with placeholders**: the user writes the raw body in a text field, with placeholders such as `{{trigger.payload.state}}` or `{{device.name}}` that Stagehand fills at runtime. Simple, flexible, matches the "text message" model, and testable with `curl`.
* **Structured builder in the UI**: form fields (key/value pairs, dropdowns for known fields) that generate the JSON. Easier for non-technical users and less error-prone, but more UI work and less flexible.

Both are planned, starting with the plain text editor with placeholders – it fits the "effort stays low" driver and the `curl`-testability pro. The structured builder is a later enhancement. Placeholders can appear in the URL and headers too, not only in the body.

## HTTP Methods

The step (outgoing) lets the user choose the HTTP method, with POST as the default.

The trigger (incoming) accepts POST, the standard for webhook delivery. The trigger answers with a meaningful status code: `200`/`202` on success, `401`/`403` when the caller's secret is missing or wrong, `404` for an unknown path, and `429` when rate-limited.

## Pros and Cons of the Options

### Webhooks as a new device category in both directions

* Good, because sending and receiving are combined in one device, as with MIDI (there, every device has a switch for sending and one for receiving).
* Good, because actions follow the familiar pattern: trigger "call on `/webhook/feinstaub/monitor`" and step "device 'Feinstaub-Ampel', URL `https://bridge.example/cmnd/feinstaub/led/red`, text `on`".
* Good, because HTTP is request/response: a step knows whether its call succeeded and can retry, and a trigger can answer the caller.
* Good, because a webhook is just an HTTP call: you can test and invoke it with `curl` from the command line, no client library, no protocol tooling.
* Neutral, because the payload is modeled as text for now, although webhook payloads are usually structured JSON (like MIDI's "named values instead of bytes").
* Bad, because incoming webhooks require Stagehand to be reachable from the outside and callers must be verified.

### Webhooks in one direction only

* Good, because it is less effort if one direction suffices.
* Bad, because the second direction is needed anyway for the example (commanding the tower's LEDs and reacting to its fine-dust status).

### WebSocket as a new device category

WebSocket is a persistent, full-duplex connection over a single TCP connection (upgraded from HTTP), on which both sides can push messages at any time.

* Good, because a single connection carries traffic in both directions, so a device and Stagehand can exchange messages continuously without re-establishing connections.
* Good, because it enables low-latency, high-frequency updates (e.g. streaming sensor telemetry) that repeated HTTP requests would not handle as well.
* Bad, because it is more complex than webhooks: the connection must be kept alive (ping/pong), reconnected when it drops, and tracked per client, which makes the server stateful and harder to scale.
* Bad, because it is not testable with plain `curl` and needs a client library or tooling on both sides.
* Bad, because the extra possibilities (streaming, push in both directions on one connection) are not needed for the current scope, which is discrete messages as triggers and steps.

### gRPC as a new device category

gRPC is an RPC framework over HTTP/2 that uses Protocol Buffers and a shared, typed contract (`.proto` file).

* Good, because the typed contract gives schema validation and versioning: both sides agree on the messages at compile time.
* Good, because it offers bidirectional streaming and multiplexing over one connection, with efficient binary encoding.
* Bad, because it is the most complex option: every device would have to implement the shared `.proto` contract, which contradicts integrating heterogeneous external devices without their manufacturers' software.
* Bad, because it needs code generation and client libraries, and is not usable with plain `curl` (only with `grpcurl` and similar tooling).
* Bad, because binary protobuf payloads are harder to inspect and debug than JSON over HTTP.

### No webhooks

* Good, because no new building block is needed.
* Bad, because webhook-capable devices would remain unreachable and you would depend on manufacturers' software or intermediate solutions.

## Open Questions

* **TLS**: for incoming webhooks, is TLS terminated by a reverse proxy in front of Stagehand, or should Stagehand serve HTTPS itself? For outgoing webhooks, HTTPS is the default where the target supports it.
* **Retries and idempotency**: should a failed outgoing step be retried, and how often? An `Idempotency-Key` header (where the target supports it) makes retries safe. Incoming webhooks are often delivered more than once by the caller – triggers must handle duplicate deliveries idempotently.
* **Timeouts and size limits**: outgoing calls need a timeout; incoming bodies need a maximum size.
* **Response to the caller**: what does a trigger return – `200 OK` immediately, or a body that the caller can use?
* **Rate limiting and abuse protection**: an exposed endpoint can be hit by anyone on the network; should requests be rate-limited?
* **Logging**: incoming and outgoing calls should be logged for debugging.

## More Information

* The status is "proposed"; the decision is still open and should be revisited once Stagehand ships (the app is "in the works" according to the website).
* [mqtt-led-tower](https://github.com/suhlig/mqtt-led-tower): example of a device, the "Feinstaub-Ampel" (fine-dust traffic light) driven by an SDS011 sensor.
* [Stagehand – MIDI](https://apps.metaebene.me/en/stagehand/features/midi): the role model for "sending" and "receiving" per device.
