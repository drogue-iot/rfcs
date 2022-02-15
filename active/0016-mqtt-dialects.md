# [Cloud] Allow the MQTT endpoint to serve different dialects

* Issue: https://github.com/drogue-iot/drogue-cloud/issues/231
* PR: https://github.com/drogue-iot/rfcs/pull/16

## Motivation

Currently, the MQTT endpoint serves a Drogue Cloud specific MQTT dialect.

However, we also agree that we want to serve other dialects too:

* https://github.com/drogue-iot/drogue-cloud/issues/90
* https://github.com/drogue-iot/drogue-cloud/issues/98
* https://github.com/drogue-iot/drogue-cloud/issues/222

As with MQTT 3.1 there is no indication a client could send to the server which dialect it want to "speak", we need another way to distinguish what the client wants. A simple way would be to simply create additional endpoints which serve one dialect each.

The downside is, that we already have 3 variants (MQTT with client certs, MQTT over WS with client certs, MQTT over WS without client certs). Each of them has a dedicated deployment. So having 4 different dialects, we would end up with 12 different services/deployments.

So, we do need a different way to solve this.

## Requirements and non-goals

There must be a way that we can serve multiple MQTT dialects and the tenant/user has the ability to select this the
way it is needed by connecting devices. It should be simple and understandable and must work with standard MQTT
functionality.

While it might be similar, this RFC does not discuss the MQTT integration (consuming events from Drouge Cloud via MQTT).

## Definitions

<dl>
    <dt>Client</dt><dd>An MQTT device, connecting to the MQTT endpoint</dd>
</dl>

## Detailed design

In the spec section (of application and device) we will make the dialect configurable. When the device connects, it
will be authenticated, and with that we do have access to the application and device configuration. This allows to
create per-application/per-device settings to tweak the dialect.

### Configuration

We will add a new section in the application spec section, which described the expected/desired MQTT dialect:

#### Variant 1

```yaml
specWithOptions:
  mqtt:
    dialect:
      type: drogue/v1
      complexTopics: true

specWithDefaults:
  mqtt:
    dialect:
      type: drogue/v1
```

#### Variant 2

```yaml
specWithOptions:
  mqtt:
    dialect:
      drogue:
        v1:
          complexTopics: true

specWithDefaults:
  mqtt:
    dialect:
      drogue:
        v1: {}
```

#### Variant 3

```yaml
specWithOptions:
  mqtt:
    dialect:
      drogue/v1:
        complexTopics: true

specWithDefaults:
  mqtt:
    dialect:
      drogue/v1: {}
```

### Defaults and overrides

If omitted, this will be the current (drogue v1) dialect. Providing an explicit setting on the application level
would overwrite this. It is possible to provide the same configuration element on the device level (for the connecting
device only), which would override the application specific settings. It is however intended to stick to a single
dialect for one application.

In case the device overrides the configuration, the full `.spec.mqtt.dialect` object is being overridden. It is not
merged.

### Dialects

The first two dialects to specify would be:

#### Drogue V1

This is the current default. Still, you can explicitly configure it.

It does not have any further options.

Example configuration:

```yaml
spec:
  mqtt:
    dialect:
      type: drogue/v1
```

#### Plain topic

This dialect just takes the full topic used for publishing and uses it as the subject. It is not possible to
publish on behalf of another device using this dialect.

It does not have any further options.

Example configuration:

```yaml
spec:
  mqtt:
    dialect:
      type: plainTopic
```

## Breaking changes

As the default is still the current MQTT dialect, there is no breaking change.

## Alternatives

### Multiple services

As mentioned in the introduction, an alternative could be to create multiple endpoints, each serving a single
dialect only. The client would make it's choice by connecting to the service/endpoint it desires.

This would create less code, but a much more complex deployment. Also, would this remove the ability to control this
behavior from the cloud side.

### MQTT 5 only

We could also use MQTT 5 properties when connecting and make this an MQTT 5 only feature. However, as MQTT 3.1 is still
pretty dominant, that seems like a bad choice.

### Encode information in the credentials

It would also be possible to encode some additional information in the credentials. Like adding a suffix to the
username, which indicates which dialect the client requests.

That would work with all current MQTT versions, but would also make the client configuration more complex. And it would
again remove the ability to control this from the cloud side.