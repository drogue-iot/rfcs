# [Cloud] Allow MQTT based gateways to deliver on behalf of another device

A device, connected via MQTT, should be able to send messages on behalf of another device. It should also be
able to receive commands on behalf of another device.

* Issue: https://github.com/drogue-iot/drogue-cloud/issues/135
* PR: https://github.com/drogue-iot/rfcs/pull/13

## Motivation

For one, we already have the functionality using HTTP. And we also use it in e.g. The Things Network integration.

Second, for MQTT it makes even more sense, as MQTT is a long-lived connection model, used by IoT gateways. And so,
keeping a single connection open, allowing the gateway to deliver various messages for its connected devices, saves
a lot of resources which would otherwise be required for handling connections.

## Requirements and non-goals

In a nutshell, the MQTT endpoint should allow the device to indicate "as which" device the message should be delivered
on the cloud side. This should follow the same behaviors as the "as" flag on the HTTP endpoint, just not only for
establishing the connection, but for each published message.

When the connection is established, the (gateway) device still gets authenticated. When publishing a message, the device
can indicate "as which" device the message should be delivered. If the device identity deviates from the authenticated
device, a check must be made, which ensures that the device is allowed to act on behalf of the other device.

If that is the case, the message will be processed. Otherwise, the message will be rejected

No new mechanism should be created (unless absolutely necessary) to evaluate if the gateway is authorized for the device
it wants to act on.

## Detailed design

### Indication using the topic name

**NOTE:** This would work with MQTT v3.1.1 and MQTTv5.

In RFC #0001, we already defined to use a topic name of `<channel>/<device>` for this: https://github.com/drogue-iot/rfcs/blob/main/active/0001-cloud-multi-tenancy.md#gateway-device.
However, it currently is not implemented that way.

My proposal is to actually follow that implementation, and only use the first segment of the topic as "channel" as
defined. And use the second path segment as device alias. Ignoring all further segments.

### Indication using an MQTTv5 user property

**NOTE:** This would only work with MQTT v5.

Similar to the HTTP endpoint, the MQTT client could add an `as` user property to indicate as which device it wants
to publish.

Pros:

* Would not affect the topic name
* Could be used in addition to a device specific topic name
* Could be used in combination with topic aliases to reduce the overall payload

Cons:

* Would only be available using MQTT v5, so it could only be an addition
* Is probably a bit more verbose than using a device specific topic name when not using topic aliases

### Authorization

For the authorization, we need to create a new API in the authentication service, as current calls are all
authenticating and authorizing at the same time, when connecting.

The call receives the device name of the gateway device and the device identifier (which could also be an alias) from
the indication step before.

The authentication service will look up the device using the alias, and then check if the device lists the provided
gateway as an allowed gateway.

The service returns "ok" or "not ok" back to the caller (endpoint), which can then act on the outcome.

### Acceptance

If the request is accepted, the endpoint passes the message on the downstream sink, using the device ID instead of the
gateway ID in the process.

### Rejection

If the request is rejected, the behavior differs based on the MQTT version and the QoS.

| MQTT version | QoS | Outcome |
| ------------ | --- | ------- |
| 3.1.1        | 0   | Message is dropped |
| 3.1.1        | 1   | Connection is closed |
| 3.1.1        | 2   | Connection is closed (as QoS is not supported) |
| 5            | 0   | Message is dropped |
| 5            | 1   | `PUBACK` sent with 135 "Not authorized" |
| 5            | 2   | Connection is closed with 155 "QoS not supported" |

## Breaking changes

### Limiting the topic name/channel

Currently, the full topic is used as channel when the device publishes. This will be changed so that only the
first path of the topic will be used as channel. The second will be an optional device identifiers. All other
segments will be ignored.

## Alternatives

### Topic naming

Instead of using a topic name of `<channel>/<device>`, we could also swap this around for `<device>/<channel>`. Both
approaches feel ok, and as we are not using topic to create a hierarchy at this location, it doesn't seem to make
a big difference.

On the other hand, we already devices to use `<channel>/<device>` for this, so we should stick to the plan, unless
there are good reasons not to.
